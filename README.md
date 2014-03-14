# MalGate

## About
This guide describes the process for setting up a lightweight (~6MB compressed) VMware Appliance that applies different
types of routing and levels of anonymity when performing malware analysis.

Virtual machines can be routed directly to host's network, over a VPN connection, or through TOR.
Logging is not currently configured, but will implemented in the future.
Bridging other physical machines into virtual networks is not yet supported but will be added (through USB nic).

## Disclaimer
This work is not yet complete. There are many features I intend to add:
+ An EC2 VPN endpoint build script
+ The VPN configuration
+ A python script for managing the TOR circuits
+ Logging configuration
+ Integration with alienvault
+ Integration with bro

## Download
My current VM is not sanitized. I intend to upload a prebuilt, sanitized VM once I have built it.

## Naming
I have done my best to keep the naming conventions clean and standardized.
I have left the default vmware virtual networks untouched, and added 3 custom networks: vmnet11, vmnet12, and vmnet13.
The subnets of each of these networks correspond to their vmnet ID.

## Access & Authentication
SSH & HTTP access to the gateway has been enabled on the WAN interface.
The default credentials are: root/malware.

## Disclaimer
This is super raw and probably completely full of errors at the moment.  Needs to be cleaned up.
ICMP from the tor network is currently being forwarded out through the WAN interface, and not being encapsulated by
TOR.  This will allow malware to test for connectivity with ICMP, but also exposes your public address in that case.

## Network Configuration

| Name    | Type      | Adapter | DHCP     | Subnet         | Note    |
| ------- | --------- | ------- | -------- | -------------- | ------- |
| vmnet0  | Bridged   | Bridged | Disabled | N/A            | Default |
| vmnet1  | Bridged   | None    | Enabled  | 192.168.58.0   | Default |
| vmnet8  | NAT       | NAT     | Enabled  | 192.168.168.0  | Default |
| vmnet11 | Host-Only | None    | Disabled | 172.21.11.0/24 | Direct  |
| vmnet12 | Host-Only | None    | Disabled | 172.21.12.0/24 | VPN     |
| vmnet13 | Host-Only | None    | Disabled | 172.21.13.0/24 | TOR     |

## Base Configuration

    # Set password for root user:
    passwd
    
    # Set the vm hostname:
    uci set system.@system[0].hostname=gateway

    # update package manifest
    opkg update

    # install vim & tor
    opkg install vim-full tor 

## WAN Configuration

    # create the WAN interface:
    uci set network.wan=interface
    uci set network.wan.ifname=eth0
    uci set network.wan.proto=dhcp
    uci set network.wan.defaultroute=0
    uci set network.wan.peerdns=0

    # permit SSH connections from WAN:
    uci add firewall rule
    uci set firewall.@rule[-1].src=wan
    uci set firewall.@rule[-1].target=ACCEPT
    uci set firewall.@rule[-1].proto=tcp
    uci set firewall.@rule[-1].dest_port=22

    # permit HTTP connections from WAN:
    uci add firewall rule
    uci set firewall.@rule[-1].src=wan
    uci set firewall.@rule[-1].target=ACCEPT
    uci set firewall.@rule[-1].proto=tcp
    uci set firewall.@rule[-1].dest_port=80

    # commit the firewall changes and restart the firewall
    uci commit firewall
    /etc/init.d/firewall restart

We can now connect to the router from the WAN interface via SSH or HTTP.

## LAN Configuration
This network routes traffic to internet via host's physical network.

    # create the network & bind to interface
    uci set network.lan=interface
    uci set network.lan.ifname=eth1
    uci set network.tor.proto=static
    uci set network.lan.ipaddr=172.21.11.1
    uci set network.lan.netmask=255.255.255.0

    # configure dhcp
    uci set dhcp.lan.start=200
    uci set dhcp.lan.stop=225
    uci set dhcp.lan.limit=25
    uci set dhcp.lan.leasetime=1h

    # commit changes and restart network services
    uci commit network
    /etc/init.d/network restart

## VPN Configuration
This network routes traffic to internet through a VPN connection.

    # create the network & bind to interface
    uci set network.vpn.ifname=eth2
    uci set network.tor.proto=static
    uci set network.vpn.ipaddr=172.21.12.1
    uci set network.vpn.netmask=255.255.255.0

    # configure dhcp
    uci set dhcp.vpn.start=200
    uci set dhcp.vpn.stop=225
    uci set dhcp.vpn.limit=25
    uci set dhcp.vpn.leasetime=1h

    # commit changes and restart network services
    uci commit network
    /etc/init.d/network restart

Note: The VPN configuration section needs to be made to work in a 'sharable' way.

## TOR Configuration
This network routes traffic to internet through a TOR gateway and proxy.
ICMP currently leaks.

    # create tor network and bind to interface
    uci set network.tor=interface
    uci set network.tor.ifname=eth3
    uci set network.tor.proto=static
    uci set network.tor.ipaddr=172.21.13.1
    uci set network.tor.netmask=255.255.255.0

    # configure dhcp
    uci set dhcp.tor=dhcp
    uci set dhcp.tor.interface=tor
    uci set dhcp.tor.start=200
    uci set dhcp.tor.stop=225
    uci set dhcp.tor.limit=25
    uci set dhcp.tor.leasetime=1h

Manually edited /etc/tor/torc
Added to end of file:

    User tor
    PidFile /var/run/tor.pid

    TransPort 9040
    TransListenAddress 172.21.13.1
    DNSPort 9053
    DNSListenAddress 172.21.13.1

TOR firewall zone configuration:

    uci set firewall.@zone[2]=zone
    uci set firewall.@zone[2].name=tor
    uci set firewall.@zone[2].network=tor
    uci set firewall.@zone[2].input=ACCEPT
    uci set firewall.@zone[2].output=ACCEPT
    uci set firewall.@zone[2].forward=REJECT
    uci set firewall.@zone[2].conntrack=1


TOR firewall redirection rules:

    uci set firewall.@redirect[0]=redirect
    uci set firewall.@redirect[0].src=tor
    uci set firewall.@redirect[0].target=DNAT
    uci set firewall.@redirect[0].proto=tcp
    uci set firewall.@redirect[0].dest_ip=172.21.13.1
    uci set firewall.@redirect[0].dest_port=9040
    uci set firewall.@redirect[0]._name=tor-fw-tcp
    uci set firewall.@redirect[0].dest=wan
    uci set firewall.@redirect[1]=redirect
    uci set firewall.@redirect[1]._name=tor-fw-dns
    uci set firewall.@redirect[1].src=tor
    uci set firewall.@redirect[1].proto=udp
    uci set firewall.@redirect[1].dest_ip=172.21.13.1
    uci set firewall.@redirect[1].dest_port=9053
    uci set firewall.@redirect[1].target=DNAT
    uci set firewall.@redirect[1].dest=wan
    uci set firewall.@redirect[1].src_dport=53
    uci set firewall.@redirect[2]=redirect
    uci set firewall.@redirect[2]._name=tor-fw-icmp
    uci set firewall.@redirect[2].src=tor
    uci set firewall.@redirect[2].proto=ICMP
    uci set firewall.@redirect[2].dest_ip=172.21.13.1
    uci set firewall.@redirect[2].target=DNAT
    uci set firewall.@redirect[2].dest=wan

Set TOR to start automatically:

    # tor won't start until after other applications are up
    # in /etc/init.d/tor make the following modification:
    # s/START=50/START=94/g

    vi /etc/init.d/tor

    /etc/init.d/tor enable

Note: Finish & add python script for managing TOR connections.

## Resources

### OpenWRT Basics

+ [OpenWRT Wiki](http://wiki.openwrt.org/start/)
+ [OpenWRT Documentation](http://wiki.openwrt.org/doc/start)
+ [OpenWRT Downloads](http://downloads.openwrt.org/)
+ [OpenWRT Beginner's Guide](http://wiki.openwrt.org/doc/howto/user.beginner)
+ [OpenWRT CLI](http://wiki.openwrt.org/doc/howto/user.beginner.cli)
+ [OpenWrt – First Login](http://wiki.openwrt.org/doc/howto/firstlogin)
+ [LuCI Essentials](http://wiki.openwrt.org/doc/howto/luci.essentials)
+ [OpenWRT Internal Layout](http://wiki.openwrt.org/doc/techref/internal.layout)

### VirtualBox Integration
+ [How to set up openwrt on Virtual Box for a network lab](http://timogoebel.name/content/2012-04-11/how-set-openwrt-virtual-box-network-lab)
+ [Buildsystem in VirtualBox](http://wiki.openwrt.org/doc/howto/buildvm)

### OpenVPN
+ [OpenVPN Manpage](http://openvpn.net/index.php/open-source/documentation/manuals/65-openvpn-20x-manpage.html)
+ [Sample OpenVPN 2.0 configuration files](http://openvpn.net/index.php/open-source/documentation/howto.html#examples)
+ [Please help using vyprvpn (as openvpn client)](https://code.google.com/p/rt-n56u/issues/detail?id=39)
+ [OpenVPN client setup](https://forum.openwrt.org/viewtopic.php?id=42990)
+ [Openwrt configuration example with 2 OpenVpn Tunnel](http://wiki.openwrt.org/doc/howto/vpn.client.openvpn.tun)
+ [OpenWRT with OpenVPN client connection & StrongVNP Example](http://vpnblog.info/openwrt-openvpn-connection-strongvpn.html)
+ [OpenWRT + OpenVPN + Routed LANs](http://theitdepartment.wordpress.com/2009/06/08/openwrt-openvpn-routed-lans/)

Tor:
+ [HOWTO:Run a transparent TOR proxy on Openwrt][https://forum.openwrt.org/viewtopic.php?id=27354]

### Tor & Python

+ http://stackoverflow.com/questions/5148589/python-urllib-over-tor
+ http://stackoverflow.com/questions/711351/using-urllib-with-tor
+ http://blog.databigbang.com/distributed-scraping-with-multiple-tor-circuits/
+ http://archives.seul.org/or/talk/Sep-2009/msg00080.html
+ http://andrewyeoman.com/2013/01/12/changing-tor-circuit-vidalia/
+ http://www.thesprawl.org/research/tor-control-protocol/
+ http://blog.databigbang.com/running-your-own-anonymous-rotating-proxies/
+ http://blog.databigbang.com/distributed-scraping-with-multiple-tor-circuits/
+ http://stackoverflow.com/questions/6283271/is-it-worth-learning-scrapy
+ https://www.torproject.org/docs/tor-manual.html.en
+ https://stem.torproject.org/api/control.html
+ http://stackoverflow.com/questions/11852999/privoxy-tor-and-curl-using-zend-http
+ http://www.thesprawl.org/research/tor-control-protocol/
+ http://www.thesprawl.org/projects/tor-autocircuit/
+ http://serverfault.com/questions/89114/finding-the-public-ip-address-in-a-shell-script

### Verifying Connectivity

    curl icanhazip.com
    curl ident.me
        curl v4.ident.me
        curl v6.ident.me
    dig @trustworthysource.com +short `hostname`
