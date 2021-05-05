---
title: "FOSS Activities in April 2021"
date: "2021-05-05T21:00:00+02:00"
tags:
  - foss
  - monthly
  - archlinux
  - english
---

Yo!

Hope people have had a lovely spring. This month has passed quickly! I have
put off writing the monthly post because I was busy with a weekend project.

My master thesis was about how to apply transparency logs and reproducible
builds to give package rebuilders the ability to produce tamper evident logs.
This is handy since any one package build can easily be proven to be part of the
log, and you can very easily fill inn the history from one point in time to
another by hashing files in the correct order.

These days transparency logs has seen a larger adoption with projects like
sigstore and trustix. What's interesting is that kernel.org publishes a
transparency log of all the git push operations.

This is handy as it allows people to verify that a commit was actually pushed
from a kernel developer. This prevents cases where someone with access to
kernel.org can't create a commit on the server without it being possible to
detect. However, transparency logs are only really useful if people monitor
them for changes and replicate them. Thus i decided to quickly hack up a monitor
on the log to make the information easily digestible.

https://tlog.linderud.dev/

I'll probably continue to hack on this project a little bit to get some
statistics between kernel releases. Maybe some graphs? But currently it works
quite nicely to display the information on the log. A fun little project!

While talking about kernel security, the Linux TAB published their report on the
entire University of Minnesota fiasco. Whats interesting about the email is that
they linked the IEEE complaint Santiago, me and a few others wrote after reading
the abstract back in December. I have no clue how they got the link since the
draft pad was never shared outside of the twitter group we had. Weird 😀

https://hackmd.io/s/BJGs6Tfiw

https://lwn.net/ml/linux-kernel/202105051005.49BFABCE@keescook/

Other then that I have submitted a patch to Golang to default add full relro
flags when building with PIE and using the host compiler. I also did my first
patch to a kernel related mailing list to patch a bug in `b4`.

Rest of the work has mostly been to get my secure boot tooling up to speed so I
can move off from the preexisting Canonical tooling and better test. This has
resulted in `go-uefi`, my Golang native library for interactive with `efivarfs`
on Linux, having some form of integration tests towards OVMF. Which is great 😃

https://github.com/Foxboron/go-uefi/tree/master/tests

Lastly the work on debug packages in Arch has slowed down due to lack of
feedback on the patches. I wanted to have the stuff out in February but sadly
it's going to take longer. Disappointing but stuff happens.

Cheers and see you next month!

{{< figure src="/img/arch.png" >}}

# Package Updates to [community]
- `go` updated to `2:1.16.3-1`
- `salt` updated to `3003-1`, `3003-2`
- `plocate` updated to `1.1.6-1`, `1.1.7-1`
- `crun` updated to `0.19-1`, `0.19.1-1`
- `github-cli` updated to `1.8.1-1`, `1.9.1-1`, `1.9.2-1`
- `k9s` updated to `0.24.7-1`
- `qutebrowser` updated to `2.1.1-1`, `2.2.0-1`, `2.2.1-1`
- `python-nbxmpp` updated to `2.0.2-1`
- `python-nvxmpp` updated to `2.0.2-1`
- `fzf` updated to `0.27.0-1`
- `archlinux-repro` updated to `20210408-1`, `20210422-1`
- `docker-compose` updated to `1.29.0-1`, `1.29.1-2`
- `python-dotenv` updated to `0.17.0-1`
- `python-docker` updated to `5.0.0-1`
- `pdfjs` updated to `2.8.335-1`
- `yubico-pam` updated to `2.27-1`
- `python-nltk` updated to `3.6.1-1`
- `buildah` updated to `1.20.0-2`, `1.20.1-1`
- `python-docs` updated to `3.9.4-1`
- `lxd` updated to `4.13-1`
- `python-adblock` updated to `0.4.4-1`
- `sbctl` updated to `0.2-1`, `0.3-1`
- `lostfiles` updated to `4.11-1`
- `step-ca` updated to `0.15.14-1`
- `skopeo` updated to `1.2.3-1`
- `podman` updated to `3.1.1-1`, `3.1.1-2`, `3.1.2-1`
- `archlinux-contrib` updated to `20210418-1`
- `poke` updated to `1.2-1`
- `helm` updated to `3.5.4-1`
- `nvme-cli` updated to `1.14-1`
- `python-m2crypto` updated to `0.37.1-2`
- `python-psycopg2` updated to `2.8.6-4`
- `b4` updated to `0.6.2-2`
- `borg` updated to `1.1.16-3`
- `mopidy` updated to `3.1.1-3`
- `raft` updated to `0.10.1-1`
- `dqlite` updated to `1.7.0-1`
- `dunst` updated to `1.6.1-1`, `1.6.1-2`
- `lxcfs` updated to `4.0.8-1`
- `lxc` updated to `1:4.0.8-1`

## Package removals from [community]
- `python2-m2crypto`
- `python2-traitlets`
- `python2-prompt_toolkit1`
- `python2-gevent`
- `python2-pathlib`
- `python2-tarantool`
- `python2-psycopg2`

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
- `buildah`: [FS#70369](https://bugs.archlinux.org/task/70369)
- `podman`: [FS#70486](https://bugs.archlinux.org/task/70486)

# Other things...
- Support Full RELRO with `-buildmode=pie`: https://github.com/golang/go/pull/45681
- Patch b4: https://lore.kernel.org/tools/20210421202942.1358011-1-foxboron@archlinux.org/T/#u 
