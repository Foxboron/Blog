---
title: "Mailpile, sendmail and procmail"
date: 2014-07-09T00:00:00+01:00
---
<link rel="alternate" type="application/rss+xml" title="{{ site.name }}" href="{{ site.url }}/feed.xml">


What is Mailpile?
=================

[Mailpile] (https://www.mailpile.is) is mail client with a rather unusual goal 
in todays world. It wants to be free, open-source, privacy oriented and easy to
use with encryption. This all comes with the goal of being self-hosted.

This is a contrast to Protonmail who still keeps all your information on their
servers, making people with a slight trust issue look at you in a rather funny
way. However, Protonmail and Mailpile is among several email providers in the
wake of the NSA scandal to try and give you secure options to gmail, outlook and
yahoo. Which is in my opinion, Awesome!

This does not solve the fundamental problem in todays world of emails. People 
can use PGP to communicate securely, but most doesn't even know how it works (its
hard, remember?). Every email you send will get in the hands of gmail, outlook
or yahoo. Which makes NSA happy in the end. But this is all in the right step to
solve this problem.

Now, Mailpile is supposed to be used with a local mail server or an existing
one. You could host it on a local server only you got access to, with full disk
encryption. Or worse, host is on a VPS and let have your private encryption keys.
For the latter there are other possible solutions, which I might take in another
blog post.


Installation
============

You will need 3 things:  
  1. sendmail  
  2. procmail  
  3. mailpile  

This setup assumes you know how to portforward port 25, which needs to be open
to receive mail, along with how to setup a domains DNS records. I will also 
assume basic knowledge on any Linux distro. Even if the configurations are 
slightly different, there should be no major issue.

Sendmail will work as the Message Transfer Agent (MTA), it will control
outgoing mail and incoming mail. Procmail works as a Mail Delivery Agent 
(MDA), whose job is to deliver the mail correctly. Mailpile works on top of this
stack as a Mail User Agent (MUA).

This setup is meant to work for one user. If you need several users, I recommend
looking into postfix as its more flexible and better suited for this task. 
  
  
Sendmail  
========  
  
You will find sendmails configuration  in `/etc/mail`. The first file to look 
at as sendmails main config file, `sendmail.mc`. If it doesn't exist, create a 
new one.

The base configuration should look something like this:

<pre>
include(`/usr/share/sendmail-cf/m4/cf.m4')
define(`confDOMAIN_NAME', `<DOMAIN>')dnl
FEATURE(use_cw_file)

define(`confAUTH_OPTIONS', `A p y')dnl
TRUST_AUTH_MECH(`LOGIN PLAIN')dnl

define(`confAUTH_MECHANISMS', `LOGIN PLAIN')dnl
define(`confCACERT_PATH',`/etc/ssl/certs')
define(`confCACERT',`/etc/ssl/certs/ca.pem')
define(`confSERVER_CERT',`/etc/ssl/certs/server.crt')
define(`confSERVER_KEY',`/etc/ssl/private/server.key')

OSTYPE(linux)dnl
MAILER(local)dnl
MAILER(smtp)dnl
</pre>

This config makes a few assumptions. The `confAUTH_OPTIONS` tells sendmail that
it won't relay mail unless the user authenticates. There won't be any plaintext
authentication on non-TLS connections. A good idea here is to read up on how to
properly generate the correct OpenSSL certs. Remember to adjust the 
`confDOMAIN_NAME` to the correct value, assuming you got a domain. Generating 
the final config should be done with the following command:  
`m4 /etc/mail/sendmail.mc > /etc/mail/sendmail.cf`

The next file is  `/etc/mail/local-host-names`, it should contain all domains to
the server. They should also resolve in your file `/etc/hosts`
  
```  
localhost
<domain>
mail.<domain>
localhost.localdomain
```
  
`/ect/mail/access` contains the list of addresses we want to relay mail from.
Since we are using mailpile, we assume we will only relay mail from our 
local IP.
  
```
127.0.0.1       RELAY
```
  
Generate the config file again as follow:
`makemap hash /etc/mail/access.db < /etc/mail/access`

The last file for sendmail will be `/ect/mail/aliases`.
This file contains the user aliases for our system. So for instance, we only
set the required mail handles to `root`, and set our user to be `root`.
Thus we can receive all the mails to `postmaster` and  `abuse` without having
to care about separate users.
Thus the only line you should change would be:
  
```
...
root:           <your user>
...
```
  
You can at this point add other aliases or other users for better management.
Generate the file for sendmail with the command `newaliases`

Launch the sendmail daemon with your service manager and you should be set!
Systemd user can do `systemctl start sendmail`.


Procmail 
======== 

Procmail will make sure the mails are put in the correct folder. Lets create
`.procmailrc` in the home directory of the user who will receive mail!
  
```
PATH=/bin:/usr/bin
MAILDIR=$HOME/Mail
DEFAULT=$HOME/Mail/inbox
LOGFILE=$HOME/Mail/logfil2
SHELL=/bin/sh

:0:
* ^To:.*domain\.tld
/home/<user>/Mail/
```

This configuration should be simple enough. We define the variables on the top
to tell where the log file should be along with the default mail directories.
`:0:` denotes possible flags, and the last colon tells procmail we will use a
lockfile.
The next is a regex which will be matched with the mail. If it matches we
forward the mail to our users Mail directory. In this case a simple check
for any mail sent to our domain should be enough.

This should seamlessly work with sendmail and put any received mail to your
inbox.


Mailpile
========

NOTE: Mailpile is currently in alpha. Anything after this point is subject to
changes. I will try my best to keep it up dated.

Start the mailpile console with `mp` and type the following lines. Remember to 
change the lines for your needs.

```
# Basic setup
setup
set profiles.0.email = <user>@<domain>
set profiles.0.name = <Your Name>

# Add mailbox and scan for mails
add /home/<user>/Mail
rescan full


# Sendmail route setup
set routes.default = {}
set routes.default.command = /usr/bin/sendmail -i %(rcpt)s

# Add route to profile
set profiles.0.messageroute = default
```

Visit `http://127.0.0.1:33411` and enjoy Mailpile!

Questions can be sent on [Twitter] (https://twitter.com/MortenLinderud)
or mail at [fox@velox.pw](mailto:fox@velox.pw).

## Additional Reading Material
http://weldon.whipple.org/sendmail/starttlstut.html
https://wiki.archlinux.org/index.php/GnuPG
https://wiki.archlinux.org/index.php/OpenSSL
https://wiki.archlinux.org/index.php/OpenDKIM
https://en.wikipedia.org/wiki/Sender_Policy_Framework

## Sources
https://wiki.archlinux.org/index.php/sendmail
https://wiki.archlinux.org/index.php/Procmail

