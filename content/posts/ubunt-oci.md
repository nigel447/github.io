---
title: "Ubuntu on oci"
date: 2022-08-09T18:08:51+03:00
description: Oracle Cloud Infrastructure Ubuntu Image Iptables
summary:  Ubuntu image on oci with  a secure firewall
draft: false
---
- default settings 
- issues
- update for TLS 
- security 

------
Default settings
----------
The Oracle Cloud Infrastructure **_(oci)_** defaults with the iptable  
```
Chain INPUT 
target  prot opt in out  source      destination         
ACCEPT  all  --  *   *   0.0.0.0/0   0.0.0.0/0     state RELATED,ESTABLISHED
ACCEPT  icmp --  *   *   0.0.0.0/0   0.0.0.0/0           
ACCEPT  all  --  lo  *   0.0.0.0/0   0.0.0.0/0           
ACCEPT  udp  --  *   *   0.0.0.0/0   0.0.0.0/0     udp spt:123
ACCEPT  tcp  --  *   *   0.0.0.0/0   0.0.0.0/0     state NEW tcp dpt:22
REJECT  all  --  *   *   0.0.0.0/0   0.0.0.0/0     reject-with icmp-host-prohibited

Chain FORWARD 
target prot opt in  out  source      destination         
REJECT all  --  *   *    0.0.0.0/0   0.0.0.0/0     reject-with icmp-host-prohibited

Chain OUTPUT 
target            prot opt in   out   source        destination         
InstanceServices  all  --  *    *     0.0.0.0/0     169.254.0.0/16      

Chain InstanceServices (1 references)
target  prot opt in out  source     destination         
ACCEPT  tcp  --  *   *   0.0.0.0/0  169.254.0.2      owner UID match 0 tcp dpt:3260 
ACCEPT  tcp  --  *   *   0.0.0.0/0  169.254.2.0/24   owner UID match 0 tcp dpt:3260 
ACCEPT  tcp  --  *   *   0.0.0.0/0  169.254.4.0/24   owner UID match 0 tcp dpt:3260 
ACCEPT  tcp  --  *   *   0.0.0.0/0  169.254.5.0/24   owner UID match 0 tcp dpt:3260 
ACCEPT  tcp  --  *   *   0.0.0.0/0  169.254.0.2      tcp dpt:80 
ACCEPT  udp  --  *   *   0.0.0.0/0  169.254.169.254  udp dpt:53 
ACCEPT  tcp  --  *   *   0.0.0.0/0  169.254.169.254  tcp dpt:53 
ACCEPT  tcp  --  *   *   0.0.0.0/0  169.254.0.3      owner UID match 0 tcp dpt:80 
ACCEPT  tcp  --  *   *   0.0.0.0/0  169.254.0.4      tcp dpt:80 
ACCEPT  tcp  --  *   *   0.0.0.0/0  169.254.169.254  tcp dpt:80 
ACCEPT  udp  --  *   *   0.0.0.0/0  169.254.169.254  udp dpt:67 
ACCEPT  udp  --  *   *   0.0.0.0/0  169.254.169.254  udp dpt:69 
ACCEPT  udp  --  *   *   0.0.0.0/0  169.254.169.254  udp dpt:123 
REJECT  tcp  --  *   *   0.0.0.0/0  169.254.0.0/16   tcp  reject-with tcp-reset
REJECT  udp  --  *   *   0.0.0.0/0  169.254.0.0/16   udp  reject-with icmp-port-unreachable
```

------
Issues
----------
- ufw is problematic 
- firewalld works
- atnum recomends manually setting INPUT rules 

ubuntu on oci will not work well with ufw, in fact oci have disabled it by default. Firewalld will work however it will delete the **InstanceServices** chain.  

This chain is important as it allows outgoing connections to the iSCSI network endpoints (169.254.0.2:3260, 169.254.2.0/24:3260) that serve the instance's boot and block volumes. Removing these rules is a security issue as it allows non-root users or non-administrators to access the instance’s boot disk volume.

The most secure solution is to manually update the iptables input chain.

> Example  
Opening up the imports for TLS  
```
sudo iptables -I  INPUT 6  -p tcp -m tcp --dport 80 -j ACCEPT
sudo iptables -I  INPUT 7  -p tcp -m tcp --dport 443 -j ACCEPT 
```  

------
Preserve security with the InstanceServices chain
--------------------------------------------------
You may be tempted to flush iptables with  
```
iptables -F
iptables -X
```  
the above command will delete the InstanceServices rules.

------
Summary
----------
By default the ubuntu image is locked down for everything other than ssh on port 22. This is a good thing, however it can cause frustration for users with out experience with iptables. Firewald is a solution but better to work with iptables to keep things simple and secure. For security its important to preserve the rules in InstanceServices chain.
