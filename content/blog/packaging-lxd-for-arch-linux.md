---
title: "Packaging LXD for Arch Linux"
date: 2020-04-28T08:00:00+01:00
tags:
  - Arch Linux
  - english
  - reproducible builds
  - packaging
---

With the release of 3.20, LXD was included into the
[community](https://wiki.archlinux.org/index.php/Official_repositories#community)
repository of Arch Linux in January, and has currently been sitting there
happily for the past months. LXD is a container manager from Canonical that
manages containers as if they where independent machines in a cluster. I have
somehow taken to calling them "containers-as-machines". This is in contrast to
podman and docker which would be "containers-as-applications". Think of lxd as
[ganeti](http://www.ganeti.org/), but for containers.

As canonical is developing the project, and they only target
[snap](https://snapcraft.io/) packages downstream, it takes quite a few
liberties with dependencies and vendored projects which makes for an interesting
package challenge.

I'm not old enough to have jammed with the console cowboys in cyberspace during
[the 90s](https://www.youtube.com/watch?v=BNtcWpY4YLY). Thus the linker magic
presented here might be painly obvious for some people, however this is a
learning exercise!  Hopefully this encourages people to try their hands at
packaging without thinking they need to know everything, or so my imposter
syndrome keeps telling me.

## Investigating the project

When you are looking to package something the ideal place to start is with the
top-level package, and work your way down. The first step is to read the
documentation.

Opening the [lxd repository](https://github.com/lxc/lxd) You end up somewhere
midway in the README when you see "From Source: Building the latest version",
which is what we are after. 

> [...] a specific release of LXD which may not be offered by their Linux distribution. Source
builds for integration into Linux distributions are not covered here and may be
covered in detail in a separate document in the future.

Sure, just tell me the dependencies, and we can deal with that.

> When building from source, it is customary to configure a GOPATH which contains
the to-be-built source code. [...] and with a little LD_LIBRARY_PATH
magic (described later), these binaries can be run directly from the built
source tree.

A little what-are-you-talking-about now?

*/me scrolls* 

{{< highlight bash >}}
cd lxd-3.18
export GOPATH=$(pwd)/_dist
{{< /highlight >}}
<!-- _ because it's annoying to miss hilight after the underscore -->

Uh...

{{< highlight bash >}}
make deps
export CGO_CFLAGS="${CGO_CFLAGS} -I${GOPATH}/deps/sqlite/ -I${GOPATH}/deps/dqlite/include/ -I${GOPATH}/deps/raft/include/ -I${GOPATH}/deps/libco/"
export CGO_LDFLAGS="${CGO_LDFLAGS} -L${GOPATH}/deps/sqlite/.libs/ -L${GOPATH}/deps/dqlite/.libs/ -L${GOPATH}/deps/raft/.libs -L${GOPATH}/deps/libco/"
export LD_LIBRARY_PATH="${GOPATH}/deps/sqlite/.libs/:${GOPATH}/deps/dqlite/.libs/:${GOPATH}/deps/raft/.libs:${GOPATH}/deps/libco/:${LD_LIBRARY_PATH}"
make
{{< /highlight >}}

### Wat ಠ_ಠ

This isn't suppose to be easy. Is it[[^1]]?


Unesting this flag magic, we know the vendored dependencies LXD vendor is:

* sqlite
* raft
* libco
* dplite

Vendoring is generally considered bad for a number of reasons. The most
important one is security. Having Z number of upstream vendors with N versions
of dependency X makes security patching and updating a nightmare for
distributions. And you can't rely on upstream tracking security issues on their
dependencies, this is hard enough for security teams today.

The second one is dependencies themselves. If we where to package this as-is,
dqlite, raft and libco would be provided by the LXD package. Inaccessible or
hard to use for other packages and users. And what if two packages vendored
dqlite? Which package should provide it?

What struck me as most interesting is the fact that sqlite is vendored. Why on
earth is it vendored? Most distributions should depend on sqlite fairly early in
their dependency chain.

# sqlite, sqlite-replication and dqlite

Turns out canonical has forked sqlite to provide a patch adding Write-Ahead
logging replication. The details are a bit beyond me as I don't know C very
well. The strange thing is that it still utilizes the same so-name and
pkg-config file as upstream sqlite. Which is a problem considering they are ABI
incompatible. I should also check, but it would be interesting to verify if
there has been attempt at upstreaming these patches.

`sqlite-replication` is used as a dependency for `dqlite` which is essentially a
distributed sqlite database with a [raft consensus
protocol](https://raft.github.io/). Rest of the dependencies are somehow
unremarkable. `raft` is a C implementation of said protocol, and `libco` is an
maintained out-of-tree fork of [byuus
libco](https://github.com/byuu/higan/tree/master/libco). Neither of these are
interesting to at look further in this post, and was fairly straight forward to
package.

Since `sqlite-replication` built normally is going to have conflicting files
with `sqlite`, we need to build any headers and libraries into it's own path.
This can be done in the configure step of the software.

{{< highlight bash >}}
./configure --prefix=/usr \
  --libdir=/usr/lib/$pkgname \
  --includedir=/usr/include/$pkgname \
  [snip]
# Note: `$pkgname` is a shorthand for `sqlite-replication`.
{{< /highlight >}}


The resulting header files would reside in `/usr/include/sqlite-replication`,
so-names in `/usr/lib/sqlite-replication` and the `pkg-config` file is going to
reside in `/usr/lib/sqlite-replication/pkgconfig`.  `dqlite` needs to be aware
of the extra path to look for linker files and libraries. We can abuse
environment files a little bit for this.

{{< highlight bash >}}
  PKG_CONFIG_PATH="/usr/lib/$pkgname/pkgconfig" ./configure --prefix=/usr
  make LDFLAGS="-Wl,-R/usr/lib/$pkgname"
{{< /highlight >}}

And this seems to work fine[[^2]]. As far as I got from the documentation is
that `-Wl,-R<path>` rewrites the included search path as it's interpreted as
`-rpath` when a complete directory is added. I think.

{{< highlight bash >}}
λ ~ » ldd /usr/lib/libdqlite.so.0.0.1 | grep replication
	libsqlite3.so.0 => /usr/lib/libsqlite3.so.0
# with -rpath:
	libsqlite3.so.0 => /usr/lib/sqlite-replication/libsqlite3.so.0
{{< /highlight >}}

Nice, this enables us to verify that we build `dqlite` linked towards the
patched `sqlite` provided by Canonical.


# Golang and linkers

Now comes the fun part, lets build `lxd`! In contrast to the other projects we
have looked at, lxd is written in go. Cgo is a special runtime in go which is
used to interface with C code, it supports it's own [set of `CGO_*`
variables](https://golang.org/cmd/cgo/#hdr-Using_cgo_with_the_go_command) that
mirrors the gcc/ld flags.

So lets try with the buildflags we used earlier. This should make `pkg-config`
capable of finding the linker flags needed, along with the `-rpath` argument to
ensure we have the correct ld path.

{{< highlight bash >}}
  export PKG_CONFIG_PATH="/usr/lib/sqlite-replication/pkgconfig" 
  export CGO_LDFLAGS="-Wl,-R/usr/lib/sqlite-replication"
{{< /highlight >}}

And the results are...

{{< highlight bash >}}
# github.com/canonical/go-dqlite/internal/bindings
/usr/bin/ld: /usr/lib/gcc/x86_64-pc-linux-gnu/9.3.0/../../../../lib/libdqlite.so: undefined reference to `sqlite3_wal_replication_frames'
/usr/bin/ld: /usr/lib/gcc/x86_64-pc-linux-gnu/9.3.0/../../../../lib/libdqlite.so: undefined reference to `sqlite3_wal_replication_enabled'
/usr/bin/ld: /usr/lib/gcc/x86_64-pc-linux-gnu/9.3.0/../../../../lib/libdqlite.so: undefined reference to `sqlite3_wal_replication_leader'
/usr/bin/ld: /usr/lib/gcc/x86_64-pc-linux-gnu/9.3.0/../../../../lib/libdqlite.so: undefined reference to `sqlite3_wal_replication_undo'
/usr/bin/ld: /usr/lib/gcc/x86_64-pc-linux-gnu/9.3.0/../../../../lib/libdqlite.so: undefined reference to `sqlite3_wal_replication_unregister'
/usr/bin/ld: /usr/lib/gcc/x86_64-pc-linux-gnu/9.3.0/../../../../lib/libdqlite.so: undefined reference to `sqlite3_wal_replication_follower'
/usr/bin/ld: /usr/lib/gcc/x86_64-pc-linux-gnu/9.3.0/../../../../lib/libdqlite.so: undefined reference to `sqlite3_wal_replication_register'
{{< /highlight >}}
<!-- ` because it's annoying to miss hilight -->

Interesting. We didn't find the library! So admittedly I'm a bit unsure what is
going on. `libdqlite.so` should be looking for the correct library, and the
library should be present. However considering we are dealing with some bindings
magic, I looked up [the
code](https://github.com/canonical/go-dqlite/blob/master/internal/bindings/build.go)
which `canonical/go-dqlite` utilizes and found the following LDFLAGS.

{{< highlight go >}}
/*
#cgo linux LDFLAGS: -lsqlite3 -lraft -lco -ldqlite
*/
{{< /highlight >}}

What I *think* is happening here is `-lsqlite3` causes dqlite to assume the
correct so-name has been found, and disregards looking for it anywhere else. It
is a bit weird that we are dealing with `pkg-config` [inside
lxd](https://github.com/lxc/lxd/blob/master/lxd/cgo.go) which behaves
differently. However, I did figure out something in the end!

{{< highlight bash >}}
export CGO_CFLAGS="-I/usr/include/sqlite-replication"
export CGO_LDFLAGS="-L/usr/lib/sqlite-replication -Wl,-R/usr/lib/sqlite-replication"
{{< /highlight >}}

What I wound up doing in the end is to include the `-rpath` stuff from
`sqlite-replication` and riffing on the flags provided by upstream for their
vendoring. It seems to work correctly and I haven't recieved any bugreports
suggesting otherwise. After getting this to work I did discover this is exactly
what Void Linux does, so oh-well. Should probably read other distribution
package files a bit better.

And a side note; `libcap` [introduced a hard
dependency](https://sites.google.com/site/fullycapable/release-notes-for-libcap/releasenotesfor228pending)
on an *optional* `libpsx` library. This file contained several flags which the
Golang compiler denies by default. You need to explicitly whitelist them when
building. This was [removed in a later
release](https://git.kernel.org/pub/scm/libs/libcap/libcap.git/commit/?id=f9d1c5ee19c96547fad2c807270e82abc9426ff8)
I discovered while backtracing the how I packaged this. This was [noted in the
build instruction](https://github.com/lxc/lxd/issues/6727) for LXD as both Void
Linux and us encountered this issue.

The whitelisting can be fixed by utilizing the `CGO_LDFLAGS_ALLOW`.
{{< highlight bash >}}
export CGO_LDFLAGS_ALLOW='-Wl,-wrap,pthread_create'
{{< /highlight >}}

After this fumbling, which is honestly is the result of two months on-and-off
work now condensed into a 10 minute long read, it works! LXD was packaged
properly and been a happy LXD user.

The resulting PKGBUILD can be found in the repository for
[LXD](https://git.archlinux.org/svntogit/community.git/tree/trunk/PKGBUILD?h=packages/lxd)
and [sqlite-replication](https://git.archlinux.org/svntogit/community.git/tree/trunk/PKGBUILD?h=packages/sqlite-replication)!

# Reproducible Builds

I don't think this is a complete blog post about packaging without looking a
little on [reproducible builds](https://reproducible-builds.org/). The goal is
to reproduce the bit-for-bit identical package which I build and distribute into
the Arch Linux mirrors. For this we can use
[archlinux-repro](https://github.com/archlinux/archlinux-repro), and the
technical details can be [read in the blog
post](/blog/reproducible-arch-linux-packages/) I wrote last year.

Can we reproduce the packages from his blog post?


{{< highlight bash >}}
λ » repro sqlite-replication-3.31.1.4-1-x86_64.pkg.tar.zst
[...snip....]
==> Finished making: sqlite-replication 3.31.1.4-1 (Mon 27 Apr 2020 11:59:03 PM CEST)
[...]
==> Comparing hashes...
==> Package is reproducible!
{{< /highlight >}}

{{< highlight bash >}}
λ » repro raft-0.9.18-1-x86_64.pkg.tar.zst
[...snip....]
==> Finished making: raft 0.9.18-1 (Tue 28 Apr 2020 12:01:03 AM CEST)
[...]
==> Comparing hashes...
==> Package is reproducible!
{{< /highlight >}}

{{< highlight bash >}}
λ » repro libco-20-2-x86_64.pkg.tar.zst
[...snip....]
==> Finished making: libco 20-2 (Tue 28 Apr 2020 12:03:02 AM CEST)
[...]
==> Comparing hashes...
==> Package is reproducible!
{{< /highlight >}}

{{< highlight bash >}}
λ » repro dqlite-1.4.1-1-x86_64.pkg.tar.zst
[...snip....]
==> Finished making: dqlite 1.4.1-1 (Tue 28 Apr 2020 12:04:17 AM CEST)
[...]
==> Comparing hashes...
==> Package is reproducible!
{{< /highlight >}}

{{< highlight bash >}}
λ » repro lxd-4.0.1-1-x86_64.pkg.tar.zst
[...snip....]
==> Finished making: lxd 4.0.1-1 (Tue 28 Apr 2020 12:10:55 AM CEST)
[...]
==> Comparing hashes...
==> Package is reproducible!
{{< /highlight >}}

Yes we can 👯🥂! Amazing!

I think it's hard to convey the importance of this. This is the combined work
between multiple distributions for well over 5 years now. We are capable of
recreating bit-for-bit identical packages to the ones distributed in the Arch
Linux repositories!

# Other distributions!

The interesting part now is to figure out what different distributions are
doing to package LXD.

* Debian doesn't package LXD because of the [Go dependency packaging](https://gobby.debian.org/export/Teams/Go/lxd-dep-packaging).
* [Alpine](https://gitlab.alpinelinux.org/alpine/aports/blob/master/testing/lxd/APKBUILD) seems content with packaging LXD as-is.
* [OpenSUSE](https://github.com/bmwiedemann/openSUSE/blob/master/packages/l/lxd/lxd.spec) seems also content with packaging it as-is.
* And [Gentoo](https://gitweb.gentoo.org/repo/gentoo.git/tree/app-emulation/lxd/lxd-3.16-r1.ebuild) seems content with packaging it as-is.
* [Void Linux](https://github.com/void-linux/void-packages/blob/master/srcpkgs/lxd/template) and [NixOS](https://github.com/NixOS/nixpkgs-channels/blob/nixos-unstable/pkgs/tools/admin/lxd/default.nix) packages it correctly by splitting up all the dependencies.
* I where unable to locate the Fedora package which is suppose to exist *somewhere*.

The intention isn't to name and shame. Different distributions has other
priorities and philosophies. However it is also interesting to compare what you
are getting with the distributed LXD package across repositories.


[^1]: I should probably disclose that at this point I had read the existing [AUR package for
LXD](https://aur.archlinux.org/cgit/aur.git/tree/PKGBUILD?h=lxd). I was however
convinced they where the ones making shortcuts. Not Canonical.

[^2]: I realized while writing this that I had specified
`LDFLAGS="-L/usr/lib/sqlite-replication"` in the call to make in the current
PKGBUILD. This shouldn't be needed as it's what the `pkg-config` file provides.
