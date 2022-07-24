## Goal

Assign entire prefixes to that clients can use a range of addresses from those
prefixes.

## Solution Components

*  Providing prefixes: ISC Kea DHCPv6 server
*  Forwarding DHCPv6 and Learning routes: Cisco DHCPv6 Relay
*  Requesting a prefix: Linux Client

### IP Addresses

In this document I use these addresses:

```
DHCP Server: 2001:db8:4:fffe::1
Routed /48 Tunnel Network: 2001:db8:4::/48
```

### Providing prefixes: ISC Kea DHCPv6 server

This [Kea DHCPv6](https://kea.readthedocs.io/en/latest/arm/dhcp6-srv.html)
server config allocates the upper half of the subnet for (up to 128) clients
that would like a /56, the ::7000:: block for (up to 4096) clients that would
like a /64, and it allocates a /64 for individual client addresses:

```
# cat /etc/kea/kea-dhcp6.conf
{
  "Dhcp6" : {
    "option-data" : [
...
      {
        "name": "unicast",
        "data": "2001:db8:4:fffe::1"
      }
    ],
...
    "subnet6" : [
      {
        "id" : 10
        "subnet" : "2001:db8:4::/48",
        "pools" : [
          { "pool" : "2001:db8:4:1::/64" }
        ],
        "pd-pools" : [
          {
            "prefix-len" : 52,
            "delegated-len" : 64,
            "prefix" : "2001:db8:4:7000::"
          },
          {
            "prefix-len" : 49,
            "delegated-len" : 56,
            "prefix" : "2001:db8:4:8000::"
          }
        ]
      }
    ],
...
```

### Forwarding DHCPv6 and Learning routes: Cisco DHCPv6 Relay

To enable DHCPv6 relay for clients on Vlan200 to a DHCPv6 server on Vlan231:

```
# configure term
(config)# interface Vlan200
(config-if)# ipv6 dhcp relay destination 2001:db8:4:fffe::1 Vlan231 
```

Once we have users, we can check on learnt routes:

```
>show ipv6 route
IPv6 Routing Table - default - 23 entries
Codes: C - Connected, L - Local, S - Static, U - Per-user Static route
       R - RIP, D - EIGRP, EX - EIGRP external, ND - ND Default
       NDp - ND Prefix, DCE - Destination, NDr - Redirect, RL - RPL
       O - OSPF Intra, OI - OSPF Inter, OE1 - OSPF ext 1, OE2 - OSPF ext 2
       ON1 - OSPF NSSA ext 1, ON2 - OSPF NSSA ext 2, a - Application
...
S   2001:db8:4:7000::/64 [1/0]
     via FE80::CC40:..., Vlan200
```

*  [Cisco provided examples](https://community.cisco.com/t5/networking-knowledge-base/stateful-dhcpv6-relay-configuration-example/ta-p/3149338)

### Requesting a prefix: Linux Client

The [Debian IPv6PrefixDelegation](https://wiki.debian.org/IPv6PrefixDelegation)
page has configurations for using SLAAC or DHCP for address assignment or prefix 
delegation, along with route discovery.

This configuration uses SLAAC for address configuration and DHCP for prefix
delegation.

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

The delegated prefix isn't assigned to anything by default, however
we can now observe it on the Cisco device (`show ipv6 route`) and
can assign addresses from it to an interface:

```
# ip address add 2001:db8:4:7000::1/64 dev wlp3s0
# traceroute6 -s 2001:db8:4:7000::1 ipv6.google.com
```

#### Systemd-networkd

Systemd-networkd users should refer to the
[relevant documentation](https://www.freedesktop.org/software/systemd/man/systemd.network.html)
for `/etc/systemd/network/`.  The examples at the end include a router which
uses prefix delegation to get a prefix from upstream.
[This blog post](https://blog.g3rt.nl/systemd-networkd-dhcpv6-pd-configuration.html) contains
a good example.

