---
title: proxy_arp test
date: 2023-04-17 16:05:47
tags:
---
```sh
ip link add veth1 type veth peer name eth1
ip netns add ns1
ip link set eth1 netns ns1
ip netns exec ns1 ip a add 10.1.1.10/24 dev eth1
ip netns exec ns1 ip link set eth1 up
ip netns exec ns1 ip route add 169.254.1.1 dev eth1 scope link
ip netns exec ns1 ip route add default via 169.254.1.1 dev eth1
ip link set veth1 up
ip route add 10.1.1.10 dev veth1 scope link
ip route add 10.1.1.10 via 172.12.1.11 dev eth0
echo 1 > /proc/sys/net/ipv4/conf/veth1/proxy_arp



ip link add veth2 type veth peer name eth2                       
ip netns add ns2                                                 
ip link set eth2 netns ns2                                       
ip netns exec ns2 ip a add 10.2.1.20/24 dev eth2                 
ip netns exec ns2 ip link set eth2 up                            
ip netns exec ns2 ip route add 169.254.1.1 dev eth2 scope link   
ip netns exec ns2 ip route add default via 169.254.1.1 dev eth2  
ip link set veth2 up                                             
ip route add 10.2.1.20 dev veth2 scope link                      
ip route add 10.2.1.20 via 172.12.1.12 dev eth0                  
echo 1 > /proc/sys/net/ipv4/conf/veth2/proxy_arp   
```