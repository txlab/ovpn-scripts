# Anti-censorship VPN for home LAN

As Internet access is being censored in some countries, there is a need
to place a home LAN behind a VPN, so that home users would enjoy the
access without having to tune proxies or VPN on every device.

This is a complete scenario and deployment instructions for such a
solution. It consists of two or more Linux instances: a VPS at some
foreign hosting provider, acting as a VPN concentrator, and a home
gateway.

Some low-cost VPS providers offer OpenVZ hypervisor, and it is not
suitable for using as VPN concentrator. Any of the following hypervizors
will work fine: KVM, Xen, VmWare. This scenario assumes that the VPS has
both public IPv4 and IPv6 addresses.

The home gateway is any Linux computer with two Ethernet interfaces. It
could also be a Raspberry Pi with an additional Ethernet-over-USB
adapter, or any other low-cost device. It only needs to support a
relatively fresh version of OpenVPN, and support Ethernet bridging.

The home gateway is bridging its home LAN port with the OpenVPN tunnel,
and the VPS acts as a router and DHCP server for the home LAN. The VPS
is also taking care of packet fragmentation and setting a lower TCP MSS
in order to avoid fragmentation for TCP traffic.

The scenario allows multiple home LANs to be terminated on a single
VPS. Each home LAN has its own OpenVPN daemon on the server, and routing
between home LANs is prohibited by the firewall on the server.

This design assumes that there is a publicly resolvable DNS name for the
VPS server. It is advised to have a separate DNS name for the VPS for
each VPN that it is hosting. This way it would be easy to move the VPNs
to another hosting if needed.

If the VPS IP address gets blocked for any reason, you can install a UDP
proxy somewhere else and modify the DNS accordingly.

There are some free DNS services that allow you to create your own A or
NS records. For example, http://freedns.afraid.org/ offers up to 5
records for free.

The scenario and scripts are optimized for Debian 9, but can also be
used with any other Linux distribution. You need to have root access to
both the VPS and the home gateway.


## Server setup

Copy-paste the following commands into your terminal under root
user. They will work without changes on Debian 9 (and probably Debian 8
and Ubuntu), and will probably require small adaptations for other OSes.


```
# install required software packages
apt-get upgrade && apt-get install -y iptables-persistent \
openvpn easy-rsa bridge-utils git dnsmasq

# enable IP routing
cat >/etc/sysctl.d/local.conf <<'EOT'
net.ipv4.ip_forward=1
net.ipv6.conf.all.forwarding=1
EOT

# find our Internet facing interface
OUTDEV=`ip route | awk '/^default/{print $5}'`

# Firewall rules for IPv4 and IPv6:
#  masquerading sets the VPS public address as source for all outbound packets
#  set-mss helps avoiding extra fragmentation for TCP sessions
#  the other rule prohibits communication between home LANs

iptables -t nat -F POSTROUTING
iptables -t nat -A POSTROUTING -o $OUTDEV -j MASQUERADE
iptables -F FORWARD
iptables -A FORWARD -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --set-mss 1350
iptables -A FORWARD -i tap+ -o tap+ -j DROP
iptables-save >/etc/iptables/rules.v4

ip6tables -t nat -F POSTROUTING
ip6tables -t nat -A POSTROUTING -o $OUTDEV -j MASQUERADE
ip6tables -F FORWARD
ip6tables -A FORWARD -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --set-mss 1350
ip6tables -A FORWARD -i tap+ -o tap+ -j DROP
ip6tables-save >/etc/iptables/rules.v6

reboot
```

```
# here we login again

OUTDEV=`ip route | awk '/^default/{print $5}'`

# dnsmasq is serving as DHCP and IPv6 configuration server
# we don't want to provide this service on our public interface
cat >/etc/dnsmasq.d/vpnserver <<EOT
except-interface=$OUTDEV
bogus-priv
EOT

# make dnsmasq dependent on openvpn, so that it restarts every time
# openvpn restarts
mkdir /etc/systemd/system/dnsmasq.service.d/
cat >/etc/systemd/system/dnsmasq.service.d/override.conf <<'EOT'
[Unit]
After=openvpn.service
BindsTo=openvpn.service
EOT
systemctl daemon-reload
systemctl reenable dnsmasq

# set up our Certificate Authority for OpenVPN certificates
cd /etc/openvpn
mkdir easy-rsa
cp -R /usr/share/easy-rsa/* easy-rsa/
cd easy-rsa/
ln -s openssl-1.0.0.cnf openssl.cnf
# You may want to edit the "vars" file and set the certificate attributes
. ./vars  
./clean-all
./build-ca
./build-dh

# place the config generation script into root's home
cd /root
git clone https://github.com/txlab/ovpn-scripts.git
```

The config generation script takes four arguments in the following order:

 1. VPN ID: a text string that is identifying this VPN instance from
 others. 0001, 0002, and so on is a good example.

 2. Fully qualified domain name of your VPN server. This needs to be a
 valid DNS name that resolves into the IP address of your VPS. It is
 recommended to have a unique name for each VPN instance, even if they
 point to the same IP address.

 3. Network number, an integer between 0 and 255. The home LAN will use
 the addresses from 172.16.X.0/24. This number has to be unique across
 VPNs terminating on one VPS.

 4. UDP port number for OpenVPN communication. This port should also be
 unique across all VPNs on your VPS. The range of acceptable numbers is
 1000-65535.

In this exampple, I created a DNS name `vpn0001.crabdance.com` uisng
freedns.afraid.org:

```
/root/ovpn-scripts/gen_conf_bridged_home_lan 0001 vpn0001.crabdance.com 1 10001
```

This command will create server and client OpenVPN configurations, and
update dnsmasq to serve as DHCP server for the new home LAN. The client
configuration will need to be copied to the home gateway Linux machine.

Keep in mind that the script restarts all OpenVPN daemons, so all
connections will be interrupted for about a minute.


## Client gateway setup

The following is a sequence of setup commands for a Debian or Ubuntu
machine. Other OSes can be configured accordingly.

In this example, `enp2s0` is the name of an Ethernet NIC in the home
gateway box. Depending on your hardware and OS, it could also be named
as `eth1` or something similar.

```
# required software packages
apt-get update && \
apt-get install -y ethtool openvpn bridge-utils 

# Bridge interface needs to be named 'br1', so that the script works
cat >/etc/network/interfaces.d/br1 <<'EOT'
auto br1
iface br1 inet manual
    bridge_ports enp2s0
    bridge_stp off
    bridge_fd 0
EOT

# This script is executed after the OpenVPN connection is established.
# It adds the VPN tunnel interface to the bridge.
cat >/etc/openvpn/addtobridge_br1 <<'EOT'
#!/bin/sh
dev=$1
mtu=$2
brctl addif br1 $dev
ip link set $dev up promisc on mtu $mtu
EOT
chmod u+x /etc/openvpn/addtobridge_br1

# Now copy the OpenVPN client configuration to /etc/openvpn/
scp root@vpn0001.crabdance.com:/etc/openvpn/client_configs/vpn0001.vpn0001.crabdance.com.conf /etc/openvpn/

# If SSH connection is not available, you can output the configuration
# into the terminal and paste it into the other terminal where your home
# gateway shell is.

# Now let OpenVPN daemon use your configuration and restart
systemctl reenable openvpn
systemctl restart openvpn
```

If you want to manage your home gateway box from the VPS, you can either
configure the `br1` interface with a static IP address from the home LAN
range, or use another OpenVPN instance for management purposes.


