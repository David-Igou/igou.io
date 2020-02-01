---
layout: post
title: "Secure Networking between local hosts and an AWS VPC"
date: 2020-01-07
categories: [openvpn, aws, homelab, vpc, firewall, raspberry pi, networking]
---

Being able to route to your AWS VPC hosts from your home network opens many doors for projects. AWS Offers a VPN Gateway service, but that costs a minimum of ~90 a month. For small labs, we can easily do this with a t2.nano.

There are numerous benefits to this setup such as lower costs by using your own hardware, and less management of outside access via security groups.

These sorts of setups are rightfully intimidating. Having any kind of entrypoint into your home network can be scary when you consider worst case scenarios. It is important to have a good background with networking, network segmentation, and incident response when designing something like this. Numerous times while I did this I send my configurations and network diagrams to people who are far more experienced than me on these subjects to validate.

![Network Topology](/images/aws-vpn-gateway.png)

# OpenVPN

This uses resources from the [OpenVPN how-to article](https://openvpn.net/community-resources/how-to/)

[I also referrenced this article on a similar setup.](http://blog.ignacio.org.mx/posts/raspberry+openvn+aws+vpc/)

## Server Config

Part of this process is generating TLS certs so your client can securely authenticate to your server. To generate certificates, [I used this guide from OpenVPN Community](https://openvpn.net/community-resources/setting-up-your-own-certificate-authority-ca/)


```bash
apt-get update
apt-get install openvpn
apt-get install easy-rsa
cp -r /usr/share/easy-rsa/* /etc/openvpn/easy-rsa/
# Follow Instructions to generate certificates
cp server.crt server.key ca.crt dh2048.pem /etc/opencpn/

./build-key client1
cp client1.crt /home/ubuntu
cp client1.key /home/ubuntu
[scp to client]

apt-get install iputils-ping
```

Update your servers config file in /etc/openvpn/server.conf

```conf
# Which TCP/UDP port should OpenVPN listen on?
# If you want to run multiple OpenVPN instances
# on the same machine, use a different port
# number for each one.  You will need to
# open up this port on your firewall.
port 1194

# TCP or UDP server?
proto udp

# "dev tun" will create a routed IP tunnel,
# "dev tap" will create an ethernet tunnel.
# Use "dev tap0" if you are ethernet bridging
# and have precreated a tap0 virtual interface
# and bridged it with your ethernet interface.
# If you want to control access policies
# over the VPN, you must create firewall
# rules for the the TUN/TAP interface.
# On non-Windows systems, you can give
# an explicit unit number, such as tun0.
# On Windows, use "dev-node" for this.
# On most systems, the VPN will not function
# unless you partially or fully disable
# the firewall for the TUN/TAP interface.
dev tun

# SSL/TLS root certificate (ca), certificate
# (cert), and private key (key).  Each client
# and the server must have their own cert and
# key file.  The server and all clients will
# use the same ca file.
#
# See the "easy-rsa" directory for a series
# of scripts for generating RSA certificates
# and private keys.  Remember to use
# a unique Common Name for the server
# and each of the client certificates.
#
# Any X509 key management system can be used.
# OpenVPN can also use a PKCS #12 formatted key file
# (see "pkcs12" directive in man page).
ca ca.crt
cert server.crt
key server.key  # This file should be kept secret

# Diffie hellman parameters.
# Generate your own with:
#   openssl dhparam -out dh2048.pem 2048
dh dh2048.pem

# Network topology
# Should be subnet (addressing via IP)
# unless Windows clients v2.0.9 and lower have to
# be supported (then net30, i.e. a /30 per client)
# Defaults to net30 (not recommended)
topology subnet

# Configure server mode and supply a VPN subnet
# for OpenVPN to draw client addresses from.
# The server will take 10.8.0.1 for itself,
# the rest will be made available to clients.
# Each client will be able to reach the server
# on 10.8.0.1. Comment this line out if you are
# ethernet bridging. See the man page for more info.
server 10.1.2.0 255.255.255.0
push "route 10.1.1.0 255.255.255.0"


# Maintain a record of client <-> virtual IP address
# associations in this file.  If OpenVPN goes down or
# is restarted, reconnecting clients can be assigned
# the same virtual IP address from the pool that was
# previously assigned.
ifconfig-pool-persist ipp.txt

# The keepalive directive causes ping-like
# messages to be sent back and forth over
# the link so that each side knows when
# the other side has gone down.
# Ping every 10 seconds, assume that remote
# peer is down if no ping received during
# a 120 second time period.
keepalive 10 120


# Select a cryptographic cipher.
# This config item must be copied to
# the client config file as well.
# Note that v2.4 client/server will automatically
# negotiate AES-256-GCM in TLS mode.
# See also the ncp-cipher option in the manpage
cipher AES-256-CBC

# The persist options will try to avoid
# accessing certain resources on restart
# that may no longer be accessible because
# of the privilege downgrade.
persist-key
persist-tun

# Output a short status file showing
# current connections, truncated
# and rewritten every minute.
status openvpn-status.log

# Set the appropriate level of log
# file verbosity.
#
# 0 is silent, except for fatal errors
# 4 is reasonable for general usage
# 5 and 6 can help to debug connection problems
# 9 is extremely verbose
verb 6

# Silence repeating messages.  At most 20
# sequential messages of the same message
# category will be output to the log.
;mute 20

# Notify the client that when the server restarts so it
# can automatically reconnect.
explicit-exit-notify 1

#Additional routing config
client-config-dir ccd
route 192.168.1.128 255.255.255.128

client-to-client
push "route 192.168.1.128 255.255.255.128"
```

Make the contents of `ccd/client`

```
iroute 192.168.1.128 255.255.255.128
```


## Important Routing Configurations to consider


```conf
# Configure server mode and supply a VPN subnet
# for OpenVPN to draw client addresses from.
# The server will take 10.8.0.1 for itself,
# the rest will be made available to clients.
# Each client will be able to reach the server
# on 10.8.0.1. Comment this line out if you are
# ethernet bridging. See the man page for more info.
server 10.1.2.0 255.255.255.0
push "route 10.1.1.0 255.255.255.0"


#Additional routing config
client-config-dir ccd
route 192.168.1.128 255.255.255.128

client-to-client
push "route 192.168.1.128 255.255.255.128"
```

With this setup, my home network can route to everything in my VPC Subnet, but only my additional K8s nodes can be routed to directly from the VPC.

Once configured, you can start the server with `systemctl enable --now openvpn@server`


## Client configuration


Copy over your client1.crt/key's and install openvpn:
```conf
# Specify that we are a client and that we
# will be pulling certain config file directives
# from the server.
client

# Use the same setting as you are using on
# the server.
# On most systems, the VPN will not function
# unless you partially or fully disable
# the firewall for the TUN/TAP interface.
dev tun

# Are we connecting to a TCP or
# UDP server?  Use the same setting as
# on the server.
;proto tcp
proto udp

# The hostname/IP and port of the server.
# You can have multiple remote entries
# to load balance between the servers.

# This value is an ElasticIP I assigned to the ec2 instance
remote 13.37.13.37 1194

# Keep trying indefinitely to resolve the
# host name of the OpenVPN server.  Very useful
# on machines which are not permanently connected
# to the internet such as laptops.
resolv-retry infinite

# Most clients don't need to bind to
# Most clients don't need to bind to
# a specific local port number.
nobind

# Try to preserve some state across restarts.
persist-key
persist-tun

# SSL/TLS parms.
# See the server config file for more
# description.  It's best to use
# a separate .crt/.key file pair
# for each client.  A single ca
# file can be used for all clients.
ca ca.crt
cert client1.crt
key client1.key

# Verify server certificate by checking that the
# certificate has the correct key usage set.
# This is an important precaution to protect against
# a potential attack discussed here:
#  http://openvpn.net/howto.html#mitm
#
# To use this feature, you will need to generate
# your server certificates with the keyUsage set to
#   digitalSignature, keyEncipherment
# and the extendedKeyUsage to
#   serverAuth
# EasyRSA can do this for you.
remote-cert-tls server

# Select a cryptographic cipher.
# If the cipher option is used on the server
# then you must also specify it here.
# Note that v2.4 client/server will automatically
# negotiate AES-256-GCM in TLS mode.
# See also the ncp-cipher option in the manpage
cipher AES-256-CBC

# Set log file verbosity.
verb 6

# Silence repeating messages
;mute 20
```

Start the service with `systemctl enable --now openvpn@client`

# Verifying VPN Connectivity

Let's make sure all the OpenVPN stuff is working before we configure this node as a gateway, so if we run into problems down the road we'll know it's not a "vpn problem"

On the Client:

```shell
root@raspberrypi:~# ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.1.3  netmask 255.255.255.0  broadcast 192.168.1.255
        inet6 fe80::ba27:ebff:fe20:5177  prefixlen 64  scopeid 0x20<link>
        ether b8:27:eb:20:51:77  txqueuelen 1000  (Ethernet)
        RX packets 60422534  bytes 30955754260 (28.8 GiB)
        RX errors 0  dropped 275  overruns 0  frame 0
        TX packets 55687632  bytes 32175457247 (29.9 GiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 27  bytes 2036 (1.9 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 27  bytes 2036 (1.9 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

tun0: flags=4305<UP,POINTOPOINT,RUNNING,NOARP,MULTICAST>  mtu 1500
        inet 10.1.2.2  netmask 255.255.255.0  destination 10.1.2.2
        inet6 fe80::1c9d:1ef2:13a3:b711  prefixlen 64  scopeid 0x20<link>
        unspec 00-00-00-00-00-00-00-00-00-00-00-00-00-00-00-00  txqueuelen 100  (UNSPEC)
        RX packets 28292997  bytes 25673283699 (23.9 GiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 26909574  bytes 3747063994 (3.4 GiB)
        TX errors 0  dropped 98 overruns 0  carrier 0  collisions 0
```

On the Server:

```shell
ubuntu@ip-10-1-1-2:~$ ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 9001
        inet 10.1.1.2  netmask 255.255.255.0  broadcast 10.1.1.255
        inet6 fe80::e:d4ff:fe76:3b90  prefixlen 64  scopeid 0x20<link>
        ether 02:0e:d2:a6:3b:90  txqueuelen 1000  (Ethernet)
        RX packets 55549787  bytes 30995333263 (30.9 GB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 60257649  bytes 31868147915 (31.8 GB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 945  bytes 90478 (90.4 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 945  bytes 90478 (90.4 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

tun0: flags=4305<UP,POINTOPOINT,RUNNING,NOARP,MULTICAST>  mtu 1500
        inet 10.1.2.1  netmask 255.255.255.0  destination 10.1.2.1
        inet6 fe80::62fa:edb:5e23:49ad  prefixlen 64  scopeid 0x20<link>
        unspec 00-00-00-00-00-00-00-00-00-00-00-00-00-00-00-00  txqueuelen 100  (UNSPEC)
        RX packets 26899124  bytes 3743687524 (3.7 GB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 28367916  bytes 25785202309 (25.7 GB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

Ping the Server's tun0:

```shell
root@raspberrypi:~# ping 10.1.2.1
PING 10.1.2.1 (10.1.2.1) 56(84) bytes of data.
64 bytes from 10.1.2.1: icmp_seq=1 ttl=64 time=37.9 ms
64 bytes from 10.1.2.1: icmp_seq=2 ttl=64 time=30.3 ms
64 bytes from 10.1.2.1: icmp_seq=3 ttl=64 time=33.5 ms
```

Ping the Server's eth0:

```shell
root@raspberrypi:~# ping 10.1.1.2
PING 10.1.1.2 (10.1.1.2) 56(84) bytes of data.
64 bytes from 10.1.1.2: icmp_seq=1 ttl=64 time=29.4 ms
64 bytes from 10.1.1.2: icmp_seq=2 ttl=64 time=29.8 ms
64 bytes from 10.1.1.2: icmp_seq=3 ttl=64 time=29.5 ms
```

# Configuring Network Forwarding and Routing

Now we need to configure both ends to forward traffic. Your home network needs to be configured to route traffic to AWS through the gateway, and the same for your AWS setup. 

## Configure a static IP for the client


Edit `/etc/network/interfaces` to add your static IP

```shell
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet static
    address 192.168.1.3
    netmask 255.255.255.0
    gateway 192.168.1.1
    dns-nameservers 8.8.8.8 8.8.4.4
    post-up iptables-restore < /etc/iptables/rules.v4
```

## IPv4 Forwarding and iptables

Enabled ip forwarding via `echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf`


Add IPTables rules and make them persistent:

```shell
# Flush the entire iptables
iptables -F
iptables -X
iptables -t nat -F
iptables -t nat -X
iptables -t mangle -F
iptables -t mangle -X
iptables -P INPUT ACCEPT
iptables -P FORWARD ACCEPT
iptables -P OUTPUT ACCEPT

# Allow established connections
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A FORWARD -m state --state ESTABLISHED,RELATED -j ACCEPT

# Masquerade the traffic
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
iptables -t nat -A POSTROUTING -o tun0 -j MASQUERADE

# Save the state to the file the interface looks for
iptables-save > /etc/iptables/rules.v4
```


## Adding routes for the rest of your network to use

Both sides will need to know how to route to the other. For example, in my home router I added the route "10.0.0.0/16 via [gatway ip]" and for AWS I added a route "192.168.1.0/24 via [OpenVPN Server instance ID]"

# Security

In its basic form, this setup is very secure between the SSL/TLS auth and a very strict firewall rule to only allow connections from your home IP.

As you add things to your VPC, you will need to be careful about what you expose - If something in your VPC gets compromised, there are a set of nodes in your home network they can now route to. Make sure those nodes are properly secured, in the event something bad does happen. You can also consider things like ACLs to prevent further pivoting from the 192.168.1.128/25 subnet.

Alternatively, you can simply remove ALL VPC -> Home routing from your configs, so nothing in AWS can initiate connections to the home network, but vise versa still works..

Security is best approached with the mentality that everything outside facing will be compromised eventually. Truly secure networks prevent intruders from pivoting deeper.

# Performance

Performance is not "Great" but for POC's I do while working on the road I'm pretty happy with.

The baseline performance for a t2.nano is 30 Megabit/s. That's 30% of my home internet bandwidth, which actually works out pretty well to stop me from having this setup suck up all of my network bandwidth.

# Possible Improvements

In the event my IP changes, I would have to update it manually - this could easily be resolved via a cronjob to check and update via the aws cli.

More in depth monitoring of this should be done to really get an understanding of the performance. That'll probably be a future blog, but until then cloudwatch metrics will do.
