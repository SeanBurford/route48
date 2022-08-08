## Goal

Configure a network so that clients can request prefixes, enabling them to
send and receive traffic for a range of addresses.


Have a look at [receiving](../../traffic/linux/receiving.md)
to learn how prefixes are requested and used on the client side.

## Solution Components

*  Providing prefixes: ISC Kea DHCPv6 server
*  Forwarding DHCPv6 and Learning routes: Cisco DHCPv6 Relay

### IP Addresses

In this document I use these addresses:

```
DHCP Server: 2001:db8:4:1::1
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
        "data": "2001:db8:4:1::1"
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

My DHCP server is on a different VLAN than my clients, so the Cisco router
in the middle needs to relay DHCP requests and responses.  When it does this
it needs to inspect the responses to learn routes for the allocated prefixes.

To enable DHCPv6 relay for clients on Vlan200 to a DHCPv6 server on Vlan231:

```
# configure term
(config)# interface Vlan200
(config-if)# ipv6 dhcp relay destination 2001:db8:4:1::1 Vlan231 
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

## What's next?

Take a look at ways to request a prefix [in Linux](../../traffic/linux/receiving.md)
