+++
title = "On LFD103 Chapter 2"
date = "2023-10-23T19:12:26+02:00"
author = "nipsn"
authorTwitter = "" #do not include @
cover = ""
tags = ["LFD103", "kernel", "linux"]
keywords = ["LFD103", "kernel", "linux"]
description = "Notes and thoughts on LFD103 Chapter 2"
showFullContent = false
readingTime = true
hideComments = true
color = "" #color from the theme settings
+++

# Chapter 2

## Thoughts

This chapter is quite simple but it serves as an introduction to how the kernel is being worked on. As an outsider, it honestly seems farfetched that this whole system works without failure. Since all code eventually channels through Linus itself, I wonder what the protocol is when he isn't there anymore, like what happened to Vim recently. It will probably be ok, but I still think that this is a lot of responsability for a single individual. I'm most likely missing the bigger picture though.

## Notes

### Release cycle

There is usually a new release of the kernel roughly every 2 months and several stable and extended stable releases once a week.Releases are time-based, not feature-based, so things are shipped when they are ready to be shipped. There is no set date.

The release cycle looks something like this:

 - Linus releases a new kernel and opens a 2-week merge window. During this time, he pulls code from subsystem maintainers, which send signed git pull requests. At the end of this period Linus releases the first release candidate (`rc1`) and thus the merge window is closed.

 - From then on, there is another 6 to 7 week period of bug fixing. Each of these weeks see the release of a `rc`. There is sometimes an `rc8` if the quality is not up to par yet.

 - Then, there is a 3 week long _quiet period_, which starts a week before the release and continues through the 2 week merge window.

See [this image](https://d36ai2hkxl16us.cloudfront.net/course-uploads/e0df7fbf-a057-42af-8a1f-590912be5460/03bno4kvu510-LinuxDevelopmentCycle.jpg) for more info.

### Subsystem maintainers

Each subsystem has its maintainer(s). For each of these there is a `MAINTAINERS` file describing the maintainers. Almost every subsystem has a mailing list.

These maintainers receive contributions and patches from developers and review them. If correct, it later gets pulled to the mainline kernel.

### Quick bits

A patch is a small incremental change made to the kernel.

The quiet period overlaps with the merge window.

The quiet period is 3 weeks long.

There are usually 7 or 8 RCs (Release Candidates) before a mainline kernel release.

The `MAINTAINERS` file contains information on who maintains the subsystem.

Several kernel git repos are hosted on kernel.org .

Patch emails are not sent to Linus.

New features don't go into stable releases.

