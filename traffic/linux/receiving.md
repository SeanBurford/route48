## Goal

Explore methods of receiving traffic for a prefix in Linux.

Have a look at [prefix delegation](../../routing/prefix_delegation/README.md)
to learn how prefixes are assigned on the server side.

## Solution Components

*  Receiving Traffic: AnyIP
*  Requesting a prefix with interfaces.d
*  Requesting a prefix with systemd-networkd

### IP Addresses

In this document I use these addresses:

```
/64 Prefix: 2001:db8:4:7000::/64
```

## Receiving Traffic: AnyIP

Once we have a prefix, 
[AnyIP](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git)
is the capability to receive packets and establish incoming connections
on IPs we have not explicitly configured on the machine.

Enabling AnyIP is simple:

```
# ip -6 route add local  2001:db8:4:7000::/64 dev lo
```

You can verify the entry with:

```
$ ip -6 route list table local
...
local 2001:db8:4:7000::/64 dev lo metric 1024 pref medium
...
```

We can now reach that host using any address in the prefix:

```
$ ping 2001:db8:4:7000::4
PING 2001:db8:4:7000::4(2001:db8:4:7000::4) 56 data bytes
64 bytes from 2001:db8:4:7000::4: icmp_seq=1 ttl=63 time=2.63 ms
64 bytes from 2001:db8:4:7000::4: icmp_seq=2 ttl=63 time=2.36 ms
```

### DHCP Script

This script will add an AnyIP route on the client machine if it receives a prefix.

`/etc/dhcp/dhclient-exit-hooks.d/prefix-delegation`:

```
#!/bin/bash

# $interface:          Device (eg. eth0)
# $reason:             Reason for DHCP result.
# $new_ip6_prefix:     Assigned prefix (eg. '2001:db8:4:7000::/56')
# $new_max_life:       TTL for new prefix ('0' = requested prefix rejected)
# $new_preferred_life  TTL for new prefix ('0' = rejected)

if [ ! -z "$new_ip6_prefix" ]; then
        if [ "$new_max_life" != "0" ]; then
                echo "Adding prefix ${new_ip6_prefix}"
                ip -6 route add local "$new_ip6_prefix" dev lo 2>/dev/null
        fi

        if [ ! -z "$old_ip6_prefix" ] && [ "$old_ip6_prefix" != "$new_ip6_prefix" ]; then
                echo "Removing old prefix ${old_ip6_prefix}"
                ip -6 route del local "$old_ip6_prefix" dev lo
        fi
fi
```

## Requesting a prefix

We will look at two methods of requesting a prefix:

*  interfaces.d
*  systemd-networkd

### Requesting a prefix with interfaces.d

The [Debian IPv6PrefixDelegation](https://wiki.debian.org/IPv6PrefixDelegation)
page has configurations for using SLAAC or DHCP for address assignment or prefix
delegation, along with route discovery.

This configuration uses SLAAC for address configuration and DHCP for prefix
delegation:

```
# cat /etc/network/interfaces.d/wlp3s0-pd
iface wlp3s0 inet6 auto
        dhcp 1
        accept_ra 2
        request_prefix 1
```

`accept_ra 2` learns the default route from SLAAC regardless of DHCP.  Possible
values are:

0.  Do not accept router advertisements
1.  Accept router advertisements if forwarding is disabled
2.  Accept router advertisements even if forwarding is enabled

This interface can be enabled with `ifup wlp3s0`:

```
# ifup wlp3s0
Listening on Socket/wlp3s0
Sending on   Socket/wlp3s0
PRC: Soliciting for leases (INIT).
XMT: Forming Solicit, 0 ms elapsed.
XMT:  X-- IA_PD c1:a1:07:3e
XMT:  | X-- Request renew in  +3600
XMT:  | X-- Request rebind in +5400
XMT: Solicit on wlp3s0, interval 1080ms.
RCV: Advertise message on wlp3s0 from fe80::...
RCV:  X-- IA_PD c1:a1:07:3e
RCV:  | X-- starts 1658559376
RCV:  | X-- t1 - renew  +1000
RCV:  | X-- t2 - rebind +2000
RCV:  | X-- [Options]
RCV:  | | X-- IAPREFIX 2001:db8:4:7000::/64
RCV:  | | | X-- Preferred lifetime 3000.
RCV:  | | | X-- Max lifetime 4000.
RCV:  X-- Server ID: 00:01:00:01:27:dd:...
RCV:  Advertisement recorded.
PRC: Selecting best advertised lease.
PRC: Considering best lease.
PRC:  X-- Initial candidate 00:01:00:01:27:dd:... (s: 10105, p: 0).
XMT: Forming Request, 0 ms elapsed.
XMT:  X-- IA_PD c1:a1:07:3e
XMT:  | X-- Requested renew  +3600
XMT:  | X-- Requested rebind +5400
XMT:  | | X-- IAPREFIX 2001:db8:4:7000::/64
XMT:  | | | X-- Preferred lifetime +7200
XMT:  | | | X-- Max lifetime +7500
XMT:  V IA_PD appended.
XMT: Request on wlp3s0, interval 1030ms.
RCV: Reply message on wlp3s0 from fe80::...
RCV:  X-- IA_PD c1:a1:07:3e
RCV:  | X-- starts 1658559378
RCV:  | X-- t1 - renew  +1000
RCV:  | X-- t2 - rebind +2000
RCV:  | X-- [Options]
RCV:  | | X-- IAPREFIX 2001:db8:4:7000::/64
RCV:  | | | X-- Preferred lifetime 3000.
RCV:  | | | X-- Max lifetime 4000.
RCV:  X-- Server ID: 00:01:00:01:27:dd:...
PRC: Bound to lease 00:01:00:01:27:dd:...
```

We can now observe a route for the prefix on the Cisco device
(`show ipv6 route`).

The delegated prefix isn't assigned to any interface by default,
unless you use a DHCP script as described above.  You can also
assign addresses from the prefix to an interface by hand:

```
# ip address add 2001:db8:4:7000::1/64 dev wlp3s0
# traceroute6 -s 2001:db8:4:7000::1 ipv6.google.com
```

### Requesting a prefix with systemd-networkd

Systemd-networkd users should refer to the
[relevant documentation](https://www.freedesktop.org/software/systemd/man/systemd.network.html)
for `/etc/systemd/network/`.  The examples at the end include a router which
uses prefix delegation to get a prefix from upstream.
[This blog post](https://blog.g3rt.nl/systemd-networkd-dhcpv6-pd-configuration.html) contains
a good example.

## What's next?

Take a look at ways to use a prefix [in Linux](README.md)
