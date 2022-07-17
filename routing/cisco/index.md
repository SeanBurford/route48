## Configurations

*  Static routing for an interface

### Static routing for an interface

#### Goal

Route IPv6 traffic for one subnet on one VLAN.

#### Cisco Configuration

Configure VLAN 201 to use subnet `2001:db8:4:64::/64` with the router at `2001:db8:4:64::ffff`:

```
# configure terminal
(config)# ipv6 general-prefix route48 2001:db8:4::/48
(config)# interface Vlan201
(config-if)# ipv6 address route48 ::64:0:0:0:ffff/64
```
