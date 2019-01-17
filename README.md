# Home-Network

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
/ip dhcp-server
add address-pool=pool-ethernet disabled=no interface=bridge-local lease-time=\
    1h name=dhcp-ethernet
/queue simple
add comment="https://forum.mikrotik.com/viewtopic.php\?t=101640" max-limit=\
    100M/10M name=prevent-buffer-bloat target=ether1
/interface bridge port
add bridge=bridge-local hw=no interface=ether2
add bridge=bridge-local hw=no interface=ether3
add bridge=bridge-local hw=no interface=ether4
add bridge=bridge-local hw=no interface=ether5
/ip address
add address=172.16.0.1/24 interface=bridge-local network=172.16.0.0
/ip dhcp-client
add dhcp-options=hostname,clientid disabled=no interface=ether1 use-peer-dns=\
    no use-peer-ntp=no
/ip dhcp-server network
add address=172.16.0.0/24 dns-server=8.8.8.8,8.8.4.4 gateway=172.16.0.1 \
    netmask=24
/ip dns
set servers=8.8.8.8,8.8.4.4
/ip firewall nat
add action=masquerade chain=srcnat out-interface=ether1
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

### IPv6
```
/ipv6 address
add address=::1 from-pool=ethernet-pool interface=bridge-local
/ipv6 dhcp-client
add add-default-route=yes comment=\
    https://www.medo64.com/2018/03/setting-ipv6-on-mikrotik/ interface=ether1 \
    pool-name=ethernet-pool request=prefix
/ipv6 firewall filter
add action=drop chain=input comment="drop invalid" connection-state=invalid
add action=accept chain=input comment="accept established and related" \
    connection-state=established,related
add action=accept chain=input comment="accept DHCP (100/sec)" in-interface=\
    ether1 limit=100,0:packet protocol=udp src-port=547
add action=drop chain=input comment="Drop DHCP" in-interface=ether1 protocol=\
    udp src-port=547
add action=accept chain=input comment="accept external ICMP (100/sec)" \
    in-interface=ether1 limit=100,0:packet protocol=icmpv6
add action=drop chain=input comment="drop external ICMP" in-interface=ether1 \
    protocol=icmpv6
add action=accept chain=input comment="accept internal ICMP" in-interface=\
    !ether1 protocol=icmpv6
add action=drop chain=input comment="drop external" in-interface=ether1
add action=reject chain=input comment="reject everything else" reject-with=\
    icmp-no-route
add action=accept chain=output comment="accept all"
add action=drop chain=forward comment="drop invalid" connection-state=invalid
add action=accept chain=forward comment="accept established and related" \
    connection-state=established,related
add action=accept chain=forward comment="accept external ICMP (100/sec)" \
    in-interface=ether1 limit=100,0:packet protocol=icmpv6
add action=drop chain=forward comment="drop external ICMP" in-interface=\
    ether1 protocol=icmpv6
add action=accept chain=forward comment="accept internal" in-interface=\
    !ether1
add action=accept chain=forward comment="accept outgoing" out-interface=\
    ether1
add action=drop chain=forward comment="drop external" in-interface=ether1
add action=reject chain=forward comment="reject everything else" reject-with=\
    icmp-no-route
```
