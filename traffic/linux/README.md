## Goal

Explore methods of sending traffic for a prefix.

Have a look at [receiving](receiving.md) to learn to request a prefix.

## Solution Components

*  Sending Traffic: Non Local Bind
   *  socat
   *  traceroute
*  Sending Traffic: IP_FREEBIND
   *  Freebind

### IP Addresses

In this document I use these addresses:

```
/64 Prefix: 2001:db8:4:7000::/64
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

## Notes

To cover:

*  https://www.davidc.net/networking/ipv6-source-address-selection-linux
*  https://superuser.com/questions/296789/how-does-ipv6-source-address-selection-work-in-linux
*  https://biplane.com.au/blog/?p=30
*  https://blog.widodh.nl/2016/03/docker-and-ipv6-prefix-delegation/




