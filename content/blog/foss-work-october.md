---
title: "FOSS Activities in October 2020"
date: "2020-11-01T00:00:00+01:00"
tags:
  - foss
  - monthly
  - archlinux
  - english
---

I wanted to start writing these for myself as I have been reading quite a few
monthly resports from Chris Lamb and other Debian contributors. They make for
interesting content for readers curious about what distribution maintainers do
during a month, and motivation for myself as not everything one does is visible
work.

I'll try have some sort of structure with them, by starting off with the menial
tasks, and add the meeting notes and misc contributions at the bottom.

If you have any feedback feel free to email me on morten@linderud.pw!

{{< figure src="/img/arch.png" >}}

## Package Updates to [community] 
- `toolbox` update to `0.0.95-1`
- `influxdb` update to `1.8.3-1`
- `buildah` update to `1.16.4-1`, `1.16.5-1`
- `python-reportlab` update to `3.5.52-1`, `3.5.53-1`, `3.5.54-1`
  - `3.5.52-1`: Removed the `ChangeLog` which has been there for close to a decade with 0 modifications 😱
- `gopls` update to `0.5.1-1`, `0.5.2-1`
- `font-awesome` update to `5.15.1-1`
- `github-cli` update to `1.1.0-1`, `1.2.0-1`
- `jp2a` update to `1.1.0-1`
- `fzf` update to `0.23.0-1`, `0.23.1-1`, `0.24.0-1`, `0.24.0.1-1`, `0.24.1-1`
- `qmk` update to `0.0.36-1`
- `python-rope` update to `0.18.0-1`
- `python-sqlparse` update to `0.4.0-1`, `0.4.1-1`
- `fuse-overlay` update to `1.2.0-1`
- `mypy` update to `0.790-1`
- `nano-syntax-highlighting` update to `2020.10.10-1`
- `staticcheck` update to `2020.1.6-1`
- `python-xlib` update to `0.28-1`
- `python-sqlobject` update to `3.8.1-1`
- `qutebrowser` update to `1.14.0-1`
- `python-language-server` update to `0.35.1-1`
- `go` update to `2:1.15.3-1`
- `perl-moox-handlesvia` update to `0.001009-1`
- `v2ray` update to `4.31.1-1`
- `nvme-cli` update to `1.13-1`
- `i3-gaps` update to  `4.18.3-1`
- `python-milc` updated to `1.0.9-1`, `1.0.10-1`
- `lostfiles` update to `4.02-2`
- `rclone` update to `1.53.2-1`
- `helm` update to `3.4.0-1`
- `plocate` update to `1.0.5-2`, `1.0.6-1`, `1.0.7-1`

## Package additions to [community]
- [`python-milc`](https://www.archlinux.org/packages/community/any/python-milc/)
  - added as new dependency for `qmk` 
  - https://milc.clueboard.co/#/
- [`plocate`](https://www.archlinux.org/packages/community/x86_64/plocate/)
  - Cool improvement over mlocate - https://plocate.sesse.net/

### Potential new packages for `[community]`
- vgrep
- git-publish
- oomd
- psi-notify
- adblock stuff for qutebrowser
- b4
- kubernetes

## Bugfixes
- `neofetch`: merged a [PR](https://github.com/Foxboron/archlinux-pkgbuilds/pull/4) from [Max](https://github.com/platy11)
  - It doesn't need `pacman-contrib` to detect package count.
- `toolbox`: [FS#68066](https://bugs.archlinux.org/task/68066) fixed missing flatpak dependency
- `qmk`: [FS#68212](https://bugs.archlinux.org/task/68212) Fixed missing runtime dependencies in qmk.
- `go`: fix colors in error
  - https://github.com/golang/go/commit/29634436fd741a7c685bf8f242b6fd62f093d1ad
  - `keybase` does weird stuff and was adding ASCII color codes to the output.
- `python-milc`: [FS#68380](https://bugs.archlinux.org/task/68380) conflicting test files.
  - Also created a [todo](https://www.archlinux.org/todo/python-modules-including-site-packagestests/) with packages that also contains the test directory.
- `fzf`
  - [FS#68266](https://bugs.archlinux.org/task/68266) added vim plugin docs
  - [FS#68448](https://bugs.archlinux.org/task/68448) failure when built without version
- `plocate`
  - `1.0.5-2`: Forgot `tmpfiles.d` when adding the package. So the database was never
    initialized properly.


## Security Team
The security team released 11 advisories this month.


## Other things....
- Announced the current plan of the Git migration and POC.
  https://lists.archlinux.org/pipermail/arch-dev-public/2020-October/030162.html
- Also did a quick hack to support `debuginfod` packages on the POC.
  https://paste.xinu.at/wxHUNspW2Y0fYOmmGZg/
- [Todo List: Remove usage of makepkg subroutines from PKGBUILDs](https://www.archlinux.org/todo/remove-usage-of-makepkg-subroutines-from-pkgbuilds/)
  - Probably marked off ~100 packages on the subroutine todo list.
- [Todo List: Python modules including site-packages/tests/](https://www.archlinux.org/todo/python-modules-including-site-packagestests/)
  - Created todo for conflicting test directories in packages.


## Arch Linux Conf 2020

Participating pulling of Arch Conf 2020, which was a community focused
conference. The last time we had something similar was back in 2010, last years
conference was mostly an internal workshop with Arch contributor, thus it was
quite exciting to organize something for the larger community.

I'll be writing a bit more about how this all went down on the production and
streaming site. Currently we are expecting the videoes to be up through the
first week in November.

## Reproducible Builds

{{< figure src="/img/reprro.jpeg" >}}

The reproducible builds has started their IRC meetings again. It was briefly
just a discussion how we should proceed with the meeting and what we should
focus on going forward. The start of these meetings where largely prompted by
COVID as we can't have a yearly summit like previous years.

Some form of meeting cadence is a good way to hash out details and draw new
contributors curious about the project. The quick recap of this meeting was to
establish some cadence of the meeting, how we should decide on meeting times and
potential topics for future meetings. We also had a quick status update by the
different projects.

[Summary of the meeting.](https://lists.reproducible-builds.org/pipermail/rb-general/2020-October/002059.html)


### Bergen Linux User Group Presentation

After my ArchConf talk Solskogen from the local LUG emailed me and asked if I
could hold a talk. BLUG usually have a talk the last thursday each month, but
because of COVID this has been hard to pull off.

The presentation was held in Norwegian and is more a general overview of the
reproducible builds project and the current progress. It was the first online
talk held by the local BLUG and I think it went quite well!

https://youtu.be/Tzc8arUBiRM?t=765


## Open-Source Security Foundation

### Third Meeting

{{< figure src="/img/openssf.png" >}}

Attended the [third
meeting](https://github.com/ossf/wg-vulnerability-disclosures/issues/51) at the [OpenSSF](https://openssf.org/) [Vulnerability Disclosure Working
Group](https://github.com/ossf/wg-vulnerability-disclosures) where we discussed
mainly 3 topics.

The first was the current standards in the space. This being the OASIS CSAF 2.0
standard, which is a JSON schema for declaring vulnerability management, along
with current automation work going on over at MITRE. This was presented by
Martin Prpic from Red Hat Product Security.

The second topic was whether or not it would be interesting to have a
presentation about the [VINCE
platform](https://www.sei.cmu.edu/news-events/news/article.cfm?assetid=641759) would be interesting for us. The platform
is suppose to be open-sourced soon and it might be interesting to the goals of
our working group. It was quickly declared interesting and hopefully there is a
presentation on a future meeting.

The third topic of the meeting was the goals of the working group as a whole. So
far we have had presentation of the workflows of different vulnerability
disclosure groups; Ruby, Arch and RedHat. But it's however important to
understand the direction of the group.

After some discussion it was declared the goals are:

1) Identifying vulnerability disclosure pain points for OSS maintainers, consumers, and reporter/finders and take steps to address them through techniques like automation and standardized data formats.

2) Documenting and promoting reasonable vulnerability disclosure and coordination practices within the OSS ecosystem for component maintainers and community members by providing documented standards and educational materials.

3) Facilitate the development and adoption of standards-based OSS Vulnerability information that uses existing industry formats. and allows OSS projects of all sizes to be able to report, share, and learn about vulnerabilities within OSS components.

Which I think are great goals of the working-group and hopefully not too broad
for it to be achievable. The details can be found in [PR#52](https://github.com/ossf/wg-vulnerability-disclosures/pull/52).

[Meeting notes.](https://github.com/ossf/wg-vulnerability-disclosures/blob/main/docs/meeting-notes/2020-10-05.md)


### Fourth Meeting

This meeting was [recorded](https://www.youtube.com/watch?v=aFFyQIoiis8).

The fourth meeting about some agreement how to proceed with meeting times and if
we should consider changing the meeting cadence. However nothing has been
explicitly decided at this moment.

Next topic was who are we really trying to solve pain-points for. The main
stakeholders defined so far has been "Maintainers", "Consumers" and "Security
Researchers".

Maintainers are the upstream project maintainers that might receive security
reports about their software. The main issue is "what do you do now". Some might
be interested handling this properly, but the current documentation is spread
across several resources and confusing at best. How do you handle an embargo,
who do you contact? How does downstream distribution handle security issues?

Security Researchers is in this case anyone looking and finding security issues.
They should, in many cases, be interested handling the disclosure of this
vulnerability properly. But again, how to contact the upstream, distributions
and how to relay information is spread and usually not an easy thing.

Consumers on the other hand is downstream distributions, or companies, that
consume information about security vulnerabilities. How to consume the
information, how to correlate the different product IDs to their provided
software isn't necessarily easy.

So how do we improve all of these things? The main thought so far is to try our
best gather the spread documentation and ensure there is a place to find it.
Giving some form of recommendation along with it. This can help people that are
interested in learning more get the relevant information instead of having to
rediscover everything.

[Meeting notes.](https://github.com/ossf/wg-vulnerability-disclosures/blob/main/docs/meeting-notes/2020-10-26.md)

The vuln-disclosure group also has a mailing list for us to use. I encourage
anyone reading this and are interested in this Working Group to subscribe.

https://lists.openssf.org/g/openssf-wg-vul-disclosures


Cheers!
