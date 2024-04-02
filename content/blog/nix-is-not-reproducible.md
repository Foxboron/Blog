---
title: "NixOS is not reproducible"
date: "2024-04-02T19:21:08+02:00"
tags:
- english
- reproduciblebuilds
- linux
---

Okay, sorry for the clickbait.

NixOS is not reproducible according to the Reproducible Builds definition.

I keep reading people making this claim repeatedly on orange-site, even
LWN.net made a similar claim when writing about Nix and Guix earlier this
week.[^lwn] Along with their recently launched [wiki](https://wiki.nixos.org/wiki/Overview_of_the_NixOS_Linux_distribution).

So, what is the Reproducible Builds definition?[^definition]

> **When is a build reproducible?**
> 
> A build is reproducible if given the same source code, build environment and
> build instructions, any party can recreate bit-by-bit identical copies of all
> specified artifacts.

Neither Nix or NixOS gives you these guarantees.

When I point this out to people they generally seem surprised to learn this, and
I'm not completely sure where this originated from.

I suspect some people think this holds true because Nix heavily lives on hashes
from the derivations files to store the files under `/nix`. An example would be
`/nix/store/bnjps68g8ax6abzvys2xpx12imrx8949-binutils-2.31.1`.

Here the package is `binutils-2.31.1` and it is prefixed with a hash that is
`bnjps68g8ax6abzvys2xpx12imrx8949`. For any casual user it's completely
reasonable to believe this hash represents the builds artifacts. But in reality
the hash is based off on the derivation file. If you are unfamiliar with nix you
can imagine that you would install Rust packages under a path that includes the
sha256 of `Cargo.lock` file.

I also suspect it is because nixos.org had "Reproducible Builds" as the first
words of their homepage for years.[^3]

And I get it? The word "reproducible builds" encapsulates the intent of what
NixOS is trying to achieve, while also being an existing buzzword and quite
catchy. Compared to "deterministic builds" which doesn't exactly roll off the
tongue. I also think parts of this is when people say "reproducible" they
sometimes mean "reproducible behaviour", as in the system behaving the same, or
"Reproducible Builds". And this confusion leads to a great deal of the
misconception.

I should clarify that I'm not saying NixOS doesn't *have* some reproducible
packages. I'm trying to point out that proving a small set of packages is
reproducible doesn't make the entire project reproducible. We still do not know
how to make that work, and any grandeur claims about having solved this problem
should be met with a lot of scepticism.

### Definitions and semantics, yay

To try and expand on the concepts of *words* in this realm I though I'd defer to
the "Building Secure and Reliable Systems" book from Google.[^book] Where they try to
define some terminology in this area.[^builds]

* Hermetic builds
* Reproducible builds
* Verifiable builds

Hermetic builds means that we are building in isolated environments, with all
the sources specified up front and some cryptographic pinning to ensure this is
true. I believe most Linux distributions and language package managers
accomplishes this to some degree and it is somewhat of an solved problem.

Reproducible builds, is well, Reproducible Builds. It means that running the
same build ensures the same build artifacts are bit-for-bit identical. This is a
work in progress for everyone involved and there are so many issues we are
trying to figure out.

Verifiable builds implies there is a trusted and validated *path* from binary to
the source. This doesn't mean the build is reproducible, but it means that we
should know what the build contains a high degree of certainty. This is what I
believe a lot of the recent years of "Supply Chain Security"-focus has given us
with build provenance, attestation and a myriad of SBOM standards. I think NixOS
can make probably make some claims about having "Verifiable Builds" in their
distro and they could probably lean a bit harder on this part.

## Addendum

I think it is important to point out I take this entire thing a bit personal for
several reasons. 

I have heavily invested my free-time on this topic since 2017, and met some
of the accomplishments we have had with "Doesn't NixOS solve this?" for just as
long... and I thought it would be of peoples interest to clarify?

Out of the last three times I went to a F/OSS event with my Reproducible Builds
t-shirt the two reactions I did get was from enthusiastic Nix users that wanted
to talk about... nix? And I'm not really into Nix after spending almost a decade
working on Arch?

But it seems to me that the misconception is so common these days that trying to
clarify is generally useful, and I have edited this on Wikipedia two times
because the claims where just wrong.[^6][^7]

But to try and end on a positive note:

#### What does all of this mean?

*There* *is* *so* *much* *work* *to* *do*. Come join the Reproducible Builds
effort in your Linux distro. There is constant work in all major Linux distros
that needs help. It's a great way to do cross-distro collaborative work and you
can learn the eldritch horror that is "build systems". I spent 2-3 months
tracking down a `gcc` but in `-flto` and I assure you I thought it was fun
at least for an hour or two.

Fedora has recently kicked off their efforts again[^fedora].

In Arch we want to try and reach the 90% mark again for our package[^arch].

NixOS has a great dashboard for their reproducible builds issues[^nixos]

We are also sufficiently old-school there are active mailing lists for this as
well. 

https://lists.reproducible-builds.org/listinfo/

Hopefully this clears up some of the misconception people have on this topic,
and maybe it inspired you to contribute to the collective effort to solve this :)



[^definition]: https://reproducible-builds.org/docs/definition/
[^lwn]: https://lwn.net/Articles/962788/
[^3]: https://github.com/NixOS/nixos-homepage/pull/1077
[^book]: https://google.github.io/building-secure-and-reliable-systems/
[^builds]: https://google.github.io/building-secure-and-reliable-systems/raw/ch14.html#hermeticcomma_reproduciblecomma_or_veri
[^6]: https://en.wikipedia.org/w/index.php?title=Comparison_of_Linux_distributions&diff=prev&oldid=1151180812
[^7]: https://en.wikipedia.org/w/index.php?title=Reproducible_builds&diff=prev&oldid=1154959180
[^arch]: https://reproducible.archlinux.org/
[^fedora]: https://discussion.fedoraproject.org/t/report-from-the-reproducible-builds-hackfest-during-flock-2023/87469
[^nixos]: https://github.com/orgs/NixOS/projects/30
