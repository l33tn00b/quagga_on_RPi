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
## Enable IPv4 and IPv6 Unicast Forwarding: 
- `echo "net.ipv4.conf.all.forwarding=1" | sudo tee -a /etc/sysctl.conf`
- `echo "net.ipv4.conf.default.forwarding=1" | sudo tee -a /etc/sysctl.conf`
- `sed 's/#net.ipv6.conf.all.forwarding=1/net.ipv6.conf.all.forwarding=1/g' /etc/sysctl.conf | sudo tee /etc/sysctl.conf`
- `echo "net.ipv6.conf.default.forwarding=1" | sudo tee -a /etc/sysctl.conf`
- `sudo sysctl -p`

## Enable IPv4 Multicast Forwarding:
- `echo "net.ipv4.conf.all.mc_forwarding=1" | sudo tee -a /etc/sysctl.conf`
- `echo "net.ipv4.conf.default.mc_forwarding=1" | sudo tee -a /etc/sysctl.conf`
- `sudo sysctl -p`

# Choose Routing Daemon
## Create config files:
- `sudo touch /etc/quagga/bgpd.conf`
- `sudo touch /etc/quagga/ospfd.conf`
- `sudo touch /etc/quagga/vtysh.conf`
- `sudo touch /etc/quagga/zebra.conf`

## Change Ownership
- `sudo chown quagga:quagga /etc/quagga/ospfd.conf && sudo chmod 640 /etc/quagga/ospfd.conf`
- `sudo chown quagga:quaggavty /etc/quagga/vtysh.conf && sudo chmod 660 /etc/quagga/vtysh.conf`
- `sudo chown quagga:quagga /etc/quagga/zebra.conf && sudo chmod 640 /etc/quagga/zebra.conf`

## Deactivate the ones not to be used:
- `sudo unlink /etc/systemd/system/multi-user.target.wants/ospfd.service`
- `sudo unlink /etc/systemd/system/multi-user.target.wants/zebra.service`
