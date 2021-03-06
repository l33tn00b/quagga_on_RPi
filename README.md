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

There's a shitload of people out on the internet going like "ospf will not work over a tun interface of OpenVPN because of multicast. You must use a tap interface." Well, these people are wrong. Go tell 'em to f-ck off and go back to school or get another job not related to networking.

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

## Deactivated one too many?
- `sudo ln -st /etc/systemd/system/multi-user.target.wants /lib/systemd/system/ospfd.service`

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

## exclude routes from distribution (e.g. when having a public internet address on eth0)
- why? because it might be distributed via other interfaces
- thus causing confusion
- we only want a single route (via the public ip)
- if that one is unavailable, well so be it.
- put an access list in your ospfd.conf
  ```
  ! make sure, the access list is _not_ in the router ospf part
  ! watch the level of indentation!!!
  access-list noeth0 deny 157.90.112.171/32
  access-list noeth0 permit any
  ```
- reference that access list _IN the OSPF ROUTER PART_!
  ```
  distribute-list noeth0 out kernel
  distribute-list noeth0 out static
  distribute-list noeth0 out connected
  ```

# troubleshooting
- check for hello packets sent/received on any interface: `sudo tcpdump -i any proto ospf` (Hello packets?)
- check multicast group membership of interfaces: `netstat -g` (must be member of ospf-all.mcast.net)
- turn up log info: `log <file name> debugging`
- `sudo vtysh`, `configure terminal`, `debug ospf event`, take a look at the log

 - got this one? 
   ```
   Packet from <ip>] received on link tun0 but no ospf_interface                      
    2022/01/12 19:28:29 OSPF: make_hello: options: 2, int: tun0:<another ip>
   ```
    It is about an incorrect net mask having been set on the interface the hello packet was received. The daemon is having trouble matching the source of the packet to the interfaces defined in its (the daemon's) config. Found out the hard way: ripd will throw an error along the lines of `2022/01/12 21:32:25 RIP: Neighbor 10.8.0.1 doesn't have connected interface!`, interface matching is done using `if_lookup_address` (see https://github.com/Quagga/quagga/blob/master/lib/if.c) in https://github.com/Quagga/quagga/blob/master/ripd/ripd.c. if_lookup_address does prefix matching. So here we go... fix the net mask and everything is fine. 
 - packets get sent but no packets received (i.e. another router sees hello packets from the one not receiving but hello packets show up in a `tcpdump -i any proto ospf`)? 
  - Does it listen on the correct interface(s)? Enabled? Disabled? (ospfd.conf)
  - Did it subscribe to the multicast group? (ospf-all.mcast.net)
  - Does it receive the packets (strace -p <pid of ospfd> 2>&1 | grep msg)
  - the process doesn't receive the hello packets but they show up in the tcpdump and it is subscribed to the multicast group? DID YOU OPEN THE FIREWALL, Idiot? (yes, that's me..) Check the firewall log... `ufw allow in to 224.0.0.0/24 `
- `Packet <another router's ip> [Hello:RECV]: NetworkMask mismatch on eth0:<our interface's ip> (configured prefix length is 25, but hello packet indicates 0).`
  - our interface's (eth0) ip has a netmask of /25 (class c network is subnetted to make two smaller networks). (no worries here)
  - but the main point is the type of network configured for ospf: `show ip ospf interfaces`, take a close look at the network type (shown right after the router id). These must match on both ends.
