Don't forget to adapt the cloud-based firewall at your server's hosting provider. 
*bonk*

# quagga_on_RPi
Setting Up Quagga on a Raspberry Pi for IPv4 routing.
Documentation:
- https://wiki.gentoo.org/wiki/Quagga (for gentoo but generally helpful)
- https://www.nongnu.org/quagga/docs/quagga.html (Quagga doc)
- https://wiki.debian.org/JonathanFerguson/Quagga

This is for a Pi (or any other Debian based distro) attached via OpenVPN to a VPN server. OpenVPN setup is documented somewhere else.
OpenVPN is used to have an encrypted tunnel from a local network to the central VPN server.
Think of attaching a branch office to the main location. Now, we may also have multiple branch offices attached to that central main location. This will give us a classic star topology. All done using static routes (set by the OpenVPN server/client). Connectivity between the branch offices is established by going via the central node (main office).
But what if the main office suffers a loss of connectivity? Wouldn't it be nice to have a backup connection between the branch offices? This is where we start using dynamic routing. So we establish additional OpenVPN tunnels between the branch offices. And need to use dynamic routing so we may detect a loss of connectivity to the main office and start using the backup routes. Or we may choose to route normal traffic between the branch offices directly (without having to go via the main office).

# Install
- `sudo apt-get install quagga
- `sudo mkdir -p /var/log/quagga && sudo chown quagga:quagga /var/log/quagga`

# Enable Kernel functionality required for Routing
## Enable IPv4 (and IPv6) Forwarding: 
- `echo "net.ipv4.forward=1" | sudo tee -a /etc/sysctl.conf`
(- `sed 's/#net.ipv6.conf.all.forwarding=1/net.ipv6.conf.all.forwarding=1/g' /etc/sysctl.conf | sudo tee /etc/sysctl.conf`)
(- `echo "net.ipv6.conf.default.forwarding=1" | sudo tee -a /etc/sysctl.conf`)
- `sudo sysctl -p`

# Choose Routing Daemon
## Create config files:
- `sudo touch /etc/quagga/ospfd.conf`
- `sudo touch /etc/quagga/vtysh.conf`
- `sudo touch /etc/quagga/zebra.conf`

## Change Ownership
- `sudo chown quagga:quagga /etc/quagga/ospfd.conf && sudo chmod 640 /etc/quagga/ospfd.conf`
- `sudo chown quagga:quaggavty /etc/quagga/vtysh.conf && sudo chmod 660 /etc/quagga/vtysh.conf`
- `sudo chown quagga:quagga /etc/quagga/zebra.conf && sudo chmod 640 /etc/quagga/zebra.conf`

## Deactivate the ones not to be used:
- `sudo unlink /etc/systemd/system/multi-user.target.wants/bgpd.service` 
- `sudo unlink /etc/systemd/system/multi-user.target.wants/isisd.service`
- `sudo unlink /etc/systemd/system/multi-user.target.wants/ospf6d.service`
- `sudo unlink /etc/systemd/system/multi-user.target.wants/pimd.service`
- `sudo unlink /etc/systemd/system/multi-user.target.wants/ripd.service`
- `sudo unlink /etc/systemd/system/multi-user.target.wants/ripngd.service`

# Set Up Zebra:
- Edit config file: `sudo nano /etc/quagga/zebra.conf`
```
!
hostname <hostname>
password <choose one>
!enable password is for commands that may be used to change the config
enable password <again, choose one>
!
interface eth0
!one tunnel to the main office
interface tun0
!
! log to dedicated log file
log file /var/log/quagga/zebra.log informational  
! enable vty config mode
line vty              
```
- re-start zebra daemon: `sudo systemctl restart zebra`
- check access: `telnet localhost 2601` (command prompt pops up (if correct password was supplied))

# Set Up OSPFD
- Edit config file: `sudo nano /etc/quagga/ospfd.conf`
```
!
hostname <to your liking>
password <choose one>
enable password <choose another one>
log file /var/log/quagga/ospfd.log
line vty

router ospf
 ospf router-id <e.g. ip>
 !the RPI has multiple interfaces we don't want to run 
 !the routing protocol on 
 !we only use our vpn tunnel(s) for routing
 passive-interface eth0
 passive-interface wlan0
 passive-interface lo
 !the purpose of dynamic routing.
 !let others know about the networks we're attached to
 !the ones we learned via ospf
 redistribute connected
 !the ones we've set
 redistribute static
 !kernel routes
 redistribute kernel
 !we'll be doing per-interface configuration
 !instead of per-network configuration
 interface ipsec1
  description tunnel to vpn server
  !make it belong to the backbone
  ip ospf area 0.0.0.0
  !it is a point-to-point connection (between us and
  !the main office location VPN server) No broadcast network.
  ip ospf network point-to-point
 interface tun0
  ip ospf network point-to-point
  ! make it belong to the backbone
  ip ospf area 0.0.0.0
```

# Activate
- `sudo systemctl restart zebra`
- `sudo systemctl restart ospfd`

# Manually with vtysh
Thank you, NVIDIA: https://docs.nvidia.com/networking-ethernet-software/cumulus-linux-40/Layer-3/Open-Shortest-Path-First-OSPF/
## add interface to ospf (activate ospf on interface)
- `sudo vtysh`
- open terminal `configure terminal` 
- choose the process that is to be configured: `router ospf`
- go to the interface: `interface tun0`
- add interface to area: `ip ospf area 0.0.0.0`

## tell ospf which routes are to be distributed
- `sudo vtysh`
- `congfigure terminal`
- `router ospf`
- `redistribute kernel
- `redistribute static`

# troubleshooting
- check for hello packets sent/received on any interface: `sudo tcpdump -i any proto ospf` (Hello packets?)
- check multicast group membership of interfaces: `netstat -g` (must be member of ospf-all.mcast.net)
- turn up log info: `log <file name> debugging`
- `sudo vtysh`, `configure terminal`, `debug ospf event`, take a look at the log

 Packet from <ip>] received on link tun0 but no ospf_interface                      2022/01/12 19:28:29 OSPF: make_hello: options: 2, int: tun0:<another ip>    
 
 so I guess this one (in my case) is about a bad netmask.
