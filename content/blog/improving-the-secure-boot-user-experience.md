---
title: "Improving the Secure Boot user experience"
date: 2020-05-18T00:00:00+02:00
tags:
  - secure boot
  - english
---

Secure boot tooling is terrible, can we do better?

Currently the most widely used tooling for secure boot is the Ubuntu
[sbsigntools](https://git.kernel.org/pub/scm/linux/kernel/git/jejb/sbsigntools.git/)
and
[efitools](https://git.kernel.org/pub/scm/linux/kernel/git/jejb/efitools.git).
If you are currently using secure boot both of these packages are probably
installed on your system. Both of them support the basics of generating
signature lists and signing the EFI variables with certificates, but they still
have differences which is a source of confusion.

`efitools` has 3 different ways of generating signature lists,
`cert-to-efi-hash-list`, `cert-to-sig-list` and `hash-to-efi-sig-list`.
"Luckily" there are man pages you can read which assumes you have some
familiarity with UEFI itself.

`sbsigntools` has only `sbsiglist` which is mostly fine, but has non-obvious
functionality. How do you sign the checksum of an EFI executable as an example?
I figure that out after some searching on the github code search:

    sha256sum file.efi | awk '{printf "0x"$1}' | xxd -r -g 1 -c 64  > sha256.bin
    sbsiglist --type sha256 --output sha256.bin.siglist sha256.bin

I think it works, I haven't tried yet. However it is not really a nice
experience to use, and the documentation of the tool is severely lacking. The
functionality should be obvious from the get go. Why do you want this? A nice
example is to keep a list of known bad checksums of things you don't want booted
on your system, without having to sign all of it.

Secure boot needs key enrollment if you want to use your own keys. `sbsigntool`
got their undocumented `sbkeysync` utility which can live enroll secure boot
keys it reads from a database in either `/etc/secureboot/keys` or
`/usr/share/secureboot/keys`. Meanwhile `efitools` supports live enrollment
through `efi-updatevar`, or if you can use the bundled `KeyTool.efi` which you
can boot and load your keys from.

But which do you use for signing EFI executables? Only `sbsigntool` implements
this! What you end up with is 3 different ways of enrolling keys, between 2-4
different ways of creating signature lists depending on your usage, and only one
way to sign EFI executables.

This is probably fine and dandy for technical people that can be bothered to
invest some time into their tools. But this isn't a great user experience for
most people, one which I'll wager is a roadblock for people to adopt secure
boot. It is a shame, people should be capable of rolling their own secure boot
keys without having to understand UEFI in detail to do so.

There are multiple guides and instruction on the [Gentoo
wiki](https://wiki.gentoo.org/wiki/Sakaki%27s_EFI_Install_Guide/Configuring_Secure_Boot),
[Arch Linux
Wiki](https://wiki.archlinux.org/index.php/Unified_Extensible_Firmware_Interface/Secure_Boot#Using_your_own_keys),
by [Rod Smith](http://www.rodsbooks.com/efi-bootloaders/controlling-sb.html) and
[Greg
Kroah-Hartman](http://www.kroah.com/log/blog/2013/09/02/booting-a-self-signed-linux-kernel/).
These are great sources of information for how things work. But the inherent
complexity of these articles underlines the problem confusing tooling.

We should do better.


## sbctl

Around two years ago I tried to roll my own secure boot keys and found the
entire experience maddening. The madness promptly made me write 400 lines of bash
for [efi-roller](https://github.com/Foxboron/efi-roller). The project enabled me
to keep track of files I needed to sign between kernel upgrades, systemd
upgrades and fwupdmgr upgrades. I didn't need to conjure up some complicated
hooks for my package manager to ensure I had all the files signed.

It has served me well quite well. However bash gets fairly limiting when you
want some trivial option parsing in-between subcommand parsing, which I needed
for my EFISTUB generation. In the end I decided it was better to start from
scratch with Go to get some sanity back and wound up with `sbctl`.

`sbctl` aims to be a easy interface for managing your own secure boot keys and
follows in the footsteps of `efi-rooler`.

{{< highlight bash >}}
$ sbctl create-keys
==> Creating secure boot keys...
  -> Using UUID d6e9af79-c6b5-4b43-b893-dbb7e6570142...
==> Signing [...]/keys/PK/PK.der.esl with [...]/keys/PK/PK.key...
==> Signing [...]/keys/KEK/KEK.der.esl with [...]/keys/PK/PK.key...
==> Signing [...]/keys/db/db.der.esl with [...]/keys/KEK/KEK.key...

$ sbctl enroll-keys
==> Syncing /usr/share/secureboot/keys to EFI variables...
==> Synced keys!

$ sbctl sign /boot/EFI/BOOT/BOOTX64.EFI
  -> Signing /boot/EFI/BOOT/BOOTX64.EFI...
{{< /highlight >}}

No `openssl` commands. No `cert-to-efi-sig-list` or `sbsiglist` commands. And no
need to figure out how to enroll keys. It just works. And this is how simple it
ought to be.

`sbctl` can also track which files you would like to sign. This can help you
make hooks towards your package manager which signs them between system
upgrades.

{{< highlight sh >}}
λ ~ » sudo sbctl list-files 
==> File: /efi/EFI/Linux/linux-linux.efi
==> File: /usr/lib/fwupd/efi/fwupdx64.efi
  -> Output: /usr/lib/fwupd/efi/fwupdx64.efi.signed
==> File: /boot/EFI/BOOT/BOOTX64.EFI
==> File: /boot/EFI/arch/fwupdx64.efi
==> File: /boot/EFI/systemd/systemd-bootx64.efi
==> File: /boot/vmlinuz-linux

λ ~ » sudo sbctl sign-all
==> /boot/vmlinuz-linux has been signed...
==> /efi/EFI/Linux/linux-linux.efi has been signed...
  -> Signing /usr/lib/fwupd/efi/fwupdx64.efi...
==> /boot/EFI/BOOT/BOOTX64.EFI has been signed...
==> /boot/EFI/arch/fwupdx64.efi has been signed...
  -> Signing /boot/EFI/systemd/systemd-bootx64.efi...
{{< /highlight >}}

The intention is to have a nice and practical utility which should be the center
of the secure boot keys. It should enroll them, rotate the keys when you need
to, and ensure they are secured well enough.

The code can be found on github: https://github.com/Foxboron/sbctl 

I haven't cut a first release yet as I would like some feedback on the general
design of the tool. It would also be interesting to lean how well it fits into
other peoples usecases for secure boot and what it covers and don't cover.
Another problem is that `sbctl` inherits the one of the problems discussed
earlier. 

## goefi

`sbctl` still relies on `sbsigntools` for all of it's functionality. Actively
shelling out to tools is easy, but it's not very satisfying when we are writing
in a system language. This lead me to downloading the 2558 page long ["Unified
Extensible Firmware Interface Specification Version
2.8"](https://uefi.org/specifications) which I started reading a month ago.

It's frankly interesting when you realize how this work on Linux. Admittedly, I
haven't done a lot of low-level programming before. I thought you had to do some
weird `ioctl` or syscalls to modify EFI variables, which during a weak moment
made me consider properly learning C. In practise you are just reading and
writing files under `/sys/firmware/efi/efivars`, which any language should
capable of doing.

So far I have managed to read most of my `Boot*` entries, create EFI signature
lists and have also reimplemented the UEFI variable signing, which you can now
use replace the base functionality of `sbsiglist` and `sbvarsig`. Quite a bit of
code left until I can use it for `sbctl`, but it's starting to shape up nicely. 

The library code is currently on github: https://github.com/Foxboron/goefi.

The end goal is to have a Go library capable of parsing and writing most of the
structs from the UEFI specification in an API friendly way. This could hopefully
enable software authors to write more user friendly secure boot tooling.

If reading `asn1parse` output and looking at hexdumps sounds like an evening
well spent, or you enjoy figuring out nice top-level APIs, please do reach out
and take a look at the code!

It would also very much like to hear about more use cases for `sbctl` and what
people currently are doing with existing secure boot tooling.
