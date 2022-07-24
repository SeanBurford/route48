## Goal

Explore methods of sending/receiving traffic for a prefix.

Have a look at [prefix delegation](../../routing/prefix_delegation/README.md)
to learn how prefixes are assigned.

## Solution Components

*  Receiving Traffic: AnyIP

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

We can now reach that host using any address in the prefix:

```
$ ping 2001:db8:4:7000::4
PING 2001:db8:4:7000::4(2001:db8:4:7000::4) 56 data bytes
64 bytes from 2001:db8:4:7000::4: icmp_seq=1 ttl=63 time=2.63 ms
64 bytes from 2001:db8:4:7000::4: icmp_seq=2 ttl=63 time=2.36 ms
```
