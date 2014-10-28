---
layout: post
title: FreeBSD, Munin, and lighttpd
---

# FreeBSD, Munin, and lighttpd

I have a NAS at home (lovingly called `nasty.local`). It runs [FreeBSD](https://www.freebsd.org/) 10.1-RC3, with [Munin](http://munin-monitoring.org/) and [lighttpd](http://www.lighttpd.net/) installed via [ports](https://www .freebsd.org/ports/). It took me some time to figure out how to get this combination to work the way I wanted, mostly due to inexperience with Munin and lighttpd. I wanted to document what I had done in the hopes that other people would find the information useful.

## Configuring munin-node

The [documentation](https://munin.readthedocs.org/) for installing and setting up `munin-node` is the first place I looked. I installed the default set of plugins with `munin-node-configure --shell | sh -x`. Over the past few months, I have removed a few and added some others, including some of the ZFS plugins from the [munin-contrib](https://github.com/munin-monitoring/contrib) repository. I ended up using these plugins:

```
$ ls /usr/local/etc/munin/plugins
cpu			netstat			smart_ada5
df			open_files		swap
if_em0			processes		systat
iostat			smart_ada0		uptime
load			smart_ada1		users
memory			smart_ada2		zfs_stats_efficiency
munin_stats		smart_ada3		zfs_stats_utilization
munin_update		smart_ada4		zfs_usage_Z
```

The default configuration mostly works, with one exception. When I started using the `smart_` plugin, it could not find the `smartctl` command because the default config uses the wrong environment variable (`env .smartctl` instead of `env.smartpath`). The default config also does not give `smartctl` enough permissions to access the devices, so my configuration looks like this:

```
$ cat /usr/local/etc/munin/plugin-conf.d/smart.conf
[smart_*]
user root
env.smartpath /usr/local/sbin/smartctl
```

Now it's just a matter of making sure that the `munin-node` service is running:

```
$ sudo sysrc munin_node_enable=YES
$ sudo service munin-node start
```

## Configuring munin-master

As with `munin-node`, the default configuration for `munin-master` mostly works. I wanted to make Munin generate the HTML and graphs on demand with the `munin-cgi-html` and `munin-cgi-graph` CGI scripts, instead of letting the cron job do it. I had hoped it was be as simple to just change the `graph_strategy` and `html_strategy`:

```
$ cat /usr/local/etc/munin/munin-conf.d/cgi.conf
graph_strategy cgi
html_strategy cgi
```

... which it was, with one caveat (see below).

## Configuring lighttpd

Munin puts its CGI and generated HTML in `/usr/local/www/cgi-bin` and `/usr/local/www/munin`, respectively. Note, however, that my `/usr/local/www/munin` directory is empty (see above). The default configuration of lighttpd uses `/usr/local/www/data` as the document-root. This directory is also empty. It's up to me to put something useful there. I haven't done this (yet!), so I get a 404 error for anything I try to browse on my NAS.

I wanted to access the Munin graphs using `nasty.local/munin/`. I had found [an example configuration](http://munin-monitoring.org/wiki/MuninConfigurationMasterCGI) that I used as a starting point, and the result turned out to be not too bad:

```
$ cat /usr/local/etc/lighttpd/conf.d/munin.conf
server.modules += ( "mod_alias", "mod_rewrite" )
include "conf.d/cgi.conf"

alias.url += ( "/munin/static" => "/usr/local/etc/munin/static" )
alias.url += ( "/munin-cgi" => "/usr/local/www/cgi-bin" )
$HTTP["url"] =~ "^/munin-cgi" {
    cgi.assign = ( "" => "" )
}

url.rewrite-repeat += (
    "^/munin/((?!static/).*\.png)$" => "/munin-cgi/munin-cgi-graph/$1",
    "^/munin/((?!static/).*\.html)*$" => "/munin-cgi/munin-cgi-html/$1"
)
```

As you can see, I'm using plain, old CGI, since this "Just Works (tm)" with the default lighttpd install. The aliases tell lighttpd where to find static content and the CGI scripts, the rewrite rules which CGI script to call for a given URL. I had to make sure that the *.html rule matched both `/munin/` and `/munin/*.html` to ensure that the index was correctly generated for `nasty.local/munin/`.

The final step was to tell lighttpd to load my conf file, which meant modifying the default configuration slightly:

```diff
$ diff -u /usr/local/etc/lighttpd/lighttpd.conf.sample /usr/local/etc/lighttpd/lighttpd.conf
--- lighttpd.conf.sample	2014-09-14 23:58:54.262130000 +0200
+++ lighttpd.conf	2014-10-27 21:42:10.252855767 +0100
@@ -457,3 +457,7 @@

 # IPv4 listening socket
 $SERVER["socket"] == "0.0.0.0:80" { }
+
+# local modifications
+server.breakagelog = "/var/log/lighttpd/breakage.log"
+include "conf.d/munin.conf"
```

Time to enable lighttpd and start it to give `nasty.local/munin/` a try in my browser:

```
$ sudo sysrc lighttpd_enable=YES
$ sudo service lighttpd start
```

## Does it work?

No. It should, but it doesn't. Here's the caveat I mentioned above: the Munin CGI scripts are run as the `www` user, but want to log to the `/var/log/munin/` directory, which is owned by the `munin` user. I didn't think about this at first. I was only after I examined the lighttpd `breakage.log` that I figured it out:

```
$ sudo grep Permission\ denied /var/log/lighttpd/breakage.log | head -n 1
... munin-cgi-graph: Can't open /var/log/munin/munin-cgi-graph.log (Permission denied) at ...
```

Thankfully, `newsyslog(8)` on FreeBSD is easy to configure to both create the log files with correct permissions and rotate them like all other system logs:

```
$ cat /usr/local/etc/newsyslog.conf.d/munin-cgi.conf
/var/log/munin/munin-cgi-graph.log www:www 644 7   *    @T00  CNX
/var/log/munin/munin-cgi-html.log www:www  644 7   *    @T00  CNX
```

The newsyslog rules are applied once an hour via `cron(8)`, but it's easy to apply them right away if you're impatient (like I am):

```
$ sudo newsyslog
```

## Surely it must work now?

Yes. Yes, it does. I still haven't put anything useful in `/usr/local/www/data`, so browsing to `nasty.local/` returns a 404, but `nasty.local/munin/` works, as well as all the URLs under it.

## Rationale

If someone were to ask me why I put everything together the way I did, I'm not sure I could give a reasonable answer. I'm not sure why. I could have certainly used the `cron` strategies when configuring `munin-master`, and found a way to link/copy/move/change the output from `/usr/local/www/munin` to `/usr/local/www/data/munin`. There are certainly other ways to get the same result that I wanted. The configuration feels minimal, and I could easily replicate this on other machines if I wanted, both privately or for work.

In the end, the result works. And that's what really matters.
