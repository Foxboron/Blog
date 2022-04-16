---
title: "Monitoring the kernel.org Transparency Log for a year"
date: "2022-03-27T17:57:42+0200"
tags:
    - english
    - linux
    - transparency log
---

Lets prefix this with: I really love Transparency Logs!

It's a fairly simple concept: If you hash elements together in a binary tree,
you can validate and verify if elements are present on a tree by hashing a
couple of elements. This is what is commonly known as a [Merkle tree](https://en.wikipedia.org/wiki/Merkle_tree).

I forget the math, but if you have a tree with a million items, you would only
really need less than 10 hashes (I think) to figure out what the hash of the top
node would be. This allows you to easily audit and verify the tree is internally
consistent. If you compare this to something like git, which people often
mistake for a Merkle tree, you would need to effectively replay a million
commits to figure out if the last commit hash is actually correct or not.

In day-to-day life this is used in Certificate Transparency Logs to have an
audit trace of who published which TLS certificate. You can read [RFC6962](https://datatracker.ietf.org/doc/html/rfc6962) for more details on this.

In 2019 I wrote my [master thesis](https://bora.uib.no/bora-xmlui/handle/1956/20411) around the same
concepts but in the context of [Reproducible Builds](https://reproducible-builds.org/). If we have several parties that
reproduce published binaries and announce them publicly, how do we know if they
won't retract this in the future? Transparency Logs, obviously 😊. It also had
some ideas on how to implement this inside Linux package managers, `apt` in this
case. More recently kpcryd has implemented the some of the same ideas with
[`pacman-bintrans`](https://github.com/kpcyrd/pacman-bintrans).

Generally speaking, transparency logs in the context of the software
supply-chain security is underutilized. This is why I was super excited about
kernel.org announcing their own transparency log!

## Git push certificates

Konstantin Ryabitsev [announced the kernel.org transparency log](https://people.kernel.org/monsieuricon/introducing-the-kernel-org-git-transparency-log) back in November 2020.  What many people probably don't realize is that git doesn't only sign commits,
it can also sign [push attestations](https://people.kernel.org/monsieuricon/signed-git-pushes). This log records who pushes what to which repositories, along with a signed attestation. 
If you record this on a transparency log then you get a tamper-evident record of all the
commits that was pushed, and by who.

Why is this useful?

* [Hackers backdoor PHP source code after breaching internal git server](https://arstechnica.com/gadgets/2021/03/hackers-backdoor-php-source-code-after-breaching-internal-git-server/) (2021)
* [Github Gentoo organization hacked](https://www.gentoo.org/news/2018/06/28/Github-gentoo-org-hacked.html) (2018)
* [Kernel.org Linux repository rooted in hack attack](https://www.theregister.com/2011/08/31/linux_kernel_security_breach/) (2011)


Logging all push actions doesn't prevent any of these issues on their own. But
they do provide tamper-evident logs in the case of a forge gets compromised.
If the pipelines publishing the artifacts do strict requirement checks, like all
commits are pushed and signed as an example, then you could prevent some of the issues above.

However, Transparency Logs are cool, but they are fairly useless without
anything looking at them and verify that the entries are consistent. Monitors
essentially track the log and validate that any recent additions to the logs
can't be retroactively removed. Thus preventing the tampering.

Last year I wrote a monitor for the kernel.org log which I have been running on
https://tlog.linderud.dev. The code can be found on
[Github](https://github.com/Foxboron/kernel.org-git-verifier).

It uses the
[pgpkeys](https://git.kernel.org/pub/scm/infra/transparency-logs/gitolite/git/1.git/tree/m?id=050341bbff698070996979b840eac013413cc242) repository
which hosts the GnuPG keys of kernel.org maintainers, and the last shard of
the [transparency log](https://git.kernel.org/pub/scm/infra/transparency-logs/gitolite/git/1.git/) which is when the signing started. 

An example log entry: https://git.kernel.org/pub/scm/infra/transparency-logs/gitolite/git/1.git/tree/m?id=050341bbff698070996979b840eac013413cc242

What the monitor does is parsing the email, validate the signature and store the result. That is what the website is currently showing. This is somewhat useful, but mostly functions as a naughty list of all the kernel devs that hasn't started signing their git
pushes yet[[^1]].


Clearly we can do more interesting stuff with the data!

## Verifying torvalds/linux.git

I included some code that stores all the witnessed commits in the logs while we
parse the emails. This helps us work more as a verifier and check if all commits
are present. If we traverse the commits between two kernel releases we can check
if we have seen any of the commits previously on the log. We also store their
signature validity to make some assertion if they where signed or not.

As we are not too strict we check for signature validity across all the
different repositories on kernel.org. If Greg Kroah-Hartman pushes 3 commits to
his kernel clone, and Linus pulls these changes. The verifier is going to mark
all the 3 commits as valid signatures even if Linus never signed his push.

I have included some results below. The first one is between the last two kernel
releases taken from [torvalds/linux](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/).
{{< highlight shell >}}
$ verifier check v5.16 v5.17
Out of 14199 commits...
14199 was found on the tlog between v5.16 and v5.17
14199 commits where done over 2956 pushes
2239 commits are covered by a valid signature
{{< /highlight >}}

(At this point I should note that I don't fully trust my own code. Don't take
this as proof that nothing malicious has happened to the repositories. But
rather keep it around as a potential data point.)

The first thing we should pay attention to is that all the commits from the
release was found on the transparency log copy we have. We can also see that
there is not a whole lot of signed pushes being done, however we do have commits
covered by a valid signature. If we didn't find commits on the log it would be a
good canary that something weird was happening, but not necessarily malicious.

I have attached rest of the results from the kernel below. Please note that log
was deployed in the v5.9 release cycle, so anything before that is not present
on the signed log.

{{< highlight shell >}}
$ verifier check v5.15 v5.16
Out of 15384 commits...
15384 was found on the tlog between v5.15 and v5.16
15384 commits where done over 2920 pushes
2642 commits are covered by a valid signature

$ verifier check v5.14 v5.15
Out of 13473 commits...
13473 was found on the tlog between v5.14 and v5.15
13473 commits where done over 2759 pushes
2235 commits are covered by a valid signature

$ verifier check v5.13 v5.14
Out of 15871 commits...
15871 was found on the tlog between v5.13 and v5.14
15871 commits where done over 2924 pushes
2815 commits are covered by a valid signature

$ verifier check v5.12 v5.13
Out of 17187 commits...
17187 was found on the tlog between v5.12 and v5.13
17187 commits where done over 3041 pushes
3068 commits are covered by a valid signature

$ verifier check v5.11 v5.12
Out of 14167 commits...
14167 was found on the tlog between v5.11 and v5.12
14167 commits where done over 2756 pushes
1795 commits are covered by a valid signature

$ verifier check v5.10 v5.11
Out of 15477 commits...
15007 was found on the tlog between v5.10 and v5.11
15477 commits where done over 2730 pushes
1687 commits are covered by a valid signature

$ verifier check v5.9 v5.10
Out of 17467 commits...
6305 was found on the tlog between v5.9 and v5.10
17467 commits where done over 863 pushes
225 commits are covered by a valid signature

$ verifier check v5.8 v5.8
Out of 1 commits...
0 was found on the tlog between v5.8 and v5.8
1 commits where done over 1 pushes
0 commits are covered by a valid signature
{{< /highlight >}}

Personally I think this is a pretty neat project. Monitors allows us to further
secure supply-chain as it makes us less reliant on centralized infrastructure.
I'm a little bit annoyed I have not been able to improve on it to such a degree
that one could trust it. But have to start somewhere :)

# Future improvements
Clearly we can do *more* and improve things.

I noted in the beginning that git is not a transparency log, sadly the
kernel.org transparency log is a git repository. It's not a true transparency
log and trying to prove the that a commit is present on the log needs one to
replay the entire log before comparing the state of the main log and the
monitor. This isn't ideal.

A big improvement would be to move this over to an actual transparency log.
[Sigstore](https://www.sigstore.dev/) attempts to be a general purpose transparency log for build artifacts. I did start working on a
[patch](https://www.sigstore.dev/) to include git-push certificates in Rekor, their transparency log implementation. But this has stalled because I lack the
time and I think it would need a bit of code on the remote end to function properly.

If you are into OpenSSF projects, the SLSA project lists several requirements to
protect a supply chain. A Transparency Log for git-push certificates neatly fits
into the B Threat, ["Compromise source control platform"](http://slsa.dev/spec/v0.1/threats). However mitigations like
transparency logs are not widely cited as being possible mitigation to this
threat. One can speculate why this is the case, but I think the lack of support in
Git forges is the main issue.

Gitlab supports [server hooks](https://docs.gitlab.com/ee/administration/server_hooks.html) and I think you could implement a git-push monitor with a bit of manual maintenance. Github does not support this, and I think that is something Github should improve on.
Generally these sort of server-side configurations are not very
accessible but very much needed. I'd love to have my own 

Another improvement on the monitor would be to look more into the code
the Trillian team at Google has been doing. Currently they have written a few
concepts for a standardized [Witness
protocol](https://github.com/google/trillian-examples/tree/master/witness) for
different transparency logs, and I think it would be neat to provide the same
interface for the kernel.org log. This would make it easier to have standardized
clients across different logs to validate entries.


# Closing
This project was mostly thrown together so I had some more content for my BSides
Oslo [talk](https://www.youtube.com/watch?v=ptoDdcJ_B_g) last year and I hope people find it interesting.

It would be great to keep better statistics around the number of signatures in
the kernel, how many kernel devs sign their pushes, and the ratio over time.
However I'm a bit unsure if I should publish results of my monitor between
kernel releases considering the state of the project.

If the wider community see this as valuable I can probably try figure something
out, please let me know.


[^1]: Bonus points if you manage to guess which kernel developer has the most signed push certificates on the log. [Answer](https://pub.linderud.dev/answer.txt).
