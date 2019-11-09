---
title: "Reproducible Arch Linux Packages"
date: 2019-11-09T11:44:32+01:00
draft: true
---


# Reproducible Arch Linux packages

Arch Linux has been involved with the reproducible builds efforts since 2016. The goal is to achieve deterministic building of software packages to enhance the security of the distribution.

After almost 3 years of efforts, along with the release of pacman 5.2 and contributions from a lot of people, we are finally able to reproduce packages distributed by Arch Linux!

This enables users to build packages and compare them with the ones distributed by the Arch Linux team. Users can independently verify the work done by our packagers, and figure out if malicious code has been included in the pristine source during building, which in turns enhances the overall supply chain security. 

That was the tl;dr! The rest of the blog post will explain the reproducible builds efforts, and the technical work that has gone into achieving this.

### Reproducible Builds

The [reproducible builds](https://reproducible-builds.org/) effort was started by Debian in 2013 and currently encompasses [several projects](https://reproducible-builds.org/who/) across the spectrum from coreboot, Bitcoin, NixOS and F-Droid to mention a few of them.

The goal of the effort is to figure out what makes software projects undeterministic, and come up with standards to solve these issues.

Two of these standards are [`SOURCE_DATE_EPOCH`](https://reproducible-builds.org/specs/source-date-epoch/) to ensure timestamps can be overridden during building, and [`BUILD_PATH_PREFIX_MAP`](https://reproducible-builds.org/specs/build-path-prefix-map/) to ensure we can consistently trim build paths across toolchains.

Another project they work on is a Continuous Integration (CI) framework to test reproducible packages. The Debian one is probably more interesting to [browse](https://tests.reproducible-builds.org/debian/reproducible.html), but Arch Linux has a [CI environment](https://tests.reproducible-builds.org/archlinux/archlinux.html) for the past two years which has been consistently rebuilding packages.

However, the CI does not address the real issue. It is a very nice tool for debugging and uncovering issues that cause undeterministic builds. As Holger explained on the debian-devel mailing list [earlier this year](https://lists.debian.org/debian-devel/2019/03/msg00017.html), "these tests are done without looking at the actual .deb files distributed from ftp.debian.org (and we always knew that and pointed it out: "93% reproducible _in our current test framework_")". We are essentially building the same set of files twice in different environments.

Ensuring software projects can be deterministically built is all fine and dandy, but we want to enable users the ability to verify the work package maintainers are doing.


### Reproducing packages

There are a few components needed to ensure packages can be rebuild. We need to record the environment of the package build. This needs to be build paths, timestamps, all dependencies installed on the system and a bunch of other information. This is standardized in different formats [across projects](https://reproducible-builds.org/docs/recording/), but the format used for pacman can be found in [`BUILDINFO(5)`](https://www.archlinux.org/pacman/BUILDINFO.5.html).

One can explore a BUILDINFO file from any package by extracting the top-level file.
```shell
bsdtar -xOf archlinux-keyring-20191018-1-any.pkg.tar.xz .BUILDINFO
```

Pacman also keeps tracks of some metadata inside `.PKGINFO`. On the surface this seems trivial, record some information and put it into a file. Like "How much space does this package take?". But counting file sizes is [a problem](https://bugs.archlinux.org/task/61717). [Counting](https://git.archlinux.org/pacman.git/commit/?id=b264fb9e9ddcc31dc8782390309421965e507383
) [file](https://git.archlinux.org/pacman.git/commit/?id=7f258619c6c0e9f441aacbabfc1a2f5980c5cb9b) [sizes](https://git.archlinux.org/pacman.git/commit/?id=3f1ea8b62f46a915c94a5b46e21ad39ea2628f65) is [hard](https://git.archlinux.org/pacman.git/commit/?id=241d6b884a3a6c883b6c61a3b175d17e7d317fc5). Like, [very](https://git.archlinux.org/pacman.git/commit/?id=f26cb61cb6a16c8ce85f33e6090763aced0118c3) [hard](https://git.archlinux.org/pacman.git/commit/?id=0272fca993718460bf7ecb7fdc3ca7dad1c7e6cd). Someone even wanted `--skipbtrfshack` in [makepkg](https://bugs.archlinux.org/task/32228) at some point. We have also had some problems [sorting files](https://git.archlinux.org/pacman.git/commit/?id=b5191ea140386dd9b73e4509ffa9a6d347c1b5fa), and bsdtar `fflags` embedding information [unique to the system](https://git.archlinux.org/pacman.git/commit/?id=a897599fa54813ea2a225271eacd9fb6e1a6762e). 

But these quirks should have been solved, and we haven't found any other issues, yet. This means we got packages which can be consistently created, and we have recorded the build information. Now we need to recreate the build environment.

Arch Linux is a rolling release distribution. We build probably build and release between 20 and 100 packages every day. With no set release schedule, packages can get rotated in, and our, of a repository on the same day. When recoding the installed packages in the `BUILDINFO` files, this could in theory change several times a day. How do we reproduce a package released 2 weeks ago?

The main solution to this is to record all released packages, and that is what we do on https://archive.archlinux.org. A lot of the older packages is uploaded to [archive.org](https://archive.org/search.php?query=creator%3A%22Arch+Linux%22) and linked from the archive. This enables us to rebuild packages back in time, as long as the installed pacman was `>=5.2`.

Recreating the build environment for the package is the next mission. Arguably this step came before other parts of this blog post, but technically this probably makes sense ¯\\_(ツ)_/¯. 

We need to create a chroot and apply the needed environment changes. In devtools we utilize `systemd-nspawn` to create these chroot containers, and this in turn was the starting point for [archlinux-repro](https://github.com/archlinux/archlinux-repro). It provides two tools `repro` which takes a package file and attempts to reproduce the package given the package file, and `buildinfo` which is a helper tool to read the `BUILDINFO` file and download packages from the archive.

The goal of `archlinux-repro` is to be distribution agnostic. You should be able to reproduce Arch packages on any distribution and verify they are the same as the distributed package. We are also working on [`devtools` additions](https://github.com/eli-schwartz/devtools/blob/reproducible/makerepropkg.in) which enables us to tooling tighter integrated into our existing tools.

The input of `repro` is a package, however this alone is not enough to rebuild a package. We need the `PKGBUILD` file. Internally all Arch packages is stored into an svn repository. This is bit tedious to work with, but we provide a tool called `asp` which pulls a given package down from a git mirror of the svn repository.


And for the fun of it, the first package we have to try reproduce is obviously pacman:

```
$ repro pacman-5.2.1-1-x86_64.pkg.tar.xz 
:: Synchronizing package databases...
 core is up to date
 extra is up to date
 community is up to date
 multilib is up to date
:: Starting full system upgrade...
 there is nothing to do
==> Starting build...
  -> Create snapshot for build...
[...]
:: Running post-transaction hooks...
(1/2) Reloading system manager configuration...
  Skipped: Current root is not booted.
(2/2) Arming ConditionNeedsUpdate...
  -> Preparing packages
Hit cache for acl-2.2.53-1-x86_64.pkg.tar.xz
Hit cache for archlinux-keyring-20191018-1-any.pkg.tar.xz
Hit cache for argon2-20190702-1-x86_64.pkg.tar.xz
[...]
  -> Finished preparing packages
==> Installing packages
loading packages...
warning: acl-2.2.53-1 is up to date -- reinstalling
warning: archlinux-keyring-20191018-1 is up to date -- reinstalling
warning: argon2-20190702-1 is up to date -- reinstalling
[...]
==> Making package: pacman 5.2.1-1 (Fri Nov  8 15:35:29 2019)
[...]
==> Finished making: pacman 5.2.1-1 (Fri Nov  8 15:36:35 2019)
==> Cleaning up...
  -> Delete snapshot for build...
==> Comparing hashes...
==> Package is reproducible!

$ sha256sum pacman-*.pkg.tar.xz build/pacman-*.pkg.tar.xz 
3a13aff27db6d671e9a816e5ed4c05cb76fe703d998e78121f43645a3f8f7bd3  pacman-5.2.1-1-x86_64.pkg.tar.xz
3a13aff27db6d671e9a816e5ed4c05cb76fe703d998e78121f43645a3f8f7bd3  build/pacman-5.2.1-1-x86_64.pkg.tar.xz
```

If the result would have wound up as unreproducible, the reproducible builds project has created diffoscope that enables you to look at the differences in multiple different format. It is used in the CI environment, so we can take a look at the [diffoscope output from glib2](https://tests.reproducible-builds.org/archlinux/core/glib2/glib2-2.62.2-1-x86_64.pkg.tar.xz.html).


### The future?

The end goal should be to make any artifacts produced by Arch Linux reproducible. We currently have [reproducible initramfs in mkinitcpio](https://github.com/archlinux/mkinitcpio/pull/1), and having this with the archiso releases would be beneficial as well. Having independent rebuilders verifying the distributed packages as they are published is also something we really want to accomplish. However we always need more hands to help!

If you are interested helping out in any of these efforts, please visit  \#archlinux-reproducible on Freenode, or \#reproducible-builds on oftc!


Thanks to $name1, $name2, $name for reviewing the draft!

