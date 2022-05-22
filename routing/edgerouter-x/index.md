## Goal

Establish an IPv6 in GRE/SIT tunnel from an EdgeRouter-X
to Route48 or Hurricane Electric.

## IP Addresses

In this document I use these addresses:

```
Server IPv4 Address (tunnel server): 128.66.0.1
Client IPv4 Address (my IP address): 192.0.2.1
Routed /48 Tunnel Network: 2001:db8:4::/48
Server IPv6 Address: 2001:db8:4::1/48
Client IPv6 Address: 2001:db8:4::2/48
```

## GRE Tunnel Interface

To establish the tunnel we create a tunnel interface:

```
set interfaces tunnel tun0 description "route48 tunnel"
set interfaces tunnel tun0 local-ip 192.0.2.1
set interfaces tunnel tun0 remote-ip 128.66.0.1 
set interfaces tunnel tun0 address 2001:db8:4::2/48
set interfaces tunnel tun0 encapsulation sit           
set interfaces tunnel tun0 multicast disable     
set interfaces tunnel tun0 ttl 64     
set interfaces tunnel tun0 firewall in name ESTAB
set interfaces tunnel tun0 firewall in ipv6-name ESTAB6
set interfaces tunnel tun0 firewall out ipv6-name TO_WAN6
set interfaces tunnel tun0 firewall out name TO_WAN      
```

For the curious, my existing `ESTAB6` and `TO_WAN6` firewall
rules look like this:

```
    ipv6-name ESTAB6 {
        default-action drop
        description "IPv6 established/related only"
        rule 10 {
            action drop
            description "Drop invalid"
            log disable
            protocol all
            state {
                invalid enable
            }
        }
        rule 20 {
            action accept
            description ICMPv6
            protocol icmpv6
        }
        rule 30 {
            action accept
            description Established/Related
            protocol all
            state {
                established enable
                related enable
            }
        }
    }
    ipv6-network-group local {
        ...
        ipv6-network 2001:db8:4::/48
    }
    ipv6-network-group blackhole {
        ...
        ipv6-network 100::/64
    }
    ipv6-name TO_WAN6 {
        default-action accept
        description "Default to WAN"
        rule 10 {
            action reject
            description "Traffic Leak"
            destination {
                group {
                    ipv6-network-group local
                }
            }
            protocol all
        }
        rule 11 {
            action reject
            description Blackhole
            destination {
                group {
                    ipv6-network-group blackhole
                }
            }
            protocol all
        }
    }
```

## Outbound Routing

I already have other IPv6 connectivity, but I want traffic for the
tunneled network to use the tunnel.

In order to send traffic for the allocated /48 through this tunnel,
without impacting traffic from other IPv6 ranges, I create a route
table and apply it to the source address range.  As a result, traffic
from my network that is using addresses in the allocated range will
go out through the tunnel:

```
set protocols static table 101 interface-route6 ::/0 next-hop-interface tun0
set firewall ipv6-modify IPv6_OUT rule 30 action modify 
set firewall ipv6-modify IPv6_OUT rule 30 description "Outbound route48 routing"
set firewall ipv6-modify IPv6_OUT rule 30 modify table 101                      
set firewall ipv6-modify IPv6_OUT rule 30 source address 2001:db8:4::/48
```

And I assign the ipv6-modify ruleset to my internal network interfaces
on the EdgeRouter-X:

```
set interfaces ethernet eth0 firewall in ipv6-modify IPv6_OUT
```

## Confirmation

Examining the routing table:

```
ubnt@ubnt:~$ show ipv6 route table 101
default dev tun0 metric 1024 pref medium
ubnt@ubnt:~$ show ipv6 route
...
C      2001:db8:4::/48 via ::, tun0, 00:17:28
```

Pinging the default gateway:

```
ubnt@ubnt:~$ sudo ping 2001:db8:4::1
PING 2001:db4:4::1(2001:db8:4::1) 56 data bytes
64 bytes from 2001:db8:4::1: icmp_seq=1 ttl=64 time=25.2 ms
64 bytes from 2001:db8:4::1: icmp_seq=2 ttl=64 time=23.8 ms
64 bytes from 2001:db8:4::1: icmp_seq=3 ttl=64 time=23.7 ms
```

