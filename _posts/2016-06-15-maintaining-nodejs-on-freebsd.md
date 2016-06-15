---
layout: post
title: Maintaining node.js on FreeBSD
---

After a series of PRs with patches to keep the FreeBSD node.js ports up-to-date with upstream, I became the maintainer for all the `www/node*` FreeBSD ports on 2016-04-11. I was humbled and honored to be offered the chance, and I accepted without hesitation.

## My first FreeBSD port maintainership

This is my first real port maintainership as well, if we ignore the `net/py-s3transfer` port I added as a dependency for `devel/awscli`.

The upstream node.js project does a good job of keeping FreeBSD builds working, and they even run the test suite regularly on FreeBSD as well. This makes my job as maintainer really easy. When a new version of node.js is released, usually all I need to do is bump the version in the `Makefile` and run `make makesum` before generating the patch for a PR.

I do quite a bit of testing of the `www/node*` ports myself, and I thought it might be interesting to describe how I do it.

## AWS EC2

To be able to run builds, I need a build machine. I could have used my NAS at home, but I opted for a `t2.medium` AWS EC2 instance instead. It's running 10.3-RELEASE, but I've been testing builds on every supported 9.x and 10.x release.

## git

I have been using git for a long time, and it is the tool I am most comfortable with. I have a clone of the [ports mirror from Github](https://github.com/freebsd/freebsd-ports) cloned in `/usr/ports`, and I use several branches to keep track of the changes that I do. This makes it easy for me push my changes to [my own Github fork](https://github.com/bradleythughes/freebsd-ports) and generate patches from there when creating and updating PRs.

## poudriere

Building patches with various combinations of options is best done with [poudriere](https://github.com/freebsd/poudriere). I've setup jails for 9.3-RELEASE, 10.1-RELEASE, 10.2-RELEASE, and 10.3-RELEASE, for both i386 and amd64. I've set it up to use the `/usr/ports` directory for all jails, so I can make modifications there and quickly run `poudriere bulk` or `poudriere testport` on my changes.

## Ready, set, go!

I feel like I've got a pretty good setup. I've been able to do some bigger changes to the port, beyond the normal version bumps. I hope to be able to consolidate all of the ports to reduce duplication and limit the number of identical deltas in PR patches. Beyond that, I'm really not sure what I'll do with it. Only time will tell.

:)
