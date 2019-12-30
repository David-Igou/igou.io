---
layout: post
title: "Building thie site (again)"
date: 2018-10-25 17:32:28 -0400
comments: true
categories:
---
My previous setup wasn't very agile. It was fun scrapping some shell scripts together but it wasn't very agile or flexible.

This new incarnation is generated using [Octopress](http://octopress.org/). It's a pretty simple deployment and not that much different from my previous setup. I didn't need to change much from my original deployment script. I took a pretty basic theme and ripped most of it out to emulate the old site as much as I could. Currently this falls apart on mobile browsers.

I literally just kick this off with a shell script to a tiny ec2 instance.

<!-- more -->

``` bash deploy.sh
#!/bin/bash
pushd octopress && bundle exec rake generate &&
popd && ansible-playbook -i inventory httpd.yml
```

I made this change because I did so many projects this summer and the ridiculous old setup prevented me from writing about them.
