---
layout: post
title: Maintaining your containers without maintaining your containers
date: 2020-12-17
categories: [containers, cves, security, patching, cicd]
---

It happens all the time: I need a container that serves a certain purpose, I make and push it to a public repository. I come back a year later and it has 43 critical CVEs discovered and thousands of pulls. (???)

How can I prevent this without pouring hundreds of my personal hours into maintaining software that doesn't make me money? What if I have a release, how do I "backport" security patches to base images? All I need to do is run `docker build` in the same directory on my laptop again. Can I automate this?

Yes, indeed, it is called ~~Lothric~~ scheduled rebuilds of images. This is pretty easy to do with a rolling release distro like Alpine. While I wouldn't bet my SLA on this, I think it is a great fit for personal images.

Using Github Actions, I have a "release" action that gets invoked on a push, or the first of every month:

```yaml
name: release
on:
  schedule:
    - cron: 0 0 1 * *
  push:

jobs:

  build:
    name: Build
    runs-on: ubuntu-latest
    steps:
[...]
```
Translation: Run this action on a push to the repository, or on 00:00 of the first day of every month.

For branching and releases, I simply make a branch for a release worth maintaining, like I do in [this repository](https://github.com/David-Igou/motion-container).

Can this break? Yeah, it's possible. You can build your containers off something like particular Debian release for some more consistency (ie `debian:bullseye`) but for something personal and easy to buid, a rolling release `alpine:latest` is totally fine.

 Often, I find lots of software maintainers don't properly manage their image builds, but Alpine's repositories are fine and backport security patches. So I can maintain my own Alpine image and meet them half way.
