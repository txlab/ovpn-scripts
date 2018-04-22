# Routed VPN

This is a typical scenario for OpenVPN deployment with a script that
automates the generation of client configuration.

The resulting VPN allows clients communicating between each other. Also
a client may route some traffic toward the VPN tunnel, and it would be
forwarded to the public Internet using NAT on the server.

You need to create a valid DNS name for your VPN server. For example,
http://freedns.afraid.org/ allows creatig up to 5 DNS names for free.

## Server setup

Copy-paste the following commands into your terminal under root
user. They will work without changes on Debian 9 or 8 or Ubuntu), and
will probably require small adaptations for other OSes.


```
# install required software packages
apt-get upgrade && apt-get install -y iptables-persistent \
openvpn easy-rsa git

# enable IP routing
cat >/etc/sysctl.d/local.conf <<'EOT'
net.ipv4.ip_forward=1
net.ipv6.conf.all.forwarding=1
EOT

# find our Internet facing interface
OUTDEV=`ip route | awk '/^default/{print $5}'`

# Firewall rules for IPv4: NAT outbound traffic.

iptables -t nat -F POSTROUTING
iptables -t nat -A POSTROUTING -o $OUTDEV -j MASQUERADE
iptables-save >/etc/iptables/rules.v4

reboot
```

```
# here we login again

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

# write server parameters for config generation scripts
# note that you need to specify your own valid FQDN for the server

cat >/etc/openvpn/ovpn_routed_params <<'EOT'
SERVER_FQDN=myvpnserver.example.com
SERVER_PORT=1194
SERVER_IPNET=10.251.0.0
SERVER_IPMASK=255.255.255.0
EOT

# generate the server configuration
/root/ovpn-scripts/gen_conf_ovpn_routed_server

# generate the client configuration
/root/ovpn-scripts/gen_conf_ovpn_routed_client mylaptop.myvpn
```

The client configuration script creates a `*.conf` file in
`/etc/openvpn/client_configs'. You need to copy it to your VPN client
device. Note that Linux clients are looking for `*.conf` files in their
`/etc/openvpn` directory, while Windows and Android clients are looking
for `*.ovpn` files, so you need to rename the configuration file
accordingly.


