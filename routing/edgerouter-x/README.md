## Goal

Establish an IPv6 in GRE/SIT tunnel from a Ubiquiti
EdgeMax device (eg. an EdgeRouter-X) to Route48 or
Hurricane Electric.

### Simple Examples

If you just want to get up and running, without any
firewalling or routing, these services provide basic
configuration examples for many devices:

Hurricane Electric:

1. Log into https://tunnelbroker.net
2. Click on one of your configured tunnels
3. Select the `Example Configurations` tab
4. Select a device from the dropdown.

Route48:

1. Log into https://route48.org
2. Select `IPv6 Tunnels`
3. Click `Config` on a GRE/SIT tunnel
4. Select a device from the tabs

### IP Addresses

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
```

## Firewall Rules

Since I want to control traffic, I add some firewall
rules to the tunnel interface:

```
set interfaces tunnel tun0 firewall in name ESTAB
set interfaces tunnel tun0 firewall in ipv6-name ESTAB6
set interfaces tunnel tun0 firewall local name ESTAB
set interfaces tunnel tun0 firewall local ipv6-name ESTAB6
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

## Simple Outbound Routing

If I just wanted to route all IPv6 traffic through the tunnel, I
would use this command:

```
set protocols static interface-route6 ::/0 next-hop-interface tun0
```

## Source Based Outbound Routing

In my case I already have other IPv6 connectivity, but I want traffic
for the tunneled network to use the tunnel.

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

And I assign the ipv6-modify ruleset to my internal network interface(s):

```
set interfaces ethernet eth0 firewall in ipv6-modify IPv6_OUT
```

## Apply the config

To check the config diff before applying it:

```
show
```

To apply and save the config:

```
commit
save
```

## Confirmation

Here are a couple of commands to verify that the router can reach the
other side of the tunnel.

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

## Notes

### UI

The EdgeMax web UI has poor support for IPv6, however
this configuration can easily be replicated either in the
`config tree` tab of the UI, or on the command line via SSH.
Examining the `config tree` tab can be helpful since it
provides hints about what settings are available.

### Hardware offload

On older firmware you needed to disable hardware offload
to avoid some problems.  See these links for more info:

*  [Random lockups with HWNAT offload enabled](https://community.ui.com/releases/EdgeMAX-EdgeRouter-Firmware-v2-0-9-v2-0-9/d75f346d-d734-4026-97a8-7b2d5cc4e079)
*  [EdgeRouter Hardware Offloading](https://help.ui.com/hc/en-us/articles/115006567467-EdgeRouter-Hardware-Offloading)
*  [Speedtest with/without HW Offload](https://www.patnotebook.com/edgerouter-x-hardware-offloading/)

## Further Reading

*  [VyOS Tunnel Interfaces Doc](https://docs.vyos.io/en/equuleus/configuration/interfaces/tunnel.html)

## What's Next?

Now that you can route tunneled traffic, take a look at
[prefix delegation](../prefix_delegation/README.md)
