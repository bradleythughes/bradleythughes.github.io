---
layout: post
title: Moving Blackbox to a New Home
---

Recent press about SourceForge prompted me to checkout an old... THE old project of mine, [Blackbox](https://github.com/bradleythughes/blackbox). It's been on SourceForge for a very long time, and is still living in CVS. Being a long-time git user, it felt like the right time to migrate away from SourceForge and CVS to Github.

I didn't expect it to be this easy.

## Step 1, rsync CVSROOT

The [documentation](http://sourceforge.net/p/forge/documentation/CVS/) about backing up your CVS data includes instructions on how to use [rsync](https://rsync.samba.org/) to get all your data. All I needed to do was copy-paste-edit the command:

```
$ mkdir blackboxwm.cvs.sf.net
$ rsync -av rsync://blackboxwm.cvs.sourceforge.net/cvsroot/blackboxwm/* blackboxwm.cvs.sf.net
```

## Step 2, git cvsimport

Once I had a copy of the CVS data, I converted the history into a new Git repo using the [git cvsimport](http://git-scm.com/docs/git-cvsimport) command:

```
$ git cvsimport -v -d `pwd`/blackboxwm.cvs.sourceforge.net -k -u -C blackbox blackbox
```

After a few minutes (yes, MINUTES), all the history with branches and tags was imported. I deleted the `origin` branch, which was an identical copy of `master`, as well as a branch named `#CVSPS.NO.BRANCH` after the import was completed.

## Step 3, git push

I created a new [Github repository for Blackbox](https://github.com/bradleythughes/blackbox). Now to just push:

```
$ git remote add origin git@github.com:bradleythughes/blackbox.git
$ git push -u origin master
$ git push origin --all
$ git push origin --tags
```

## Done

This was so easy, and I'm so pleased and proud to be able to give Blackbox a new home, even if it is only a trophy case.

I am unsure what I want to do with the [blackboxwm SourceForge project](http://sourceforge.net/projects/blackboxwm/) and [website](http://blackboxwm.sourceforge.net/). I don't even know if I have the password or access to the email I used for my account. Maybe it will just linger there forever?
