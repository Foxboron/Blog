---
title: "PAM Bypass: when null(is not)ok"
date: "2020-11-24T20:00:00+02:00"
tags:
  - archlinux
  - security
  - english
---

## The Problem

Someone enters an IRC support channel and proclaims their dovecot server has
been hacked and a non existing user sends spam email from their server. The
initial reaction might be something along the lines of

**Wat ಠ_ಠ**

With the following assumption that the user *clearly* did something wrong.
Hosting email is difficult after all. I don't quite recall how rest of the
support went, but it was solved and the root cause was not found. However, we
keep on rolling! Then someone posts about a similar incident on [r/archlinux](https://www.reddit.com/r/archlinux/comments/jvh38a/postfix_dovecot_got_hacked/).

Now, if this happens twice something is amiss! Arch has had a few issues with
PAM lately, thus it could be that there is a configuration issue.  Johannes and
I try to reproduce, but I don't get far and Johannes keeps on working on the
issue.


## The Setup

The first thing you notice looking into the [`/etc/pamd.d/system-auth`](https://github.com/archlinux/svntogit-packages/blob/packages/pambase/trunk/system-auth)
of Arch Linux is the following lines:

{{< highlight pam >}}
auth  [success=2 default=ignore]  pam_unix.so   try_first_pass nullok
							       ^^^^^^
{{< /highlight >}}

This allows a user with a blank password to go forward with the PAM
authentication. As the [manpage](https://linux.die.net/man/8/pam_unix) explains;

> The default action of this module is to not permit the user access to a service if their official password is blank. The **nullok** argument overrides this default.

The second relevant line is the inclusion of [pam_permit.so](https://linux.die.net/man/8/pam_permit) which indiscriminately
allows anyone reaching this far access to the system. Clearly a must have for
any well functioning system regardless of being "very dangerous" and "used with 
extreme caution" 🙄.

Now, keep all of this in mind as we continue.

The first hint towards the culprit of the issue is when the author of the reddit
posts submits an email to security@archlinux.org:

> Back in May 2020 there was a change to root account in shadow file such that root with no password was no longer supported.
>
> During patching this created a file /etc/shadow.pacnew
>
> If that pacnew was not merged to the shadow file this will result in pam allowing any invalid account to successfully auth with no password.
>
> The problem is that if the * is missing from the root line in the shadow file then the most recent pam system-auth config will allow auth bypass.
>
> This impacted me when my mail server (dovecot/postfix) got breached via a “no password” and sent significant spam.

The change which is mentioned is the following change to the `filesystem` in the
[file `/etc/shadow`](https://github.com/archlinux/svntogit-packages/commit/0320c909f3867d47576083e853543bab1705185b#diff-3e341d2d9c67be01819b25b25d5e53ea3cdf3a38d28846cda85a195eb9b7203a)

{{< highlight diff >}}
-root::14871::::::
+root:*:14871::::::
{{< /highlight >}}

This is something most shadow configurations in Linux distributions carry these
days. Through a bit of oversight the root account of any Arch installation has
no root password set, thus you need to set one yourself or else you can swap tty
and log into the root user. Now this hole was fixed. 

Since this file was changed `pacman` is going to see that the local file has
modification (you probably have more users on your system!) and stuff this
change into `/etc/shadow.pacnew` as noted in the [manpage](https://www.archlinux.org/pacman/pacman.8.html#_handling_config_files_a_id_hcf_a).  This is also part of the `pacman` output, but I guess you can see how it's easy
to miss when you run a server with a few hundred packages to update.

{{< highlight text>}}
[root@archlinux ~]# pacman -S filesystem
resolving dependencies...
looking for conflicting packages...

Packages (1) filesystem-2020.09.03-1

Total Installed Size:   0.03 MiB
Net Upgrade Size:      -0.01 MiB

:: Proceed with installation? [Y/n] 
(1/1) checking keys in keyring                         [###############] 100%
(1/1) checking package integrity                       [###############] 100%
(1/1) loading package files                            [###############] 100%
(1/1) checking for file conflicts                      [###############] 100%
(1/1) checking available disk space                    [###############] 100%
:: Processing package changes.
(1/1) upgrading filesystem                             [###############] 100%
warning: /etc/shadow installed as /etc/shadow.pacnew
:: Running post-transaction hooks...
(1/4) Creating system user accounts...
(2/4) Applying kernel sysctl settings...
(3/4) Creating temporary files...
(4/4) Arming ConditionNeedsUpdate...
{{< /highlight >}}

Usually people install `pacdiff` from `pacman-contrib` to deal with these
issues, as they are made a bit more explicit.

{{< highlight text>}}
[root@archlinux ~]# pacdiff 
==> pacnew file found for /etc/shadow
:: (V)iew, (S)kip, (R)emove pacnew, (O)verwrite with pacnew, (Q)uit:
{{< /highlight >}}

This is the setup of the issue. The shadow file was updated, and the users did
not merge the change. The root account is without any password!

But how does this lead to an authentication bypass in PAM for *invalid* users?
This only applies for root after all.

## The Vulnerability

Levente Polyak theorized that these invalid users clearly was returning
something valid for `pam_unix.so`. How else would they continue to authenticate?
Johannes spelunks through code, looking for the code path that would allow
invalid users to authenticate.

{{< highlight pam >}}
  demize  : I think it might be because of some changes they did to try to 
	    make the password checking for existing and non-existing users 
	    take the same amount of time.
  demize  : On the first iteration it'll try to get the password hash for the 
	    user. It doesn't exist, so it tries against against root, and 
	    since root did have a null password...
anthraxx  : yeah that would explain why it passes with nullok for non existing 
	    users
anthraxx  : that patch itself makes perfect sense to mitigate side channels
{{< /highlight >}}

The patch in question is the commit [`pam_unix: avoid determining if user
exists`](https://github.com/linux-pam/linux-pam/commit/af0faf666c5008e54dfe43684f210e3581ff1bca).

The commit attempts to avoid a timing attack against PAM. Some attacker can know
valid user names by timing how quickly PAM returns an error, so the fix is to
use an existing user in the system we always validate against to ensure a
consistent timing. But which user is always present on a Linux system? root!

The code does *not* check if root has any valid passwords set. An invalid user
would fail, loop over to root and try validate. root has no password. It's
blank. We have `nullok` set. And we have `pam_permit.so`. The invalid user is
authenticated. We have enough information to do a quick POC.

## The POC

{{< highlight text>}}
[root@archlinux ~]# pacman -Q pam dovecot
pam 1.5.0-1
dovecot 2.3.11.3-2

[root@archlinux ~]# cat /etc/shadow
root:*:14871::::::

[root@archlinux ~]# doveadm auth test something
Password: 
passdb: something auth failed
extra fields:
  user=something
  
[root@archlinux ~]# sed -i 's/root:\*/root:/' /etc/shadow

[root@archlinux ~]# cat /etc/shadow
root::14871::::::

[root@archlinux ~]# doveadm auth test something
Password: 
passdb: something auth succeeded
extra fields:
  user=something
  
[root@archlinux ~]# doveadm auth test this-user-is-invalid
Password: 
passdb: this-user-is-invalid auth succeeded
extra fields:
  user=this-user-is-invalid
{{< /highlight >}}

This is clearly unfortunate for people that rely on PAM authentication for
their systems, and a good lecture as to why you probably shouldn't use PAM for
this. Also some material for people that strongly believe Arch is not suitable
for servers. Win-win!

As of taping, the PAM package has been patched in Arch and currently going
through some testing. Luckily it's a compound issue that needs a few things to
go wrong over quite a few months before it amounts to an exploit.

<span title="*hugs from demize*">The vulnerability has been assigned
CVE-2020-27780, and the fixed commit checks if root has a valid password
set.</span>

[`Second blank check with root for non-existent users must never return
1`](https://github.com/linux-pam/linux-pam/commit/30fdfb90d9864bcc254a62760aaa149d373fd4eb)


Thanks to Johannes Löthberg, Santiago Torres and Levente Polyak for reading over
the draft!


