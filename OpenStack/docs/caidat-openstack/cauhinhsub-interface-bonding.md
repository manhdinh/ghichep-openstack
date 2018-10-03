# Tài liệu hướng dẫn cấu hình Sub-interface VLAN trên Bonding

## 1. Sub-interface VLAN là gì?

**Sub-interface** là các interface ảo, được tạo ra bằng cách chia một interface vật lý (physical-interface) thành các logical interface. Mỗi sub-interface sẽ được gán vào một VLAN.

## 2. Hướng dẫn cấu hình sub-interface VLAN trên Linux (RHEL 7)

**Chú ý**. Để cấu hình thành công, cần có các điều kiện sau :

 -	Switch hỗ trợ chuẩn IEEE 802.1q và đã cấu hình trunk VLAN
 -	NIC hoạt động được với Linux và hỗ trợ chuẩn 802.1q
 
### 2.1 Cấu hình trên switch
 - Cấu hình trunk vlan trên switch
 
```sh
Switch#(config)interface Gi3/41
Switch#(config-if)no switchport mode access
Switch#(config-if)switchport mode trunk
Switch#(config-if)switchport trunk allowed vlan 192
```
 
### 2.2 Cấu hình trên server Linux
 
 - Load driver 802.1q Linux kernel
 
`modprobe 8021q`

 - Kiểm tra xem driver đã được load hay chưa.
 
`lsmod | grep 8021q`

 - Kiểm tra lại cấu hình bonding ban đầu
 
```sh
~]$ ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN 
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,SLAVE,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast master bond0 state UP qlen 1000
    link/ether 52:54:00:19:28:fe brd ff:ff:ff:ff:ff:ff
3: eth1: <BROADCAST,MULTICAST,SLAVE,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast master bond0 state UP qlen 1000
    link/ether 52:54:00:f6:63:9a brd ff:ff:ff:ff:ff:ff
4: bond0: <BROADCAST,MULTICAST,MASTER,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP 
    link/ether 52:54:00:19:28:fe brd ff:ff:ff:ff:ff:ff
    inet 192.168.100.100/24 brd 192.168.100.255 scope global bond0
    inet6 fe80::5054:ff:fe19:28fe/64 scope link 
       valid_lft forever preferred_lft forever
```

Chú ý **eth0** và **eth1** có trạng thái **master bond0 state UP** và bond0 có trạng thái **MASTER,UP**

 - Kiểm tra định tuyến trên server 
 
```sh
~]$ ip route
192.168.100.0/24 dev bond0  proto kernel  scope link  src 192.168.100.100
169.254.0.0/16 dev bond0  scope link  metric 1004
```

Giả sử trên Switch đã có sẵn 1 VLAN với ID 192 với dải mạng 192.168.10.0/24, gateway 192.168.10.1. Ta sẽ tiến hành gán sub-interface vào VLAN này. Nếu như gán thành công, từ máy server phải ping được tới gateway của dải 192.168.10.0
 
 - Tạo file cấu hình cho sub-interface 
 
`vi /etc/sysconfig/network-scripts/ifcfg-bond0.192`

```sh
DEVICE=bond0.192
NAME=bond0.192
BOOTPROTO=none
ONPARENT=yes
IPADDR=192.168.10.1
NETMASK=255.255.255.0
VLAN=yes
NM_CONTROLLED=no
```

**Chú ý**. Đặt tên của sub-interface trùng với VLAN ID, ở ví dụ này VLAN-ID là 192.

 - Restart network
 
`systemctl restart network`

 - Kiểm tra lại cấu hình network, và ping thử tới gateway của dải 192.168.10.0
 
```sh
~]# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,SLAVE,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast master bond0 state UP qlen 1000
    link/ether 52:54:00:19:28:fe brd ff:ff:ff:ff:ff:ff
3: eth1: <BROADCAST,MULTICAST,SLAVE,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast master bond0 state UP qlen 1000
    link/ether 52:54:00:f6:63:9a brd ff:ff:ff:ff:ff:ff
4: bond0: <BROADCAST,MULTICAST,MASTER,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP 
    link/ether 52:54:00:19:28:fe brd ff:ff:ff:ff:ff:ff
    inet 192.168.100.100/24 brd 192.168.100.255 scope global bond0
    inet6 fe80::5054:ff:fe19:28fe/64 scope link 
       valid_lft forever preferred_lft forever
5: bond0.192@bond0: <BROADCAST,MULTICAST,MASTER,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP
    link/ether 52:54:00:19:28:fe brd ff:ff:ff:ff:ff:ff
    inet 192.168.10.1/24 brd 192.168.10.255 scope global bond0.192
    inet6 fe80::5054:ff:fe19:28fe/64 scope link
       valid_lft forever preferred_lft forever
```

 - Kiểm tra định tuyến trên server
 
```sh
~]$ ip route
192.168.100.0/24 dev bond0  proto kernel  scope link  src 192.168.100.100
192.168.10.0/24 dev bond0.192  proto kernel  scope link  src 192.168.10.1
169.254.0.0/16 dev bond0  scope link  metric 1004 
169.254.0.0/16 dev bond0.192  scope link  metric 1005
```

 - Ping tới gateway để kiểm tra
 
```sh
~]# ping -c2 192.168.10.99
PING 192.168.10.99 (192.168.10.99) 56(84) bytes of data.
64 bytes from 192.168.10.99: icmp_seq=1 ttl=64 time=0.781 ms
64 bytes from 192.168.10.99: icmp_seq=2 ttl=64 time=0.977 ms
--- 192.168.10.99 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 0.781/0.879/0.977/0.098 ms
```

 - Kết quả như vậy nghĩa là đã cấu hình sub-interface VLAN thành công !