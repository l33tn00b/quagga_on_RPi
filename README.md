Don't forget to adapt the cloud-based firewall at your server's hosting provider.


# quagga_on_RPi
Setting Up Quagga on a Raspberry Pi
Documentation:
- https://wiki.gentoo.org/wiki/Quagga (for gentoo but generally helpful)
- https://www.nongnu.org/quagga/docs/quagga.html
- https://wiki.debian.org/JonathanFerguson/Quagga
- 

# Install
- `sudo apt-get install quagga
- `sudo mkdir -p /var/log/quagga && sudo chown quagga:quagga /var/log/quagga`

# Enable Kernel functionality required for Routing
## Enable IPv4 and IPv6 Forwarding: 
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
enable password <again, choose one>
!
interface eth0
!
interface tun0
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
```

# Manually with vtysh
Thank you, NVIDIA: https://docs.nvidia.com/networking-ethernet-software/cumulus-linux-40/Layer-3/Open-Shortest-Path-First-OSPF/
- `sudo vtysh`
- `configure terminal' 
- `router ospf`
- `ìnterface tun0`
- `ip ospf area 0.0.0.0`
