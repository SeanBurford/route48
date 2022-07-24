## Goal

Explore methods of sending/receiving traffic for a prefix.

Have a look at [prefix delegation](../../routing/prefix_delegation/README.md)
to learn how prefixes are assigned.

## Solution Components

*  DHCP script
*  Receiving Traffic: AnyIP
*  Sending Traffic: Non Local Bind
   *  socat
   *  traceroute
*  Sending Traffic: IP_FREEBIND
   *  Freebind

### DHCP Script

This script will add an AnyIP route, as described below.

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

### IP Addresses

In this document I use these addresses:

```
/64 Prefix: 2001:db8:4:7000::/64
```

## Receiving Traffic: AnyIP

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

## Sending Traffic: Non Local Bind

`IP_FREEBIND` allows processes to bind IP addresses that are non local
or that don't exist (yet).

To enable outbound traffic from addresses that aren't bound to interfaces
we need to either provide the `IP_FREEBIND` socket option or enable non
local bind system wide.  To enable it system wide:

```
echo 1 > /proc/sys/net/ipv6/ip_nonlocal_bind
```

*  [IP_FREEBIND reference](https://man7.org/linux/man-pages/man7/ip.7.html)

### socat

With AnyIP and local bind set up we can use socat to originate traffic from our prefix:

```
$ socat -dd TCP6:telnetmyip.com:23,bind=[2001:db8:4:7000::1:2:3],setsockopt-int=0:15:1 -
{
  "comment": "##     Your IP Address is 2001:db8:4:7000:0:1:2:3 (52933)     ##",
  "family": "ipv6",
  "ip": "2001:db8:4:7000:0:1:2:3",
  "port": "52933",
  "protocol": "telnet",
  "version": "v1.3.0",
  "force_ipv4": "ipv4.telnetmyip.com",
  "force_ipv6": "ipv6.telnetmyip.com",
  "website": "https://github.com/packetsar/checkmyip",
}
```

### traceroute

With AnyIP and local bind set up we can use traceroute to originate traffic from our prefix:

```
$ traceroute6 -s 2001:db8:4:7000::5 ipv6.google.com
traceroute to ipv6.l.google.com (2404:6800:4006:80b::200e) from 2001:db8:4:7000::5, port 33434, from port 22508, 30 hops max, 60 bytes packets
...
 3  tid-2365.au-adelaide.route48.org (2001:db8:4::1)  32.289 ms  29.969 ms  29.958 ms 
 4  xe-0-0-0.cr1.dci.sa.au.cloudie.network (2602:fba1:a00::1)  28.940 ms  29.221 ms  30.106 ms 
 5  2407:a080:5000:b::a (2407:a080:5000:b::a)  29.114 ms  29.997 ms  27.323 ms 
 6  2407:a080:5000:10::4 (2407:a080:5000:10::4)  36.101 ms  37.274 ms  34.995 ms 
 7  2402:1b80:401::6 (2402:1b80:401::6)  27.389 ms  30.915 ms  25.225 ms 
...
```

## Sending Traffic: IP_FREEBIND

### Freebind

https://github.com/blechschmidt/freebind hooks socket calls to enable `IP_FREEBIND`
for any tool:

```
$ freebind -r 2001:db8:4:7000::/64 curl https://ifconfig.co/
2001:db8:4:7000:9edd:db79:a5b3:8464
$ freebind -r 2001:db8:4:7000::/64 curl https://ifconfig.co/
2001:db8:4:7000:f896:47fb:3ec7:5aaf
```



