---
title: "FOSS Activities in December 2020"
date: "2021-01-02T11:20:00+02:00"
tags:
  - foss
  - monthly
  - archlinux
  - english
---

End of the year and third blog post! Hope everyone has had a nice new years eve :)

The first news of the month is that Remi Gacogne was [accepted](https://lists.archlinux.org/pipermail/aur-general/2020-December/036034.html) as Trusted User.
Congratulations to him and super exciting.

Other then that I have had a meeting with the devops team discussing how we
should implement the [debuginfod system](https://lists.archlinux.org/pipermail/arch-dev-public/2020-November/030222.html) on our infrastructure. I have written up
the [ansible role](https://gitlab.archlinux.org/archlinux/infrastructure/-/merge_requests/168) for debuginfod and it was more or less decided that we want to
host it on a small VPS for the service itself, and sync debug packages to the
host to serve them. This avoid the problem of hosting more services on our
server which distributes packages with services it does not really need.

My hope is to have debuginfod implemented by February, but there is still quite
a few things to figure out!

## Secureboot.dev 
I have started planning a bit how to organize the notes and stuff i learn while
programming on `sbctl` and `go-uefi`. My current plan is to document it on a
website since I got a pretty nice domain name: [https://secureboot.dev/](https://secureboot.dev/).

The idea is to try document the existing secure boot tools, what they cover and
where they fit. Along with any information that might be useful for users that
intend to use Secure Boot on Linux. I also want to try lay out some pointers of
the UEFI spec. It would mostly deal with how `efivars` works on Linux, and point
out the relevant parts from the specification. The intention is to aid people
that want to provide better userspace tooling for secure boot.

It is currently a work in progress, but there is a [Github repo](https://github.com/Foxboron/secureboot.dev) with the source
if someone is interested contributing to the page.

My talk to the FOSDEM [Open Source Firmware, BMC and Bootloader devroom](https://lists.fosdem.org/pipermail/fosdem/2020q4/003154.html) has also
been accepted. I'll introduce some of the work I have been doing with `sbctl`
and `go-uefi` the past year to make things easier for users, along with my
current gripes with the existing secure boot tooling. Super exciting!

I'm also going to rename the go library from `goefi` to `go-uefi` as it makes
more sense and resonates better with the go libraries in the same space. Like
`go-tpm` and `go-attestation`.

{{< figure src="/img/arch.png" >}}

# Package Updates to [community]
- `podman` updated to `2.2.0-1`, `2.2.1-1` 
- `docker` updated to `1:19.03.14-1`, `1:19.03.14-2`
- `libslirp` updated to  `4.4.0-1`
- `python-google-api-python-client` updated to `1.12.8-1`
- `python-google-api-core` updated to `1.23.0-1`, `1.24.0-1`, `1.24.1-1`
- `python-pandas` updated to `1.1.4-1`, `1.1.5-1`
- `python-pipenv` updated to `2020.11.15-1`
- `python-pykka` updated to `2.0.3-1`
- `python-xlib` updated to `0.29-1`
- `python-jsonrpclib-pelix` updated to `0.4.2-1`
- `python-language-server` updated to `0.36.1-1`, `0.36.2-1` 
- `python-docs` updated to `3.9.0-1`, `3.9.1-1`
- `python-hidapi` updated to `0.10.1-1`
- `python-docker` updated to `4.4.0-1`, `4.4.1-1`
- `python-pyserial` updated to `3.5-1`
- `python-babel` updated to `2.9.0-1`
- `slirp4netns` updated to `1.1.8-1`
- `qmk` updated to `0.0.37-1`
- `github-cli` updated to `1.3.1-1`, `1.4.0-1`
- `go` updated to `2:1.15.6-1`
- `k9s` updated to `0.24.2-1`
- `glider` updated to `0.13.0-1`
- `qutebrowser` updated to `1.14.1-1`
- `gopls` updated to `0.5.5-1`, `0.6.0-1`, `0.6.1-1`
- `blueman` updated to `2.1.4-1`
- `fzf` updated to `0.24.4-1`
- `archlinux-contrib` updated to `20201205-1`
- `screenkey` updated to `1.3-1`
- `python-m2crypto` updated to `0.37.1-1`
- `dns-over-https` updated to `2.2.4-1`
- `delve` updated to `1.5.1-1`
- `helm` updated to `3.4.2-1`
- `cni-plugins` updated to `0.9.0-1`, `0.9.0-3`
- `plocate` updated to `1.1.2-1`, `1.1.3-1`
- `lxd` updated to `4.9-1`
- `git-lfs` updated to `2.13.1-1`
- `python-autobahn` updated to `20.12.1-1`, `20.12.2-1` 
- `staticcheck` updated to `2020.2-1`
- `python-sqlobject` updated to `3.9.0-1`
- `docker` updated to `1:20.10.1-1`
- `mopidy` updated to `3.1.0-1`, `3.1.1-1`
- `conmon` updated to `1:2.0.22-1`
- `darktable` updated to `2:3.5.0-1`
- `libmd` updated to `1.0.2-1`
- `python-reportlab` updated to `3.5.57-1`

## Package additions to [community]
- `python-adblock`: Allows `qutebrowser` to implement ABP style adblocking

### Potential new packages for `[community]`
- oomd
- vgrep
- git-publish
- b4
- psi-notify
- etcd
- micro
- tailscale
  - Is going to be uploaded to `[community]` through january.

# Bugfixes
- `qmk`: Removed the udev files in favour of the combined file file from upstream.
- `docker`: Fixed [FS#68833](https://bugs.archlinux.org/task/68833)
  - No more `-ce` embedded in the binary.
- `cni-plugins`: Ensure binaries are symlinked into `/opt/cni/bin`

## Security Team
This month the security team has published 26 advisories and created around 88
Advisory Groups. A lot of this work has been done by Jonas Witschel!

# Other things...
- `archlinux-contrib`
  - Merged a fix for broken `checkservices`: [#30](https://github.com/archlinux/contrib/pull/30)
  - `cleanup-list` now has suffix removed and takes input from stdin: [#28](https://github.com/archlinux/contrib/pull/28)
- `archlinux-repro`
  - Some issues for locking the chroots so need to rework the logic a bit. [#88](https://github.com/archlinux/archlinux-repro/issues/88)
  - We are having new(!) issues with the keyring and disabled keys. This time it's the expired keys backwards in time that gives us problems when we are peaking at them 4 months after expiry. [#87](https://github.com/archlinux/archlinux-repro/issues/87)

Cheers and a happy new year :)
