This doc will describe how to setup an IPv6 tunnel over IPv4 on Azure VM network environment. </br>
We setup two VM on Azure different region. One is at East Asia region, the other is at US West region.</br>
Those two VM only have IPv4 public IP, East Asis VM is 168.63.201.67, US West VM is 52.183.45.102.</br>
We will simulate two IPv6 network on each VM and setup IPv6 over IPv4 VxLAN tunnel to make those IPv6 network reaching to each other.</br>

## Simulate IPv6 interface on each VM. We setup loopback interface and assign IPv6 address to each interface. 
East Asia VM setup
```
ip link add lo1 type dummy
ip -6 addr add 2001:db8:2::1/64 dev lo1
ip link set up dev lo1
```
US West VM setup
```
ip link add lo1 type dummy
ip -6 addr add 2001:db8:3::1/64 dev lo1
ip link set up dev lo1
```
## Setup VxLAN interface on each VM.
East Asia VM setup
```
ip link add vx0 type vxlan id 100 remote 52.183.45.102 dev eth0 dstport 4789
ip link set up dev vx0
ip addr add 192.168.200.2/24 dev vx0
ip -6 addr add 2001:db8:1::2/64 dev vx0
```
Result will show as, East Asia VxLAN interface vx0 have MAC address fa:c2:5f:15:51:0d, this is important for next step.
```
3: vx0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether fa:c2:5f:15:51:0d brd ff:ff:ff:ff:ff:ff
    inet 192.168.200.2/24 scope global vx0
       valid_lft forever preferred_lft forever
    inet6 2001:db8:1::2/64 scope global
       valid_lft forever preferred_lft forever
    inet6 fe80::f8c2:5fff:fe15:510d/64 scope link
       valid_lft forever preferred_lft forever
```
US West VM setup
```
ip link add vx0 type vxlan id 100 remote 168.63.201.67 dev eth0 dstport 4789
ip link set up dev vx0
ip addr add 192.168.200.3/24 dev vx0
ip -6 addr add 2001:db8:1::3/64 dev vx0
```
Result will show as, US West VxLAN interface vx0 have MAC address 5e:5a:9a:30:ab:27, this is important for next step.
```
3: vx0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether 5e:5a:9a:30:ab:27 brd ff:ff:ff:ff:ff:ff
    inet 192.168.200.3/24 scope global vx0
       valid_lft forever preferred_lft forever
    inet6 2001:db8:1::3/64 scope global
       valid_lft forever preferred_lft forever
    inet6 fe80::5c5a:9aff:fe30:ab27/64 scope link
       valid_lft forever preferred_lft forever
```
## Setup IPv6 route on each VM
East Asia VM setup
```
ip -6 route add 2001:db8:3::/64 dev vx0
```
Result will show as:
```
ip -6 route
::1 dev lo proto kernel metric 256 pref medium
2001:db8:1::/64 dev vx0 proto kernel metric 256 pref medium
2001:db8:2::/64 dev lo1 proto kernel metric 256 pref medium
2001:db8:3::/64 dev vx0 metric 1024 pref medium
fe80::/64 dev eth0 proto kernel metric 256 pref medium
fe80::/64 dev vx0 proto kernel metric 256 pref medium
fe80::/64 dev lo1 proto kernel metric 256 pref medium
```
US West VM setup
```
ip -6 route add 2001:db8:2::/64 dev vx0
```
Reslut will show as:
```
ip -6 route
::1 dev lo proto kernel metric 256 pref medium
2001:db8:1::/64 dev vx0 proto kernel metric 256 pref medium
2001:db8:2::/64 dev vx0 metric 1024 pref medium
2001:db8:3::/64 dev lo1 proto kernel metric 256 pref medium
fe80::/64 dev eth0 proto kernel metric 256 pref medium
fe80::/64 dev vx0 proto kernel metric 256 pref medium
fe80::/64 dev lo1 proto kernel metric 256 pref medium
```
## Setup IPv6 link level address information

East Asia VM setup, setup destination IPv6 and link layer address with US West VxLAN interface MAC address.
```
ip -6 neigh add 2001:db8:3::1 lladdr 5e:5a:9a:30:ab:27 dev vx0
```
Reslut will show as:
```
ip -6 neigh show
2001:db8:3::1 dev vx0 lladdr 5e:5a:9a:30:ab:27 PERMANENT
2001:db8:1::3 dev vx0 lladdr 5e:5a:9a:30:ab:27 STALE
```
US West VM setup,setup destination IPv6 and link layer address with East Asia VxLAN interface MAC address.
```
ip -6 neigh add 2001:db8:2::1 lladdr fa:c2:5f:15:51:0d dev vx0
```
Reslut will show as:
```
ip -6 neigh show
2001:db8:1::2 dev vx0 lladdr fa:c2:5f:15:51:0d STALE
2001:db8:2::1 dev vx0 lladdr fa:c2:5f:15:51:0d PERMANENT
```

