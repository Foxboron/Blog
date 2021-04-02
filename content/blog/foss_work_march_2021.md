---
title: "FOSS Activities in March 2021"
date: "2021-04-02T14:00:00+02:00"
tags:
  - foss
  - monthly
  - archlinux
  - english
---

Yoooo!

Another month has passed which means another status update.

The python2 removal has been steady and several packages has been removed this
month. Currently a query for `python2` on archweb returns 139 matches. At the
start of the month it was around 160-170. Progress! 

I have suggested we [remove `checkdepends`](https://lists.archlinux.org/pipermail/arch-dev-public/2021-March/030390.html) on python2 packages to ease the cleanup of dependency cycles. The
response has been lukewarm at best so we'll see how that progresses. Hopefully
more is being removed in the upcoming months.

Support for debug in Arch Linux is in large parts written and under review. The
dbscripts patches, which is how we administer packages to our repositories, has
all the needed [patches](https://github.com/archlinux/dbscripts/pull/21) and a passing test suite! But it is currently missing a
review from the current maintainer. I have also [shaped up the infrastructure](https://gitlab.archlinux.org/archlinux/infrastructure/-/merge_requests/168)
part so we can provide `debuginfod`. The general goal is to start providing
debug packages through `debuginfod` and at a later date decide if we want to
distribute the packages to some or all mirrors.

Hopefully it's just a few weeks left until we can provide this to our users!

Arch recent got a new [RFC process](https://gitlab.archlinux.org/archlinux/rfcs/-/blob/master/rfcs/0001-using-rfcs.rst) which is intended to create more structure
around changes to the distribution. This is a neat change and there have already
been 3 RFCs up for discussion this month from Allan.

Something that was brought up [in 2019](https://lists.archlinux.org/pipermail/arch-dev-public/2019-October/029695.html) was to move our `license` field in
packages from loosely defined strings to [SPDX identifiers](https://spdx.dev/ids/#how). This allows us to
standardize our license fields and allow people to better figure out which
license a package has. This is going to a lot of work as we need to restructure
our current license package with [SPDX identifiers](https://spdx.org/licenses/index.html), update the archwiki license
guidelines and formulate an RFC. Hopefully we'll have some work done on this
through April and an RFC before next month :)

And for the polarizing news. Richard Stallman was secretly reelected back on the
board of the FSF. I personally think this is a move that firmly cements the Free
Software Foundation as organization suffering from ["Founder's Syndrome"](https://en.wikipedia.org/wiki/Founder%27s_syndrome) and
incapable of modernizing. FSF is struggling with relevance and replacing the
board, along with Stallman, is a better course of action to ensure the
organization doesn't die out.

I have signed the open letter along with several Arch maintainer and people I
have got to know over the years which got me into FOSS development in the first
place. 

https://rms-open-letter.github.io/

And last I'd like to congratulate Kristian Klausen for being accepted as [junior
devops](https://lists.archlinux.org/pipermail/arch-devops/2021-March/000502.html). They have done great work 
for Arch Linux on the infrastructure side of things and I hope there are more
things to come :)

Cheers and happy holidays!

{{< figure src="/img/arch.png" >}}

# Package Updates to [community]
- `bash-bats` updated to `1.2.1-2`
- `docker` updated to `1:20.10.5-1` 
- `lxd` updated to `4.12-1`, `4.12-2`
- `github-cli` updated to `1.7.0-1`, `1.7.0-2`, `1.8.0-1`
- `udiskie` updated to `2.3.3-1`
- `buildah` updated to `1.19.7-1`, `1.19.8-1`, `1.20.0-1`
- `python-google-api-core` updated to `1.26.1-1`
- `plocate` updated to `1.1.5-3`, `1.1.5-4`
- `cni-plugins` updated to `0.9.1-3`
- `qmk` updated to `0.0.40-1`, `0.0.45-1`
- `v2ray` updated to `4.35.1-1`
- `python-docs` updated to `3.9.2-1`
- `go` updated to `2:1.16.1-1`, `2:1.16.2-1`
- `conmon` updated to `1:2.0.27-1`
- `staticcheck` updated to `2020.2.3-1`
- `bash-bats` updated to `1.3.0-1`
- `python-reportlab` updated to `3.5.65-1`
- `python-pandas` updated to `1.2.3-1`
- `python-sqlobject` updated to `3.9.1-1`
- `python-prompt_toolkit` updated to `3.0.17-1`, `3.0.18-1`
- `qutebrowser` updated to `2.1.0-1`
- `lostfiles` updated to `4.10-1`
- `borg` updated to `1.1.15-1`, `1.1.15-1`, `1.1.16-1`
- `tailscale` updated to `1.4.6-1`, `1.6.0-1`
- `python-language-server` updated to `0.36.2-3`
- `helm` updated to `3.5.3-1`
- `python-pyserial` updated to
- `nageru` updated to `1.8.6-9`
- `font-awesome` updated to `5.15.3-1`
- `gopass` updated to `1.12.4-1`, `1.12.5-1`
- `python-adblock` updated to `0.4.3-1`
- `poke` updated to `1.1-1`
- `step-ca` updated to `0.15.10-1`, `0.15.11-1`
- `k9s` updated to `0.24.3-1`, `0.24.6-1`
- `python-reportlab` updated to `3.5.66-1`
- `yubikey-manager-qt` updated to `1.2.0-1`, `1.2.1-1`
- `podman-dnsname` updated to `1.1.1-1`, `1.2.0-1`
- `dns-over-https` updated to `2.2.4-2`, `2.2.5-1`
- `git-lfs` updated to `2.13.2-2`, `2.13.3-1`
- `go-md2man` updated to `2.0.0-4`
- `runc` updated to `1.0.0rc93-2`
- `fuse-overlayfs` updated to `1.5.0-1`
- `k9s` updated to `0.24.4-1`
- `raft` updated to `0.10.0-1`
- `salt` updated to `3002.6-1`
- `python-milc` updated to `1.2.0-1`, `1.3.0-1`
- `podman` updated to `3.1.0-1`


## Package additions to [community]
- `podman-dnsname`
  - For proper `docker-compose` support with `podman`.
- `python-dotty-dict`
  - New dependency for `qmk`

## Package removals from [community]
- `python2-bcrypt`
- `syncthing-gtk`
- `python2-pyserial`
- `python2-futures`
- `python2-tornado`
- `dep`


### Potential new packages for 
- `oomd`
- `vgrep`
- `git-publish`
- `psi-notify`
- `etcd`
- `gosec`
- `kind`
- `nomad`
- `distrobuilder`
- `hunspell-nb`
- `hunspell-nn`
- `magic-wormhole`

# Bugfixes
- `bash-bats`: [FS#63099](https://bugs.archlinux.org/task/63099)
- `github-cli`: [FS#69787](https://bugs.archlinux.org/task/69787)
- `lxd`: [FS#69352](https://bugs.archlinux.org/task/69352)
- `plocate`: [FS#69884](https://bugs.archlinux.org/task/69884)
- `cni-plugins`: [FS#69626](https://bugs.archlinux.org/task/69626)
- `gopass`: [FS#70097](https://bugs.archlinux.org/task/70097)
- `jp2a`: [FS#6997](https://bugs.archlinux.org/task/6997)
- `qmk`: [FS#69908](https://bugs.archlinux.org/task/69908)
- `podman`: [FS#70087](https://bugs.archlinux.org/task/70087)

## Security Team
The security team has released 27 advisories. The most notable this month has been the OpenSSL
security advisory [ASA-202103-10](https://security.archlinux.org/ASA-202103-10).

# Other Things
Helped fix the ["granite 6.0.0 rebuild"](https://archlinux.org/todo/granite-600-rebuild/) which was blocking
the Gnome 40 update. This resulted in 2 patches for some vala code I have no
clue about.
- [torrential](https://github.com/davidmhewitt/torrential/commit/dcbdc86646268236dd30a938f0ac24224a02a310)
- [vocal](https://github.com/needle-and-thread/vocal/issues/483)
