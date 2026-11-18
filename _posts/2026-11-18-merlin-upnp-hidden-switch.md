---
layout: post
title: "Asuswrt-Merlin: UPnP can say it's on and still forward nothing"
date: 2026-11-18
categories: [infrastructure]
tags: [asus, asuswrt-merlin, upnp, nat-pmp, tailscale, router]
---

I recently moved my remote access from WireGuard to [Tailscale](https://tailscale.com/). Tailscale connects two devices directly when it can, and through its own relay servers when it can't. My phone on mobile data reached my home server just fine, but the app kept saying **relayed**, at 59-64 ms a ping. That bugged me, because UPnP was on in my router, which is exactly what is supposed to make a direct connection possible.

It turned out my Asus router (Asuswrt-Merlin firmware) had two UPnP switches, and only one of them is in the UI.

## Relayed, though the router granted the port

When one side can't be reached through hole punching (mobile carriers' NAT often can't), Tailscale needs an open port on the other side. It asks the router for one with NAT-PMP, a protocol a LAN device uses to ask the router to forward a port to it. On Asus routers, miniupnpd answers it, as well as UPnP, the older protocol for the same job.

The router said yes. Tailscale on the server reported a mapped port, and the router's lease file listed it:

```bash
cat /tmp/upnp.leases
```

Yet the phone stayed relayed. My router already logged dropped packets, and there they were: the phone's UDP packets, sent straight to the mapped ports, dropped at the WAN.

So the mapping existed on paper only. Port forwards on Asus routers live in the `nat` table of iptables (the Linux firewall), in a chain (a named list of rules) called `VSERVER`. UPnP mappings go into their own chain, `VUPNP`, which `VSERVER` should jump to:

```
# iptables -t nat -S VUPNP
-N VUPNP
# iptables -t nat -S VSERVER | grep VUPNP
#
```

`VUPNP` was empty, and nothing jumped to it.

## Two switches, one in the UI

The router keeps its settings in nvram, a key-value store that the firmware reads at boot and whenever it rebuilds the firewall. The **Enable UPnP** checkbox on the WAN page is `wan0_upnp_enable`, and it starts miniupnpd. But the jump into `VUPNP` checks a different, global flag, [`upnp_enable`](https://github.com/RMerl/asuswrt-merlin.ng/blob/3006.102.9/release/src/router/rc/firewall.c#L2024):

```c
if (is_nat_enabled() && nvram_get_int("upnp_enable") && *wan_if)
{
	/* call UPNP chain */
	fprintf(fp, "-A VSERVER -j VUPNP\n");
	fprintf(fp, "-A POSTROUTING -o %s -j PUPNP\n", wan_if);
}
```

It has no checkbox anywhere and [defaults to `1`](https://github.com/RMerl/asuswrt-merlin.ng/blob/3006.102.9/release/src/router/shared/defaults.c#L2631). On my router it was `0`:

```
# nvram get wan0_upnp_enable
1
# nvram get upnp_enable
0
```

With that combination, miniupnpd grants every mapping and none of them ever apply. No error, no log line: the UI says on, the leases look right, and the packets die at the WAN.

What I couldn't find is what set it to `0`. Merlin's source never writes it, and neither do my own scripts. I factory reset the router when I moved to Merlin in October and set it up again by hand, so it wasn't an old setting carried over either. I did this digging with Claude Code (an AI agent): it read the router's settings and firewall rules and searched the firmware source, while I ran the tests from the phone and decided what to change.

## The fix

Three commands on the router set it back:

```bash
nvram set upnp_enable=1
nvram commit
service restart_firewall
```

`nvram commit` writes it to flash, so it survives a reboot (I rebooted to make sure). After that, `VSERVER` ends with the jump, and the mappings show up in `VUPNP`. I restarted Tailscale on the server so it asked for a fresh mapping right away. The next ping from the phone on mobile data went direct, at 33 ms.

## Check your router

If you rely on UPnP or NAT-PMP on Asuswrt-Merlin, one command on the router tells you:

```bash
iptables -t nat -S VSERVER | grep VUPNP
```

You want to see `-A VSERVER -j VUPNP`. If it's missing while UPnP is on in the UI, `nvram get upnp_enable` is likely `0`.

## Setup

Asus GT-AX6000 on Asuswrt-Merlin 3006.102.9, Tailscale 1.104 on the server, checked on 2026-11-17. I haven't checked stock Asus firmware.

## Closing thoughts

I knew from the start that UPnP was the problem: a phone on mobile data has no trouble going direct, so the router was the piece letting me down. What took the time was finding where, because everywhere I knew to look said all was fine.

How `upnp_enable` got to `0` on my router is still a mystery to me. It also makes me wonder how many more settings like it sit in nvram: a default, no UI, no log line, just waiting to break something.
