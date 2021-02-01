---
title: "FOSS Activities in January 2021"
date: "2021-02-01T00:21:00+02:00"
tags:
  - foss
  - monthly
  - archlinux
  - english
---

And January is over! Time has frankly been moving fast the past days.

Packaging wise, things has been fine. Added `tailscale` and some other minor
packages, but had a real purge of old packages from resigned maintainers. Also
dropped `ntop` to the AUR which hasn't been actively developed for years at this
point. I'm curious when people are going to bug me about that one :)

On the security side of things there has been quite a lot happening just the
past week. `sudo` had [CVE-2021-3156](https://security.archlinux.org/CVE-2021-3156) and `libgcrypt` had [CVE-2021-3345](https://security.archlinux.org/CVE-2021-3345) which are
both are quite severe. My personal take is that the `sudo` one bad, but not *that*
bad. While `libgcrypt` is a bit more terrible considering the data is parsed
before it's authenticated. However both was patched fairly quickly in Arch.

In other news things are moving fine on different projects. Did some progress on
finishing up `debuginfod` on the infrastructure side. Not that much happening on
the `dbscripts` side of things. Slowly been hacking away on some CVE tracking
for the security team, but the CVE dataset is a mess. Among more exciting stuff
I have [submitted](https://github.com/WerWolv/ImHex-Patterns/pull/8) some WIP UEFI structs for ImHex which is a neat hex editor.

I have 3-4 blog posts about Arch Conf and `archlinux-repro` rework that I have
in the pipeline. But been a bit lazy in the evenings rewatching Halt and Catch
Fire (which is a terrific TV show).

Other then that I have a talk about `sbctl` this weekend for FOSDEM.
[Improving the Secure Boot landscape: sbctl & go-uefi](https://fosdem.org/2021/schedule/event/firmware_itsblsg/).

Enjoy this weeks summary and stay safe!

{{< figure src="/img/arch.png" >}}

# Package Updates to [community]
- `python-milc` updated to `1.0.12-1`, `1.0.13-1`
- `python-reportlab` updated to `3.5.59-1`, `3.5.60-1`
- `python-pygame` updated to `2.0.1-1`
- `python-typed-ast` updated to `1.4.2-1`
- `v2ray` updated to `4.34.0-1` 
- `fzf` updated to `0.25.0-1`
- `jgmenu` updated to `4.3.0-1`
- `libmd` updated to `1.0.3-1`
- `plocate` updated to `1.1.3-2`
- `toolbox` updated to `0.0.98-1`, `0.0.98.1-1`, `0.0.99-1`
- `gopls` updated to `0.6.2-1`, `0.6.3-1`, `0.6.4-1`
- `python-prompt_toolkit` updated to `3.0.9-1`, `3.0.11-1`, `3.0.13-1`, `3.0.14-1`
- `minicom` updated to `2.8-1`
- `lxd` updated to `4.10-1`
- `python-prompt_toolkit` updated to `3.0.10-1`
- `python-pandas` updated to `1.2.0-1`
- `buildah` updated to `1.19.0-1`, `1.19.2-1`, `1.19.3-1`
- `skopeo` updated to `1.2.1-1`
- `gopass` updated to `1.11.0-1`
- `qmk` updated to `0.0.39-1`
- `perl-type-tiny` updated to `1.012001-1` 
- `git-lfs` updated to `2.13.2-1`
- `font-awesome` updated to `5.15.2-1`
- `conmon` updated to `1:2.0.23-1`, `1:2.0.24-1`, `1:2.0.25-1`
- `helm` updated to `3.5.0-1`, `3.5.1-1`
- `buildah` updated to `1.19.2-1`
- `python-google-api-core` updated to `1.25.0-1`, `1.25.1-1`
- `docker-compose` updated to `1.28.0-1`, `1.28.2-1`
- `nvme-cli` updated to `1.13-2`
- `go` updated to `2:1.15.7-1`
- `staticcheck` updated to `2020.2.1-1`
- `github-cli` updated to `1.5.0-1`
- `fuse-overlayfs` updated to `1.4.0-1`
- `crun` updated to `0.17-1`
- `udiskie` updated to `2.3.0-1`, `2.3.2-1`
- `screenkey` updated to `1.4.1-1`
- `yubikey-manager` updated to `3.1.2-1`
- `hy` updated to `0.20.0-1`
- `pdfjs` updated to `2.7.570-1`
- `cni-plugins` updated to `0.9.0-5`
- `perl-gnupg-interface` updated to `1.01-1`
- `qutebrowser` updated to `2.0.0-1`, `2.0.1-1`
- `tailscale` updated to `1.4.0-1`, `1.4.1-1`
- `python-adblock` updated to `0.4.1-1`
- `delve` updated to `1.6.0-1`
- `step-cli` updated to `0.15.3-1`

## Package additions to `[community]`
- `tailscale`
- `step-ca`

## Package removals to `[community]`
- `eric`
- `eric-i18n`
- `pychecker`
- `canorus`
- `julius`
- `libasl`
- `python-pmw`
- `libmatio`      
- `pdfsam`
- `voxforge-am-julius`
- `g15daemon`
- `libg15`
- `libg15render`
- `libco`
- `sqlite-replication`
- `ntop`

### Potential new packages for `[community]`
- `oomd`
- `vgrep`
- `git-publish`
- `b4`
- `psi-notify`
- `etcd`
- `micro`

# Bugfixes
- `plocate`: Should enable the timer, nor the service. Slight typi there!
- `nvme-cli`: Fixed [FS#69374](https://bugs.archlinux.org/task/69374)
  - Some dracut warning. Backported upstream patch.
- `cni-plugins`: Fixed [FS#69276](https://bugs.archlinux.org/task/69276)
  - Had to copy the binaries into `/opt/cni/bin`
- `hy`: Fixed [FS#69390](https://bugs.archlinux.org/task/69390)
  - Removed `python-clint` for `python-colorama`

## Security Team
We have published 45 advisories this month. We have in total published 1000
advisories since the tracker was deployed in 2016!

# Other things...
- `archlinux-repro`
  - Merged a few patches this months!
- `pacman`:
  - [pacman-key: Close msg string in generate_master_key](https://lists.archlinux.org/pipermail/pacman-dev/2021-January/024797.html)
  - [pacman-key: --refresh-keys queries WKD before keyserver](https://lists.archlinux.org/pipermail/pacman-dev/2021-January/024798.html)
- `infrastructure`:
  - [Draft: debuginfod: Implement role](https://gitlab.archlinux.org/archlinux/infrastructure/-/merge_requests/168)
  - Continued the work on the `debuginfod` role.


Cheers!
