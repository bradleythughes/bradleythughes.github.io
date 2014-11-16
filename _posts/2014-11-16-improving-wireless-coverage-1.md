---
layout: post
title: Improving wireless coverage at home, part 1 - AP bridge on FreeBSD
---

I have an all-in-one wireless router, ethernet switch, cable modem, cable decoder, PVR, and IP telephony device from our cable provider, the [Get box II](http://www.get.no/produkter/tv/getboksene).

{% include block-image.html src="//www.get.no/produkter/tv/getboksene/_image/73282.png" %}

It works well, has both 5 GHz and 2.4 GHz radios, supports IPv4 port forwarding, and even has full IPv6 support. It looks good too. I like it.

## But?

Being an all-in-one device has its drawbacks, though. In particular, the Get box II lives in the entertainment center in one corner of the living room. It has to, since that's where the TV is. The living room is on the top floor of our 3-floor duplex. Getting a good wireless connection is not hard, as long as you are on the top floor or the rooms directly under the living room on the middle floor. The rest of the house is painful. Dining room is iffy, kitchen is mostly a no-go, and you can just forget about the basement.

## Get nasty!

I have [previously mentioned](/2014/10/28/freebsd-munin-lighttpd.html) my home NAS, `nasty.home`. The [motherboard](http://www.asrock.com/mb/amd/fm2a88x-itx+/) I bought for it has built-in wireless.

{% include block-image.html src="//www.asrock.com/mb/photo/FM2A88X-ITX+(m).jpg" %}

I wanted to try and set `nasty.home` up as an AP bridge without disrupting its main functions. The hope is that an additional AP can provide coverage for the painful areas of the house.

At this point in time, `nasty.home` is running [FreeBSD](https://www.freebsd.org) [10.1-RELEASE](https://www.freebsd.org/releases/10.1R/announce.html), which includes all the necessary drivers (`ath(4)`) and software (`hostapd(8)`) needed. Setting everything up turned out to be easier than I expected.

## Setting up the WLAN

First, I recommend reading the the FreeBSD handbook chapter on [wireless networking](https://www.freebsd.org/doc/en_US.ISO8859-1/books/handbook/network-wireless.html). I found everything I needed there.

After playing around with `ifconfig(8)` on the command line, I ended up putting the following into `/etc/rc.conf`:

```
wlans_ath0="wlan0"
create_args_wlan0="wlanmode hostap country NO"
ifconfig_wlan0="mode 11a"
```

This creates the `wlan0` interface but does not bring it up. I want to configure encryption first.

## Setting up WPA/WPA2

I use `hostapd(8)` to secure the WLAN with WPA/WAP2 Personal. The SSID and PSK are put into `/etc/hostapd.conf`:

```
interface=wlan0
debug=1
ctrl_interface=/var/run/hostapd
ctrl_interface_group=wheel
ssid=High
wpa=3
wpa_passphrase=REDACTED
wpa_key_mgmt=WPA-PSK
wpa_pairwise=CCMP TKIP
```

Be sure to enable `hostapd(8)` in `/etc/rc.conf` as well:

```
hostapd_enable="YES"
```

Once started, `hostapd(8)` will bring up the wlan0 device.

## Setting up the bridge

Again, I recommend reading the FreeBSD handbook chapter on [network bridging](https://www.freebsd.org/doc/en_US.ISO8859-1/books/handbook/network-bridging.html). The [man page](https://www.freebsd.org/cgi/man.cgi?query=bridge&apropos=0&sektion=4&manpath=FreeBSD+10.1-RELEASE&arch=default&format=html) for `bridge(4)` is also useful reading.

I configured the bridge by hand a few times before landing on the following in `/etc/rc.conf`:

```
cloned_interfaces="bridge0"
ifconfig_bridge0="addm wlan0 addm em0 SYNCDHCP"
ifconfig_bridge0_ipv6="inet6 accept_rtadv auto_linklocal"
```

Note that the `bridge0` interface is configured to do DHCP and IPv6 SLAAC instead of the wired `em0` device, as documented in the handbook.

### Potential snag

The first time I rebooted to test that everything worked automatically from boot, the bridge interface did not get an IP address. This is because I forget to tell the system to "up" the wired interface, `em0`. I didn't have to do this when tinkering with `ifconfig(8)`, since `em0` was already up at that point. The fix is easy, it's just one more line for `/etc/rc.conf`:

```
ifconfig_em0="up"
```

## Testing connectivity

Once everything is configured and running, I wanted to test to make sure that the AP on `nasty.home` actually worked. I should be able to just turn off the radio on the Get box II, since I am using the same SSID on both the Get box II and `nasty.home`. Once I had done that, I turned back to `nasty.home` to see if it had any associated clients (spoiler alert: it did).

```
$ ifconfig wlan0 list sta
ADDR               AID CHAN RATE RSSI IDLE  TXSEQ  RXSEQ CAPS FLAG
00:00:00:00:00:00    2   52 108M 20.0    0    862  44448 EP   AQHTRS+ HTCAP WME RSN
00:00:00:00:00:00    3   52 270M 15.0    0   1804  56576 EP   AQHTRS+ RSN HTCAP WME
```

My laptop had even the same IP address(es) as before, and none of my active TCP connections had been interrupted. Perfect! Happy that it works, I re-enabled the radio on the Get box II.

## Is the wireless coverage better?

At this point, I have accomplished what I set out to do: setup a working AP bridge. However, the wireless coverage in the house is not any better than when I started. I neglected to mention one tiny detail at the beginning of my post: `nasty.home` lives on the floor next to the entertainment center, less than a meter from the Get box II. It has to, because that's where the switch is (in the back of the all-in-one Get box II).

## To be continued...

Before I can determine if this exercise is a success, I need to move `nasty.home` somewhere else in the house (probably the basement). In order to move the NAS, I need to get a wired connection from the Get box II to the new location.

I plan to document the move in part two. Currently, I am eyeing some some [ethernet-over-power adapters](https://www.komplett.no/av600-gigabit-mini-powerline-adapter-kit/810081) to aid the move.

{% include block-image.html src="//www.komplett.no/img/p/400/f805b600-e4aa-4f54-8a2e-4402934afb0a.jpg" %}

 A friend of mine recently purchased these adapters with good success, which makes them a natural starting point. I will post part 2 once I get `nasty.home` moved and can test the wireless coverage in the troublesome areas of the house.
