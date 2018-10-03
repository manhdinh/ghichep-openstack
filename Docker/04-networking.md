## Docker networking
Các tính năng Docker networking cung cấp độc lập hoàn toàn cho các container. Bài dưới dây sẽ mô tả các dạng của network mặc định được tạo và cách tạo các network do người dùng định ra. Mô hình Docker networking dựa vào [libnetwork](https://github.com/docker/libnetwork/blob/master/docs/design.md)

### 1. Các network mặc định
Cài đặt Docker Engine, có 3 loại network mặc định trên host :
```sh
# docker network ls
NETWORK ID          NAME                DRIVER
34a43cf4f024        bridge              bridge
83af822c1610        host                host
02c1fcf12795        none                null
```

Bridge network đại diện cho `docker0` network, cũng là 1 phần của host network stack :
```sh
# ifconfig
docker0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.17.0.1  netmask 255.255.0.0  broadcast 0.0.0.0
        inet6 fe80::42:67ff:fe78:6b35  prefixlen 64  scopeid 0x20<link>
        ether 02:42:67:78:6b:35  txqueuelen 0  (Ethernet)

ens32: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.10.10.26  netmask 255.255.255.0  broadcast 10.10.10.255
        inet6 fe80::20c:29ff:fe1d:93bc  prefixlen 64  scopeid 0x20<link>
        ether 00:0c:29:1d:93:bc  txqueuelen 1000  (Ethernet)
...
```

`docker0` interface thuộc về Linux bridge nework được tạo bởi docker tại tiến trình daemon lúc khởi tạo. Docker ngẫu nhiên chọn 1 địa chỉ và subnet từ range private được chỉ định bởi RFC1918 (không được dùng trên host) và gắn chúng vào `docker0` :
```sh
bridge name     bridge id               STP enabled     interfaces
docker0         8000.024267786b35       no
```

Tất cả các docker container sẽ được kết nối tới `docker0` bridge theo mặc định thông qua 1 virtual Ethernet interface. Các container được kết nối tới `docker0` bridge có thể dùng iptables NAT rule được tạo bởi docker để liên lạc với bên ngoài. Không gian địa chỉ IP của bridge network là 172.17.0.0/16 theo mặc định.
Docker networking phân level trên Linux namespace kernel. KHi namespace được liệt kê dưới thư mục `/var/run/netns`, để dùng được các tool Linux net namespace tiêu chuẩn như `ip netns`, bạn nên link thư mục docker namepace `var/run/docker/netns`
```sh
cd /var/run/
ln -s /var/run/docker/netns .
ls -l
...
lrwxrwxrwx  1 root     root       21 Mar 23 12:55 netns -> /var/run/docker/netns
```

Tạo 1 container và kiểm tra việc host tạo virtual Ethernet interface mới : 
```sh
docker run -d -p 80:80 --name webserver kalise/httpd
ifconfig
...
veth80f977c: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet6 fe80::4d2:f3ff:fee5:c975  prefixlen 64  scopeid 0x20<link>
        ether 06:d2:f3:e5:c9:75  txqueuelen 0  (Ethernet)
...
brctl show
bridge name     bridge id               STP enabled     interfaces
docker0         8000.024267786b35       no              veth80f977c
```

Kết nối tới container vừa được tạo để kiểm tra network stack :
```sh
docker exec -it webserver /bin/bash
[root@7867782e7966 /]# yum install net-tools -y
[root@7867782e7966 /]# ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.17.0.2  netmask 255.255.0.0  broadcast 0.0.0.0
        inet6 fe80::42:acff:fe11:2  prefixlen 64  scopeid 0x20<link>
        ether 02:42:ac:11:00:02  txqueuelen 0  (Ethernet)

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>

[root@7867782e7966 /]# exit
```

Câu lệnh `docker run` tự động thêm mới các container tới bridge network. Các container trên network mặc định này sẽ có khả năng liên lạc được với các container khác bằng cách sử dụng `docker0` bridge IP. Docker không hỗ trợ automatic service discovery trên bridge network mặc định.

Ảnh sau thể hiện việc các container kết nối tới host qua `docker0` bridge
![Docker](/ManhDV/Docker/images/docker-net.png)

1 virtual ethernet device hoặc `veth` là 1 Linux networking interface hoạt động như là ống kết nối giữa 2 network namespace. 1 veth là 1 full duplex link với các inteface nằm trên các namespace. Traffic trên 1 interface được chuyển hướng ra trực tiếp tới interface khác. Docker network driver dùng các veth để cung cấp các kết nối giữa các namespace khi Docker network được tạo. Khi 1 container được attach vào Docker network, 1 đầu của veth sẽ đặt bên trong container, thông thường là `eth0` interface, đầu còn lại sẽ được attach tới Docker network.

`iptables` là hệ thống lọc packet native, là 1 phần của Linux kernel. Nó là L3/L4 firewall cung cấp các rule chain cho việc marking (đánh dấu), masquerading (cải biến) và dropping packet. Các built-in Docker network driver sử dụng iptables để phân mảnh (segment) network traffic, cung cấp host port mapping, và đánh dấu traffic cho các quyết định cân bằng tải.

Kiểm tra bridge network :
```sh
docker network inspect bridge
```

Output nhận được sẽ là :
```sh
[
    {
        "Name": "bridge",
        "Id": "edfb283dfae43d2d820c8a54e0ca4c80eee079f52093763dd1cbc8c2cfbf050d",
        "Scope": "local",
        "Driver": "bridge",
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.17.0.0/16",
                    "Gateway": "172.17.0.1"
                }
            ]
        },
        "Containers": {
            "af999a749414b55ca2d5280e22ae677a51d0fe27004aeb16a6b7d39b8dde7a1e": {
                "Name": "webserver",
                "EndpointID": "0df526d9aade3476a3bf53c4b1205d960621e6bef8dd334f2a5cb6b563122da3",
                "MacAddress": "02:42:ac:11:00:02",
                "IPv4Address": "172.17.0.2/16",
                "IPv6Address": ""
            }
        },
        "Options": {
            "com.docker.network.bridge.default_bridge": "true",
            "com.docker.network.bridge.enable_icc": "true",
            "com.docker.network.bridge.enable_ip_masquerade": "true",
            "com.docker.network.bridge.host_binding_ipv4": "0.0.0.0",
            "com.docker.network.bridge.name": "docker0",
            "com.docker.network.driver.mtu": "1500"
        }
    }
]
```

Bridge network là option thông thường nhất cho Docker container. Các lựa chọn khác như :

	- 1. host mode
	- 2. container mode
	- 3. none mode
	
### Host mode
Bên trong host mode, container chia sẻ networking namespace của host, thể hiện chúng trực tiếp ra bên ngoài. Điều này có nghĩa rằng bạn cần sử dụng *port mapping* để dùng các dịch vụ bên trong container

```sh
docker run -d -p 80 --net=host --name webserver httpd
docker exec -it webserver /bin/bash
/usr/local/apache2# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: ens32: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:1d:93:bc brd ff:ff:ff:ff:ff:ff
    inet 10.10.10.26/24 brd 10.10.10.255 scope global ens32
       valid_lft forever preferred_lft forever
    inet6 fe80::20c:29ff:fe1d:93bc/64 scope link
       valid_lft forever preferred_lft forever
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
    link/ether 02:42:67:78:6b:35 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:67ff:fe78:6b35/64 scope link
       valid_lft forever preferred_lft forever
/usr/local/apache2# exit
```

Kiểm tra host network :
```sh
docker network inspect host
```

Output sẽ là :
```sh
[
    {
        "Name": "host",
        "Id": "3dfa03da9332cb7e8ad1776fbc4f2c27536a3450d2e186e85ed2941177ddda90",
        "Scope": "local",
        "Driver": "host",
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": []
        },
        "Containers": {
            "bad4aa0bc42a685928b44e2cb97b8a6598f72c046af35e08e883c00e49c22f22": {
                "Name": "webserver",
                "EndpointID": "2fff57f78811a3b9a4ac7b299362ed579e73e0101d4438584a7cdfec57cfe9fb",
                "MacAddress": "",
                "IPv4Address": "",
                "IPv6Address": ""
            }
        },
        "Options": {}
    }
]
```

#### Container mode
Với container mode, container bị buộc dùng lại netwroking namespace của container khác. Điều này hữu dụng nếu bạn muốn cung cấp networking tùy chỉnh từ container được nói tới. VD : Kubernetes dùng cung cấp networking cho nhiều container tại cùng thời điểm bằng cách sử dụng 1 IP.
```sh
docker run -d -p 80:80 --name=web httpd
docker run -it --net=container:web --name=shell busybox
/ # ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:AC:11:00:02
          inet addr:172.17.0.2  Bcast:0.0.0.0  Mask:255.255.0.0
          inet6 addr: fe80::42:acff:fe11:2/64 Scope:Link

lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host

/ # exit
```

Chế độ `none mode` không cấu hình networking, chỉ được dùng cho các container không yêu cầu truy cập network :
```sh
docker run --net=none -itd --name=centos-none centos
```

### 2. User defined bridge network

Docker cho phép tạo các user define network để cô lập các container. Docker cung cấp các network driver cho việc tạo các network này. Với Docker, hoàn toàn có thể tạo ra nhiều network và add các container vào nhiều hơn 1 network. Các container có thể chỉ liên lạc bên trong network chứ không thể đi qua. Container được attach tới 2 network có thể liên lạc với các container member trong cả 2 network. Khi 1 container  được liên kết tới nhiều network, các kết nối tới bên ngoài được cung cấp thông qua non-internal network đầu tiên.

Ảnh sau thể hiện mô hình của Docker multiple networking :

![Docker](/ManhDV/Docker/images/cnm-model.jpg)

Tạo 1 custom bridge network :
```sh
docker network create --subnet=192.168.1.0/24 --gateway=192.168.1.1 bridge1
docker network ls
NETWORK ID          NAME                DRIVER
34a43cf4f024        bridge              bridge
abb8c6759678        bridge1             bridge
83af822c1610        host                host
02c1fcf12795        none                null
```

Một interface mới được gọi là `br-<net-id>` được thêm tới host network stack :
```sh
ifconfig
br-86e7fa795e38: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 192.168.1.1  netmask 255.255.255.0  broadcast 0.0.0.0
        ether 02:42:dc:79:45:15  txqueuelen 0  (Ethernet)

docker0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.17.0.1  netmask 255.255.0.0  broadcast 0.0.0.0
        ether 02:42:e1:f4:01:73  txqueuelen 0  (Ethernet)

brctl show
bridge name     bridge id               STP enabled     interfaces
br-86e7fa795e38 8000.0242e3712735       no
docker0         8000.024267786b35       no
```

Có thể đổi tên của interface để giống với network label :
```sh
docker network create \
--subnet=192.168.2.0/24 \
--gateway=192.168.2.1 \
--opt="com.docker.network.bridge.name=bridge2" \
bridge2

ifconfig
br-c5df690ed81e: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 192.168.1.1  netmask 255.255.255.0  broadcast 0.0.0.0
        ether 02:42:dc:79:45:15  txqueuelen 0  (Ethernet)

bridge2: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.2.1  netmask 255.255.255.0  broadcast 0.0.0.0
        inet6 fe80::42:e8ff:fe12:3268  prefixlen 64  scopeid 0x20<link>
        ether 02:42:e8:12:32:68  txqueuelen 0  (Ethernet)

docker0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.17.0.1  netmask 255.255.0.0  broadcast 0.0.0.0
        ether 02:42:e1:f4:01:73  txqueuelen 0  (Ethernet)
```

Kiểm tra network vừa được tạo :
```sh
docker network inspect bridge2
```

Output như sau :
```sh
[
    {
        "Name": "bridge2",
        "Id": "edb88c34314b639632ee73d499bb0a4a25a343b401d857d050fcb46cc2d8105d",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "192.168.2.0/24",
                    "Gateway": "192.168.2.1"
                }
            ]
        },
        "Internal": false,
        "Containers": {
            }
        },
        "Options": {
            "com.docker.network.bridge.name": "bridge2"
        },
        "Labels": {}
    }
]
```

Khởi động 1 container với network này :
```sh
docker run -d -p 80:80 --net=bridge2 --name apache kalise/httpd
```

Kiểm tra container :
```sh
[
    {
        "Name": "bridge2",
        "Id": "edb88c34314b639632ee73d499bb0a4a25a343b401d857d050fcb46cc2d8105d",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "192.168.2.0/24",
                    "Gateway": "192.168.2.1"
                }
            ]
        },
        "Internal": false,
        "Containers": {
            "5194ff09e858ea16a6e5577f9c33bb1313a06a599c348f216990d6c54dfdae26": {
                "Name": "apache",
                "EndpointID": "73e7931dc11d967c2129e7e53c5b47d1f212ec7977e8df6bf9bf81e94252fc3b",
                "MacAddress": "02:42:c0:a8:02:03",
                "IPv4Address": "192.168.2.3/24",
                "IPv6Address": ""
            }
        },
        "Options": {
            "com.docker.network.bridge.name": "bridge2"
        },
        "Labels": {}
    }
]
```

Một container đang chạy có thể được attach vào network
```sh
docker network connect bridge1 apache
docker exec -it apache bash
[root@5194ff09e858 /]# ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.2.3  netmask 255.255.255.0  broadcast 0.0.0.0
        ether 02:42:c0:a8:02:03  txqueuelen 0  (Ethernet)

eth1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.1.3  netmask 255.255.255.0  broadcast 0.0.0.0
        ether 02:42:c0:a8:01:03  txqueuelen 0  (Ethernet)

[root@5194ff09e858 /]# exit
```

Remove network :
```sh
docker network rm bridge2
Error response from daemon: network bridge2 has active endpoints
docker rm -f apache
apache
docker network rm bridge2
```

### 3. Sử dụng custom Docker network
Docker networking cho phép xây dụng các hạ tầng network cho các use case thực tế. Ở ví dụ sau, chúng ta sẽ triển khai một ứng dụng multi-tier như Wordpress blog platform gồm MariaDB và Apache/PHP server. Ứng dụng Apache/PHP được lăng nghe cho các kết nối đầu vào từ host trên port 80. Nó sử dụng 1 internal user-defined network để liên lạc với MariaDB server lăng nghe trên port 3306. MariaDB server được attach tới internal network và không expose bất kỳ port nào tới host.

Tạo 1 internal network :
```sh
docker network create --subnet=192.168.1.0/24 --gateway=192.168.1.1 internal
```

và start 1 MariaDB server trên network này :
```sh
docker run --name mariadb -d  --net internal \
         -e MARIADB_ROOT_PASSWORD="bitnami123" \
         -e MARIADB_DATABASE="wordpress" \
         -e MARIADB_USER="bitnami" \
         -e MARIADB_PASSWORD="bitnami123" \
         bitnami/mariadb:latest
```

Start Wordpress app trên external network và expose port 80 ra host :
```sh
docker run --name wordpress -d -p 80:80 \
         -e WORDPRESS_DATABASE_NAME="wordpress" \
         -e WORDPRESS_DATABASE_USER="bitnami" \
         -e WORDPRESS_DATABASE_PASSWORD="bitnami123" \
         bitnami/wordpress:latest
```

Sau đó attach wordpress container tới internal network để liên lạc với mariaDB container
```sh
docker network connect internal wordpress
```

Bây giờ 2 container là mariadb và wordpress sẽ liên lạc với nhau qua internal network

### 3. Inter Container Communication

Mặc định thì, Docker có `inter-container communication` được bật bởi option `enable_icc=true`, có nghĩa rằng các container trên host được free để liên lạc. Việc liên lạc với bên ngoài được kiểm soát thông qua *itables* và *ip_forwarding*.

Vì lý do an ninh, có thể disable `inter-container communication* bằng việc đặt option `enable_icc=false` trong các user define network. Tạo một internal network với `inter-container communication` được tắt :
```sh
# docker network create \
> --subnet=192.168.2.0/24 \
> --gateway=192.168.2.1 \
> --opt="com.docker.network.bridge.enable_icc=false" internal

# docker network inspect internal | grep icc
            "com.docker.network.bridge.enable_icc": "false"
```

Tạo 2 container với network này và kiểm tra việc chúng không thể kết nối với nhau :
```sh
# docker run -itd --name=centos1 --net=internal --ip=192.168.2.11 centos
# docker run -itd --name=centos2 --net=internal --ip=192.168.2.12 centos

# docker exec -it centos1 bash
[root@8afc63c7994c /]# ifconfig eth0
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.2.11  netmask 255.255.255.0  broadcast 0.0.0.0

[root@8afc63c7994c /]# ping 192.168.2.12
--- 192.168.2.12 ping statistics ---
...
568 packets transmitted, 0 received, 100% packet loss, time 567000ms
[root@8afc63c7994c /]# exit

# docker exec -it centos2 bash
[root@64b72b9b605c /]# ifconfig eth0
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.2.12  netmask 255.255.255.0  broadcast 0.0.0.0
Cping 192.168.2.11
...
--- 192.168.2.11 ping statistics ---
766 packets transmitted, 0 received, 100% packet loss, time 764999ms
[root@64b72b9b605c /]# exit
```

### 4. Embedded DNS Service
Từ Docker 1.10, docker daemon giới thiệu `embedded DNS server` trên user define network, dùng để cung cấp built-in service discovery cho bất kỳ container nào được tạo với tên hợp lệ. Service này KHÔNG khả dụng trên default bridge network để tương thích ngược với các phiên bản truyền thống. Container name được dùng để discover một container bên trong một user-defined docker network. Embedded DNS server duy trì sự ánh xạ giữa container name và IP của nó trên network mà container được kết nối tới.

Tạo một container trên default bridge nework 
```sh
# docker run --net=bridge -itd --name=centos_one centos
```

Tạo container thứ 2 và thứ 3 trên internal network :
```sh
# docker run --net=internal -itd --name=centos_two centos
# docker run --net=internal -itd --name=centos_three centos
```

Kết nối container thứ 2 tới default bridge network vì thế nó sẽ có 2 địa chỉ IP. IP đầu tiên trên internal network và IP thứ 2 trên default network :
```sh
# docker network connect bridge centos_two
```

Attach container thứ 2 và kiểm tra kết nối của các container khác
```sh
# docker attach centos_two
[root@602c086b65df /]# ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.17.0.3  netmask 255.255.0.0  broadcast 0.0.0.0

eth1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.1.2  netmask 255.255.255.0  broadcast 0.0.0.0

[root@602c086b65df /]# ping centos_three
PING centos_three (192.168.1.3) 56(84) bytes of data.
64 bytes from centos_three.internal (192.168.1.3): icmp_seq=1 ttl=64 time=1.17 ms
64 bytes from centos_three.internal (192.168.1.3): icmp_seq=2 ttl=64 time=0.054 ms
^C
[root@602c086b65df /]# ping centos_one

ping: unknown host centos_one
[root@602c086b65df /]# ping 172.17.0.2
PING 172.17.0.2 (172.17.0.2) 56(84) bytes of data.
64 bytes from 172.17.0.2: icmp_seq=1 ttl=64 time=0.071 ms
64 bytes from 172.17.0.2: icmp_seq=2 ttl=64 time=0.057 ms
```

Container thứ 2 có thể kết nối từ container 1 bởi IP nhưng không thể phân giải được tên bởi vì container 1 nằm trên network default, nơi mà dịch vụ DNS không thể chạy. Bên cạnh đó, container thứ 2 có thể phân giải nhờ Embedded DNS service chạy trên internal network.
Kiểm tra `resolv.conf` trên container số 1 :
```sh
# docker attach centos_one
[root@ceb267dd3044 /]# cat /etc/resolv.conf
# Generated by NetworkManager
nameserver 8.8.8.8
[root@ceb267dd3044 /]#
```

Kiểm tra trên container 2 :
```sh
# docker attach centos_two
[root@602c086b65df /]# cat /etc/resolv.conf
nameserver 127.0.0.11
options ndots:0
[root@602c086b65df /]#
```

Kiểm tra trên container 3 :
```sh
# docker attach centos_three
[root@5c667ea10650 /]# cat /etc/resolv.conf
nameserver 127.0.0.11
options ndots:0
[root@5c667ea10650 /]#
```

Như ta thấy, container đầu tiên file `resolv.conf` chỏ tới host file trong khi container 2 và 3 chỏ tới DNS service `127.0.0.11`. Chú ý đó không phải là *loopback interface*.
