---
title: "FOSS Activities in November 2020"
date: "2020-12-01T00:00:00+02:00"
tags:
  - foss
  - monthly
  - archlinux
  - english
---

Second month of doing these posts. In short not much has been happening the past
weeks, but that would be a slight lie.

I have sponsored rgacognes Trusted User application. The application was posted
to the [mailing list](https://lists.archlinux.org/pipermail/aur-general/2020-November/036017.html), and it's currently being voted and decided by a weeks time.

There has also been some discussion for *years* about bringing debug packages
into Arch. This has largely been stalled but I brought it back to life again.
Essentially the problem might be solved by utilizing the new [debuginfod](https://developers.redhat.com/blog/2019/10/14/introducing-debuginfod-the-elfutils-debuginfo-server/) project,
and we can later distribute the packages itself when we understand the new
mirror requirements. There is currently a discussion on [`[arch-dev-public]`](https://lists.archlinux.org/pipermail/arch-dev-public/2020-November/030222.html)
about it.

Along with the above, chugging along nicely with packages. Python has been
[rebuilt](https://lists.archlinux.org/pipermail/arch-dev-public/2020-November/030186.html) for the Python 3.9 release. This means there hasn't been as many python
package updates. Currently everything is in testing and we should see packages
move to the stable repositories early next week. I simply haven't been bothered
going through the hoops of releasing package updates into stable and then deal
with a rebuild for python 3.9 for [testing](https://lists.archlinux.org/pipermail/arch-dev-public/2020-November/030223.html).

In other news, `kubernetes` is now packaged into `[community]`. I'm probably doing
a write up on this later this week. Most of this work has been done by David
Runge, with me as supporting character. This also includes a reorganization of
the different container files between the Red Hat container ecosystem. The files
has been spread among `container`, `buildah` and `skopeo`. This gives weird
dependencies such as `containerd` depending on `skopeo`. But this is only for the
files it provides. These files have been moved to `containerd-common` and
dependant packages has been (mostly) updated.

For questions or suggestions on this post, please reach to me on morten@linderud.pw.

{{< figure src="/img/arch.png" >}}

# Package Updates to [community]
- `k9s` updated to `0.23.3-1`, `0.23.4-1`, `0.23.9-1`, `0.23.10-1`, `0.24.1-1`
- `buildah` updated to `1.17.0-1`, `1.18.0-1`
- `perl-type-tiny` updated to `0.012000`
- `python-reportlab` updated to `3.5.55-1`
- `python-pygame` updated to `2.0.0-1`
- `archlinux-contrib` updated to `20201101-1`, `20201108-1`
- `toolbox` updated to `0.0.97-1`
- `crun` updated to `0.15.1-1`
- `python-language-server` updated to `0.36.0-1`
- `fzf` updated to `0.24.2-1`, `0.24.3-1`
- `slirp4netns` updated to `1.1.6-1`
- `git-lfs` updated to `2.12.1-1`
- `restic` updated to `0.11.0-1`
- `go` updated to `1.15.4-1`, `1.15.5-1`, `1.15.5-2`
- `helm` updated to `3.4.1-1`
- `github-cli` updated to `1.2.1-1`, `1.3.0-1`
- `gopls` updated to `0.5.3-1`
- `i3-gaps` updated to `4.19-1`
- `rclone` updated to `1.53.3-1`
- `v2ray` updated to `4.33.0-1`
- `dns-over-https` updated to `2.2.3-1`
- `rofi` updated to `1.6.1-1`
- `crun` updated to `0.16-1`
- `slirp4netns` updated to `1.1.7-1`
- `fuse-overlayfs` updated to `1.3.0-1`
- `containerd` updated to `1.4.2-1`, `1.4.2-2`, `1.4.3-1`
- `lostfiles` updated to `4.08-1`

## Package additions to [community]
- kubernetes
  - David did most of the work, but tested and adopted!

## Package adoptions in [community]
- python-google-api-core
- python-google-api-python-client
- python-pandas
- python-pandas-datareader
- docker

### Potential new packages for `[community]`
- oomd
- vgrep
- git-publish
- b4
- psi-notify
- etcd
- micro
- adblock stuff for qutebrowser

## Bugfixes
- [FS#68456](https://bugs.archlinux.org/task/68456): `fzf` had RELRO removed as part of the move to use the Makefile
- [FS#68271](https://bugs.archlinux.org/task/68271): containerd got signatures
- `go`: With a recent security issue in the compiler, LDFLAGS was restrictied to such a degree go packages wouldn't be built with `CGO_LDFLAGS`.  [#42565](https://github.com/golang/go/issues/42565) solves the problem.

## Security Team
We released a total of 29 advisories the past month.

## Other things...
- Python rebuilds has started!
  - Python rebuilds also managed to hit `[testing]` during the same month :)
- Release `archlinux-repro` `20201114`
  - Reworked the container handling
- Patched `docker` to support `SOURCE_DATE_EPOCH` when building
  - https://github.com/docker/cli/pull/2855

## Presentations
- Published my presentation from Arch Conf 2020 about Reproducible Builds.
  - https://media.ccc.de/v/arch-conf-online-2020-6308-the-state-of-reproducible-builds 
- Did a presentation about reproducible builds for the local Linux User Group (Norwegian).
  - https://www.youtube.com/watch?v=Tzc8arUBiRM 



Cheers!
