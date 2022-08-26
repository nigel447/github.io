---
title: "Ubuntu Wireguard Oci"
date: 2022-08-16T00:34:28+03:00
description: Oracle Cloud Infrastructure Ubuntu Wireguard
summary:  Ubuntu image on oci with Wireguard
draft: false
---
- wireguard concepts 
- virtual network interface
- virtual network tunnel
- configuration
- firewall
------
concepts
----------
For clarity this post is restricted to IP4 address. FW is equivalent to iptable. It is assumed port forwarding ```/etc/sysctl.conf``` is enabled for IP4 and port 51820 open when needed.

At first it can be useful to think of wireguard both as a virtual network interface VNI and a virtual network (VN). This VN uses the [private CIDR blocks](https://en.wikipedia.org/wiki/Private_network#Private_IPv4_addresses), for this discussion we use the 10.X.X.X/24 for a single class A network.  
For a simple VPN between your box(node) and a remote peer(server) everything is connected to the VN via a VNI we call `wgo`. The VN is a peer network, hence its nodes can connect / exchange traffic. This is not ideal and a security risk. To mitigate we need to isolate the clients, this is just a single FW rule in the FW FORWARD chain.
```
iptables -I FORWARD -i wg0 -o wg0 -j REJECT --reject-with icmp-adm-prohibited
```
virtual network interface (wg0)
----------
Consider your box(node) is a peer connected to the VN via the wg0 VNI. Consider a VN of just 2 peers, peer X a remote VM and peer Y a local client.  
Y has config settings for its Y wg0, and config settings for its connection to the remote X wg0. This is symmetric for X. The configuration files for wgo allows for the setup of the WireGuard encrypted UDP tunnel. This tunnel has the following attributes. 
- Endpoint:  a physical socket for a remote peer.
- AllowedIPs: CIDR block(s) to control how wgo will route to the the WireGuard tunnel.
- Address:  virtual IP address of wg0 for the VN.
tunneling
----------
The Address attribute for wg0 is a virtual ip in the CIDR range of the VN, for the following discussion we will create a VN as shown below  
![wireguard Virtual Network](/image/wgVN.png)
Address Y = 10.0.2.3/24, here the Y wg0 virtual ip is 10.0.2.3 and packets routed through the Y wg0 Endpoint will be a source IP 10.0.2.3 to X, symmetrically packets received from the VN with destination 10.0.2.3 will be routed by the Y wg0.  

Y has a physical ip (YIP), if we ping X (10.0.2.1) from Y wg0 will generate packets with source 10.0.2.3 and destination 10.0.2.1, wgo then encrypts and wraps them in UDP with source YIP and destination Endpoint IP with the default wireguard port of 51820. X listening on Endpoint unwraps and decrypts them onto their wgo VN source 10.0.2.3 and destination 10.0.2.1.   

Symmetrically X receives the ping response as UDP from Endpoint, then wg0 unwraps and decrypts to ICMP source address of 10.0.2.1 with destination address of 10.0.2.3.

config
----------
On ubuntu installing wireguard ```sudo apt install net-tools wireguard``` along with initial setup(keys and ports) is well documented. The focus here is on the VN configuration, concepts and principles,

------
Remote peer X (server)
----------
#### config file 
```
sudo nano /etc/wireguard/wg0.conf
```
```
[Interface]
Address = 10.0.2.1/24
SaveConfig = true
PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -t nat -A POSTROUTING -o ens3 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -t nat -D POSTROUTING -o ens3 -j MASQUERADE
ListenPort = 51820
PrivateKey = XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX

[Peer]
PublicKey = YYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYY
AllowedIPs = 10.0.2.0/24
Endpoint = YIP:51820
```
#### Explanation
PostUp and PostDown are FW rules appended and deleted when we create or destory the VNI wg0  
We can see the Endpoint socket for the peer Y.  
The PrivateKey on X is used for packet decryption for incoming data encrypted with X's public key shared with Y, The PublicKey of Y is shared with X and becomes part of the connection configuration for X. X will encrypt data going out with the public key of Y then Y decrypts that data with its private key. 

Peer Y
----------

#### config file 
```
sudo nano /etc/wireguard/wg0.conf
```
```
[Interface]
PrivateKey = YYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYY
Address = 10.0.2.3/24

[Peer]
PublicKey = XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXZ
Endpoint = XIP:51820
AllowedIPs = 10.0.2.1/24
```
#### Explanation
As for the configuration for the node X we see the symmetric configuration. The Endpoint socket for X, and VCN ip addresses.  
A good question here is why do we not have the PostUp and PostDown are FW rules?  In this example X is inside an oci virtual cloud network VCN. The PostUp/PostDown rules allow an internal socket on X behind wgo for nat routing to the VCN. We dont need this for a desktop linux client.

firewall
----------
In the atnum blog post "Ubuntu on oci" we go through the FW issues, for this config the VN interfaces (wgo) will not be able to connect to the VN on the server (X) as it will be blocked by the default FW rules. We need to open the wireguard UDP port 51280.
```
sudo iptables -I  INPUT 8  -p udp -m udp --dport 51820 -m state --state NEW -j ACCEPT
```  
with this input rule the VN 10.0.2.0/24 will be accessible to the virtual network interface wgo, we can test this by pinging the server (X) from the client (Y) via the wireguard udp tunnel.
```
$ ping 10.0.2.1
PING 10.0.2.1 (10.0.2.1) 56(84) bytes of data.
64 bytes from 10.0.2.1: icmp_seq=1 ttl=64 time=98.3 ms
64 bytes from 10.0.2.1: icmp_seq=2 ttl=64 time=48.8 ms
64 bytes from 10.0.2.1: icmp_seq=3 ttl=64 time=49.6 ms
```
summary
----------
To implement this example with Ubuntu on oci you will need port 51820 open in your NSG or equivalent. You will also need the  basic packages to work with iptables,  and the wireguard package itself.  
Wireguard is a simple to implement solution whenever you need a secure virtual network.