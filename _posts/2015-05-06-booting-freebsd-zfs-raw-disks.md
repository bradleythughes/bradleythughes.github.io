---
layout: post
title: Booting FreeBSD from ZFS on Raw Disks
---

I recently converted my home NAS running FreeBSD and ZFS to using whole, raw disks instead of GPT labeled partitions. While trying to find out how to do it, I had problems finding documentation about the subject. Thankfully, I came across a post on the FreeBSD forums about [booting from ZFS on raw disks](https://forums.freebsd.org/threads/boot-from-zfs-root-on-raw-disks.50525/). Thanks to that thread, along with some documentation found in the `10-STABLE` branch, I had the information I needed to convert my pool to whole disks.

## But, Why?

At first, I became irrationally obsessed with the "wasted" space from GPT partition alignment and having six 2GB swap partitions (one on each disk in my RAID-Z2). At one point, I also came across some documentation that [recommends using whole disks](http://www.solarisinternals.com/wiki/index.php/ZFS_Best_Practices_Guide#Storage_Pools) for various reasons, citing enabling disk caching as the first. But as pointed out in the forum thread mentioned above, [this is old advice](https://lists.freebsd.org/pipermail/freebsd-questions/2013-January/248701.html).

When I setup my RAID-Z2, I used GPT labeled partitions with arbitrary numbering, `/dev/gpt/zfs#`. I noticed that FreeBSD labels disks by serial number in `/dev/diskid/` if it does not have a GPT. I wanted to replace these GPT labels with labels using disk serial numbers. Yes, I could have just relabeled the GPT partitions, but that is not what I set out to do, was it?

So in the end, I ended up doing it out of curiosity and for the experience.

## The Process

When I labeled the `gpt/zfs#` partitions, I used the same numbers as the `ada` devices. After destroying the GPT, an easy way to find the new label is using the `glabel status` command. This makes it quick and easy to find which labels to use in the replace step below. For example:

```
$ glabel status ada0
                Name  Status  Components
diskid/DISK-S1E33M3X     N/A  ada0
```

Replacing a disk in a pool is a well documented process. Mine is slightly different, but familiar nonetheless. Take the disk offline, destroy the GPT on it, and then replace the disk with the new `diskid/DISK-$SERIALNO` label, and wait for it to resilver. Repeat this process on each disk. Done.

Well, almost done...

## You Need To Update the Boot Code

Yes, I know.

This is also where the documentation was lacking. This is what was stopping me from converting to using whole disks in the firts place. Thanks to the forum post mentioned above, I found the `zfsboot(8)` man page in the `10-STABLE` branch, which states `dd(1)` is typically used to install the boot code:

```
dd if=/boot/zfsboot of=/dev/ada0 count=1
dd if=/boot/zfsboot of=/dev/ada0 iseek=1 oseek=1024
```

Repeat this process on each disk. Done.

## The Payoff

My pool now only uses whole disks, and I have disk labels showing the serial numbers of each disk.

```
$ zpool status
  pool: Z
 state: ONLINE
  scan: scrub repaired 0 in 3h52m with 0 errors on Thu Apr 30 07:01:50 2015
config:

	NAME                      STATE     READ WRITE CKSUM
	Z                         ONLINE       0     0     0
	  raidz2-0                ONLINE       0     0     0
	    diskid/DISK-S1E33M3X  ONLINE       0     0     0
	    diskid/DISK-S1E3362L  ONLINE       0     0     0
	    diskid/DISK-S1E33QDQ  ONLINE       0     0     0
	    diskid/DISK-S1E33M98  ONLINE       0     0     0
	    diskid/DISK-S1E33P14  ONLINE       0     0     0
	    diskid/DISK-S1E33M65  ONLINE       0     0     0

errors: No known data errors
```

The serial number for any disk with errors is clearly identifiable now. Should I ever need to pull a disk for replacement, I can look at the serial numbers on the physical disks themselves to find the correct one. I will have the serial number for the new disk in hand, making it easy to find the label for the replacement as well.

I am pleased.

## Remaining Debt

There is a downside, however. The `zfsboot(8)` man page states it perfectly in the BUGS section: "Installing zfsboot with dd(1) is a hack.  ZFS needs a command to properly install zfsboot onto a ZFS-controlled disk or partition." Should I ever upgrade the pool with new features, or replace a failing disk, I need to remember the two magic `dd(1)` incantations to reinstall the boot code.

I hope to find an easy way to remedy that soon...
