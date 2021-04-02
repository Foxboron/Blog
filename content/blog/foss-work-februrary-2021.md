
---
title: "FOSS Activities in February 2021"
date: "2021-03-01T18:00:00+02:00"
tags:
  - foss
  - monthly
  - archlinux
  - english
---

Yo!

New month, new update!

The start of this month was marked with FOSDEM! I held a talk about secure boot
and the tooling stuff I have written, `sbctl`. It's a tool to help you manage
secure boot keys and signing files. With help from `sbsigntools` it also does
live enrollment of keys.

The talk went great (I think) and it was fun to see how FOSDEM pulled off the
conference with matrix and jitsi. I gave me some inspiration for Arch Conf 2021
that I should try kick off some planning on.

The talk can be found here: https://fosdem.org/2021/schedule/event/firmware_itsblsg/

I also released the 0.1 of `sbctl` and packaged it up for Arch Linux. I have
sorta been planning for a 1.0 release, but I work too slowly for it to happen in
a reasonable time frame. Doing a beta release allows people to package it and
better provide some testing releases to people.

On the other side of things there has been a [Go 1.16 release](https://golang.org/doc/go1.16). This means we
we do a [rebuild](https://archlinux.org/todo/go-116-rebuild/) of all the packages to get any recent improvements from the
compiler. The status can be found on our todo list. You might be wondering what
the changes are. And it's a bit complicated.

Arch has been building with the `pie` buildmode for a while. This allowed us to
utilize `CGO_LDFLAGS` and other cgo flags to pass linker commands. This lets us
utilize binary hardening options like stack canaries, full relro and fortify. It
turns out that the PIE build mode for the compiler always included the cgo
runtime when build with the external linker. This was fixed in Go 1.16 which
limits the usability of the `CGO_*` flags for bypassing compiler flags in random
build systems upstreams have. Go doesn't allow you to force cgo either (except
for an documented bug in the compiler).

Issue: https://github.com/golang/go/issues/44480  
`runtime/cgo` fix: https://github.com/golang/go/commit/6c0135d377  
Documented bug: https://github.com/golang/go/issues/31544  

Effectively this means binary hardening is getting more complicated again.
`GOFLAGS` is a Go compiler flag which should let you pass compiler and linker
flags during building. The problem is that `GOFLAGS` has a broken parser which
doesn't parse spaces that well. It also doesn't help that multiple build systems
utilize the variable name for the command line flags itself. This makes it a
50/50 if you can use the env flag without breaking Makefiles upstreams provides.

That's the gist of it but it's annoying to say the least. I have been working on
a small patch to the go compiler that would enable full relro if possible which
would give us back some hardening abilities if we are able to specify
`-buildmode=pie`. But the situation is not really ideal.

Other then that I have packages up a few new packages. The most exciting has
been GNU poke. It's binary data editor and has it's own domain specific language
for working with formats. It allows you to edit and patch binary files with
little fuzz. http://www.jemarch.net/poke

I have needed this since working with UEFI and Secure Boot stuff has been a lot
of diffing hexdumps and looking at asn1parse output from openssl. This allows me
to specify and programmatically check my data structures in an easier language
then the manual code I have to write up for go.

Another interesting project is ImHex which is an complete editor. But the binary
data parser language is more limited currently.

Poke: http://www.jemarch.net/poke  
ImHex: https://github.com/WerWolv/ImHex  

Some contributions for my UEFI and Secure Boot endeavours which I hope to
upstream when more is done and I understand the language better. I'll probably
try write up a blog post on all of this whenever I'm done playing valheim :)

ImHex-Patterns PR: https://github.com/WerWolv/ImHex-Patterns/pull/8  
poke-uefi: https://github.com/Foxboron/poke-uefi  

If you have questions, want to reach out or suggestions for these posts please
poke me on IRC as `Foxboron` or email me on morten@linderud.pw.

Now rest of the post follows as usual :) 

{{< figure src="/img/arch.png" >}}

# Package Updates to [community]
- `i3-gaps` updated to `4.19.1`
- `tailscale` updated to `1.4.2-1`, `1.4.3-1`, `1.4.4-1`, `1.4.4-2`, `1.4.5-1`
- `python-adblock` updated to `0.4.2-1`
- `go` updated to `2:1.15.8-1`, `2:1.16-1`
- `helm` updated to `3.5.2-1`
- `runc` updated to `1.0.0rc93-1`
- `conmon` updated to `1:2.0.26-1`
- `fzf` updated to `0.25.1-1`, `0.25.1-2`
- `lxd` updated to `4.11-1`, `4.11-2`
- `buildah` updated to `1.19.4-1`, `1.19.6-1`, `1.19.6-2`
- `cni-plugins` updated to `0.9.1-1`, `0.9.1-2`
- `gopass` updated to `1.12.0-1`, `1.12.1-1`
- `python-google-api-core` updated to `1.26.0-1`
- `python-autobahn` updated to `21.2.1-1`
- `python-prompt_toolkit` updated to `3.0.16-1`
- `podman` updated to `3.0.0-1`, `3.0.1-1`, `3.0.1-2`
- `plocate` updated to `1.1.4-1`, `1.1.5-1`, `1.1.5-2`
- `python-pandas` updated to `1.2.2-1`
- `buildah` updated to `1.19.6-1`
- `crun` updated to `0.18-1`
- `docker-compose` updated to `1:24.4-1`, `1:20.10.3-3`
- `staticcheck` updated to `2020.2.2-1`, `2020.2.2-2`
- `step-ca` updated to `0.15.8-1`
- `mypy` updated to `0.812-1`
- `skopeo` updated to `1.2.2-1`
- `python-docker` updated to `4.4.3-1`
- `gopls` updated to `0.6.4-2`
- `go-tools` updated to `2:1.16+4895+c1934b75d0-3`
- `micro` updated to `2.0.8-4`
- `toolbox` updated to `0.0.99-2`, `0.0.99.1-1`
- `lostfiles` updated to `4.09-1`
- `archlinux-contrib` updated to `20210221-1`
- `github-cli` updated to `1.6.2-1`
- `slirp4netns` updated to `1.1.9-1`
- `influxdb` updated to `2.0.4-1`
- `salt` updated to `3002.5-3`
- `docker` updated to `1:20.10.4-1`


## Package additions to [community]
- `micro`
- `b4`
- `poke`
- `sbctl`

## Package removals from [community]
- `dep`
- `python2-pyzmq`

### Potential new packages for 
- `oomd`
- `vgrep`
- `git-publish`
- `psi-notify`
- `etcd`
- `gosec`
- `kind`
- `nomad`

# Bugfixes
- `go`: [FS#69408](https://bugs.archlinux.org/task/69408)
- `podman`: [FS#69651](https://bugs.archlinux.org/task/69651)
- `lxd`: [FS#69582](https://bugs.archlinux.org/task/69582)
- `plocate`: [FS#69670](https://bugs.archlinux.org/task/69670) [FS#69640](https://bugs.archlinux.org/task/69640)

## Security Team
The security team has had a steady stream of advisories. This month we have
published 43 of them. There are some mailing list issues for mailman, so if you
want to find published advisories you might not find them through mailman and
I'd much rather advice you to check the main website.

https://security.archlinux.org/


Cheers and see you next month!
