# Home Network

## Comments
Originally I had planned on having my wired and wireless networks on the same subnet. The problem was that with `ether5` bridged, IPv6 addresses were leaking to my guest wireless network. My solution was to unbridge `ether5` and move my wireless network to a different subnet (`172.16.1.0/24`). This has the added benefit of preventing my guest network from communicating with my wired and wireless networks. Finally, I added some firewall rules to prevent guest clients from communicating with each other; however, it remains untested as I lack a second ping capable wireless device.

## Goal
To create a secure, high performance, home network with dedicated access point and protected guest WiFi.

## Hardware
* MikroTik hEX RB750Gr3 5-port Ethernet Gigabit Router
* Ubiquiti UAP-AC-PRO-US

## Action Items
1. Secure router by setting strong admin password.
1. Allowed wired devices to join LAN and connect to the internet over IPv4 and IPv6.
1. Configure firewall to prevent unsolicited incoming connections.
1. Create queue to reduce bufferbloat.
1. Configure access point to serve WiFi over 2.4 GHz and 5 GHz secured by WPA2 AES.
1. Configure guest access point to serve WiFi over 2.4 GHz and 5 GHz. All traffic must be transmitted over anonymous VPN.

## Network
* Upstream will connect to router on Ether 1
* LAN will operate on `172.16.0.0/24` over Ether 2-5
* Guest WiFi will operate on `192.168.0.0/24` over VLAN on Ether 5 (last port)

## Configuration

### IPv4
```
/interface bridge
add fast-forward=no name=bridge-local
/interface ethernet
set [ find default-name=ether1 ] comment=Internet
/ip pool
add name=pool-ethernet ranges=172.16.0.2-172.16.0.254
add name=pool-wireless ranges=172.16.1.2-172.16.1.254
/ip dhcp-server
add address-pool=pool-ethernet disabled=no interface=bridge-local lease-time=1h name=dhcp-ethernet
/queue simple
add comment="https://forum.mikrotik.com/viewtopic.php\?t=101640" max-limit=100M/10M name=prevent-buffer-bloat target=ether1
/interface bridge port
add bridge=bridge-local hw=no interface=ether2
add bridge=bridge-local hw=no interface=ether3
add bridge=bridge-local hw=no interface=ether4
add bridge=bridge-local disabled=yes interface=ether5
/ip address
add address=172.16.0.1/24 interface=bridge-local network=172.16.0.0
add address=172.16.1.1/24 interface=ether5 network=172.16.1.0
/ip dhcp-client
add dhcp-options=hostname,clientid disabled=no interface=ether1 use-peer-dns=no use-peer-ntp=no
/ip dhcp-server network
add address=172.16.0.0/24 dns-server=8.8.8.8,8.8.4.4 gateway=172.16.0.1 netmask=24
add address=172.16.1.0/24 dns-server=8.8.8.8,8.8.4.4 gateway=172.16.1.1 netmask=24
/ip dns
set servers=8.8.8.8,8.8.4.4
/ip firewall nat
add action=masquerade chain=srcnat out-interface=ether1
```

### IPv4 Firewall
```
/ip firewall filter
add action=drop chain=input comment="drop invalid" connection-state=invalid
add action=accept chain=input comment="accept established and related" connection-state=established,related
add action=accept chain=input comment="accept DHCP (100/sec)" in-interface=ether1 limit=100,0:packet protocol=udp src-port=547
add action=drop chain=input comment="Drop DHCP" in-interface=ether1 protocol=udp src-port=547
add action=accept chain=input comment="accept external ICMP (100/sec)" in-interface=ether1 limit=100,0:packet protocol=icmpv6
add action=drop chain=input comment="drop external ICMP" in-interface=ether1 protocol=icmpv6
add action=accept chain=input comment="accept internal ICMP" in-interface=!ether1 protocol=icmpv6
add action=drop chain=input comment="drop external" in-interface=ether1
add action=reject chain=input comment="reject everything else"
add action=accept chain=output comment="accept all"
add action=drop chain=forward comment="drop invalid" connection-state=invalid
add action=accept chain=forward comment="accept established and related" connection-state=established,related
add action=accept chain=forward comment="accept external ICMP (100/sec)" in-interface=ether1 limit=100,0:packet protocol=icmpv6
add action=drop chain=forward comment="drop external ICMP" in-interface=ether1 protocol=icmpv6
add action=accept chain=forward comment="accept internal" in-interface=!ether1
add action=accept chain=forward comment="accept outgoing" out-interface=ether1
add action=drop chain=forward comment="drop external" in-interface=ether1
add action=reject chain=forward comment="reject everything else"
```

### Miscellaneous
```
/snmp community
set [ find default=yes ] addresses=0.0.0.0/0
/system clock
set time-zone-name=America/Denver
/system ntp client
set enabled=yes primary-ntp=63.211.239.58 secondary-ntp=45.63.20.61
/system routerboard settings
set silent-boot=no
```

### IPv6 Firewall
```
/ipv6 firewall filter
add action=drop chain=input comment="drop invalid" connection-state=invalid
add action=accept chain=input comment="accept established and related" connection-state=established,related
add action=accept chain=input comment="accept DHCP (100/sec)" in-interface=ether1 limit=100,0:packet protocol=udp src-port=547
add action=drop chain=input comment="Drop DHCP" in-interface=ether1 protocol=udp src-port=547
add action=accept chain=input comment="accept external ICMP (100/sec)" in-interface=ether1 limit=100,0:packet protocol=icmpv6
add action=drop chain=input comment="drop external ICMP" in-interface=ether1 protocol=icmpv6
add action=accept chain=input comment="accept internal ICMP" in-interface=!ether1 protocol=icmpv6
add action=drop chain=input comment="drop external" in-interface=ether1
add action=reject chain=input comment="reject everything else" reject-with=icmp-no-route
add action=accept chain=output comment="accept all"
add action=drop chain=forward comment="drop invalid" connection-state=invalid
add action=accept chain=forward comment="accept established and related" connection-state=established,related
add action=accept chain=forward comment="accept external ICMP (100/sec)" in-interface=ether1 limit=100,0:packet protocol=icmpv6
add action=drop chain=forward comment="drop external ICMP" in-interface=ether1 protocol=icmpv6
add action=accept chain=forward comment="accept internal" in-interface=!ether1
add action=accept chain=forward comment="accept outgoing" out-interface=ether1
add action=drop chain=forward comment="drop external" in-interface=ether1
add action=reject chain=forward comment="reject everything else" reject-with=icmp-no-route
```

### IPv6
```
/ipv6 address
add address=::1 from-pool=ethernet-pool interface=bridge-local
/ipv6 dhcp-client
add add-default-route=yes comment=\
    https://www.medo64.com/2018/03/setting-ipv6-on-mikrotik/ interface=ether1 pool-name=ethernet-pool request=prefix
```

### Guest WiFi IPv4
```
/interface vlan
add interface=ether5 name=vlan100-ethereal vlan-id=100
/ip address
add address=192.168.0.1/24 interface=vlan100-ethereal network=192.168.0.0
/ip pool
add name=pool-ethereal ranges=192.168.0.2-192.168.0.254
/ip dhcp-server
add address-pool=pool-ethereal disabled=no interface=ethereal lease-time=1h name=dhcp-ethereal
/ip dhcp-server network
add address=192.168.0.0/24 dns-server=8.8.8.8,8.8.4.4 gateway=192.168.0.1 netmask=24

/ppp profile
add name=privateinternetaccess-com use-encryption=required
/interface pptp-client
add allow=mschap2 comment="https://www.reddit.com/r/mikrotik/comments/2yb6ph/virtual_access_points_different_dhcp_ip_ranges/cp7xdke" connect-to=us-denver.privateinternetaccess.com disabled=no mrru=1600 name=us-denver-privateinternetaccess-com password=PASSWORD profile=privateinternetaccess-com user=x0000000

/ip firewall filter
add action=drop chain=forward comment="Prevent VPN IP Address Leak" out-interface=ether1 routing-mark=use-us-denver-privateinternetaccess-com
add action=accept chain=forward comment="Ethereal Wireless Isolation TEST ME" dst-address=192.168.0.1 src-address=192.168.0.0/16
add action=drop chain=forward comment="Ethereal Wireless Isolation TEST ME" dst-address=!198.168.0.1 src-address=198.168.0.0/16
/ip firewall mangle
add action=mark-routing chain=prerouting new-routing-mark=use-us-denver-privateinternetaccess-com passthrough=yes src-address=192.168.0.0/24
/ip firewall nat
add action=masquerade chain=srcnat out-interface=us-denver-privateinternetaccess-com
/ip route
add distance=1 gateway=us-denver-privateinternetaccess-com routing-mark=use-us-denver-privateinternetaccess-com

# Optionally throttle guest connections.
/queue simple
add max-limit=1M/8M name=ethereal-throttle target=192.168.0.0/12
```
