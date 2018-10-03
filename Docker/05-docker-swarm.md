## Docker Swarm

Swarm node cung cấp cấp giải pháp all-built-in Docker cluster bao gồm orchestrator của các contaienr và network cho multi host. Mục tiêu của Docker Swarm là : 

1. High avaibility

2. Scaling

3. Load balancing
 
Một Swarm cluster được tạo bởi các node manager và các node worker :
 - **Managers** : các node chịu trách nhiệm cho việc quản lý như : quản lý cluster và các dịch vụ chạy trên nó. 
 - **Worker** : các node chạy các dịch vụ của user.
 
Ban đầu, cluster sẽ gồm 3 node :1 node manager và 2 node worker. Trong Swarm mode, 1 node có thể là manager và worker đồng thời. Với mô hình multiple manager thì số node được khuyến nghị là 3 node. Swarm manager sử dụng [Raft](https://raft.github.io/) consensus algorithm để quản lý trạng thái swarm. Raft sử dụng `quorum` cho các propose update tới swarm và đồng bộ trạng thái cho tất cả các manager node

### 1. Setup Swarm

Trên cả 3 node, cài đặt Docker engine. Các port sau phải được mở trên cluster network : 
 
 - TCP port 2377 : port liên lạc trong việc quản lý cluster
 - TCP và UDP port 7946 : liên lạc giữa các node
 - UDP port 4789 : overlay network traffic

Port 2375 và 2376 (cho TLS) nên được mở ở frontend network cho Docker API service trong trường hợp cần sử dụng việc quản lý từ xa. 

Đảm bảo Docker engine daemon chạy trên tất cả các host machine :

```sh
systemctl start docker
systemctl enable docker
systemctl status docker
```

Login tới manager node và tạo swarm 

```sh
docker swarm init --advertise-addr ens33
```

Option `--advertise-addr ens33` cho biết cluster back network được attach tới port `ens33`. Các node khác trên swarm cần phải kết nối được tới manager thông qua interface này. Mô hình này sử dụng `ens32` cho việc try cập tới các service của user và `ens33` cho các traffic clustering giữa các node.

Kiểm tra swarm node status : 

```sh
docker node list
ID                           HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
tkxuxun03da7y2bozka50mpxi *  swarm00   Ready   Active        Leader
```

Thêm các node khác vào cluster như là worker node, ta cần 1 token từ swarm manager
```sh
docker swarm join-token worker
To add a worker to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-3mbvdd9cay5eobj2om2pg5bdnl22z0qntmvvvslzsyt14mhgro-2qdyke72teir2erm9tuezaudj \
    192.168.2.60:2377
```

Login vào các node khác và thêm chúng vào cluster :
```sh
docker swarm join \
    --token SWMTKN-1-3mbvdd9cay5eobj2om2pg5bdnl22z0qntmvvvslzsyt14mhgro-2qdyke72teir2erm9tuezaudj \
    192.168.2.60:2377

This node joined a swarm as a worker.
```

Trên manager, kiểm tra swarm :
```sh
docker node list
ID                           HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
ickjus9bta1julsad2ls3paum    swarm01   Ready   Active
ryc6zlbxrhvblc7lk8pl3jsva    swarm02   Ready   Active
tkxuxun03da7y2bozka50mpxi *  swarm00   Ready   Active        Leader
```

Các worker node có thể thúc đẩy để trở thành có manager role:
```sh
docker node promote swarm01 swarm02

docker node list

ID                           HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
3kru4v3w3lezys0tc9cnoczou *  swarm00   Ready   Active        Leader
e2kqucdbmkzm5ks6s2pciyg9v    swarm01   Ready   Active        Reachable
mdmnfom70hh86go8up5zl272y    swarm02   Ready   Active        Reachable
```

Các node có thể được thêm như là manager bằng lấy token từ manager đã có :
```sh
docker swarm join-token manager
To add a manager to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-06y3xg5vjkh0tzla9fodgp6zrzsqqus8974b75umbvxeemwds9-1dlmdl3jxiz27ptggus4d9plg \
    10.10.10.60:2377
```

và chạy join command trên node mới :
```sh
[root@swarm09 ~]# docker swarm join \
    --token SWMTKN-1-06y3xg5vjkh0tzla9fodgp6zrzsqqus8974b75umbvxeemwds9-1dlmdl3jxiz27ptggus4d9plg \
    10.10.10.60:2377
```

Khi thêm các manager node tới 1 swarm, chú ý tới datacenter topology. Các distribute manager node cần ít nhất 3 avaibility zone để phòng khi toàn bộ cụm máy có vấn đề.

Để giáng cấp 1 node manager thành worker :
```sh
run docker node demote swarm02
```

Remove node ra khỏi swarm :
```sh
run docker node rm swarm01
```

Để rejoin 1 node tới swarm với trạng thái mới :
```sh
docker swarm join swarm01
```

Trong môi trường production, nên cân nhắc chạy các dịch vụ của user trên workker node để tránh việc thiết tài nguyên CPU và RAM. Để tránh việc ảnh hưỡng giữa các hành động quản lý và các service người dùng, bạn có thể `drain` 1 manager node để khiến nó không khả dụng cho các dịch vụ :
```sh
docker node update --availability drain swarm00
ID                           HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
3kru4v3w3lezys0tc9cnoczou *  swarm00   Ready   Drain         Leader
e2kqucdbmkzm5ks6s2pciyg9v    swarm01   Ready   Active        Reachable
mdmnfom70hh86go8up5zl272y    swarm02   Ready   Active        Reachable
```

Khi drain 1 node, scheduler sẽ phân bố lại các service của user chạy trên node này tới các worker node khác trên cluster.
Nếu bạn chỉ muốn sửa lỗi 1 node mà không dịch chuyển các service đã tồn tại, bạn có thể `pause` node : 
```sh
docker node update --availability pause swarm00
ID                           HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
3kru4v3w3lezys0tc9cnoczou *  swarm00   Ready   Pause         Leader
e2kqucdbmkzm5ks6s2pciyg9v    swarm01   Ready   Active        Reachable
mdmnfom70hh86go8up5zl272y    swarm02   Ready   Active        Reachable
```

Để `active` lại node :
```sh
docker node update --availability pause swarm00
ID                           HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
3kru4v3w3lezys0tc9cnoczou *  swarm00   Ready   Active        Leader
e2kqucdbmkzm5ks6s2pciyg9v    swarm01   Ready   Active        Reachable
mdmnfom70hh86go8up5zl272y    swarm02   Ready   Active        Reachable
```

### 2. Swarm networking
Swarm mode setup tạo ra một networking layout dựa trên overlay network driver. Overlay network là một network được dựng trên top của các network khác. Các node trong overlay network có thể được kết nối bởi các virutal hoặc logical link, mỗi link tương ứng với 1 phân xuyên suất của một hoặc nhiều physical link trên các network lớp dưới.

Swarm mode thúc đẩy trên các overlay network để khiến tất cả các container đang chạy trên bất kỳ node nào trên cluster cũng thuộc về cùng 1 L2 network. Một khi đã được kích hoạt, swarm tạo 1 overlay layout để kết nối tấ cả các container với nhau, không quan trọng container đang chạy trên node nào.

Trong phần này, chúng ta sẽ xem chi tiết layout này hoạt động. Swarm overlay network dựa bên VXLAN technology.

Login vào 1 trong các node các kiểm tra network :
```sh
docker network list
NETWORK ID          NAME                DRIVER              SCOPE
862ce491c4d4        bridge              bridge              local
e73fde81ef50        docker_gwbridge     bridge              local
f0c9ed46f0b6        host                host                local
1qc6vhwhaeqn        ingress             overlay             swarm
eaef890efed3        none                null                local
```

Có thể thấy over network với tên `ingress` và 1 gateway bridge network tên là `docker_gwbridge`. Kiểm tra kỹ càng với overlay đầu tiên :
```sh
docker network inspect ingress
```

```sh
[
    {
        "Name": "ingress",
        "Id": "1qc6vhwhaeqn0n9z2hdlawz72",
        "Created": "2017-03-27T15:47:53.139535526+02:00",
        "Scope": "swarm",
        "Driver": "overlay",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "10.255.0.0/16",
                    "Gateway": "10.255.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Containers": {
            "ingress-sbox": {
                "Name": "ingress-endpoint",
                "EndpointID": "c8295e304e17f569575a6c0ba35c0d41a25515672776cf2da95f3715096ba442",
                "MacAddress": "02:42:0a:ff:00:03",
                "IPv4Address": "10.255.0.3/16",
                "IPv6Address": ""
            }
        },
        "Options": {
            "com.docker.network.driver.overlay.vxlanid_list": "4096"
        },
        "Labels": {},
        "Peers": [
            {
                "Name": "swarm00-a73dffac9ac0",
                "IP": "192.168.2.60"
            },
            {
                "Name": "swarm01-5bed1d4ed40a",
                "IP": "192.168.2.61"
            },
            {
                "Name": "swarm02-c97504fdbbf3",
                "IP": "192.168.2.62"
            }
        ]
    }
]
```

Trên network này có dải là `10.255.0.0/16`. Đây là 1 container gọi là `ingress-sbox`. Container này là container đặc biệt được tạo tự động bởi swarm trên mỗi node trong cluster.

Inspect gateway bride network 
```sh
docker network inspect docker_gwbridge
```

```sh
[
    {
        "Name": "docker_gwbridge",
        "Id": "e73fde81ef50f8bc38516ff0545be26a150af8a4020fbc4ff07b5bb5050db84f",
        "Created": "2017-03-14T16:56:46.874874148+01:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.18.0.0/16",
                    "Gateway": "172.18.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Containers": {
            "ingress-sbox": {
                "Name": "gateway_ingress-sbox",
                "EndpointID": "ee7552c5fd805e4f186eeb3d2ccf0576abdf0fca00c53f07cbb33d1a399610bc",
                "MacAddress": "02:42:ac:12:00:02",
                "IPv4Address": "172.18.0.2/16",
                "IPv6Address": ""
            }
        },
        "Options": {
            "com.docker.network.bridge.enable_icc": "false",
            "com.docker.network.bridge.enable_ip_masquerade": "true",
            "com.docker.network.bridge.name": "docker_gwbridge"
        },
        "Labels": {}
    }
]
```

Dải mạng này là `172.18.0.0/16`, đây là container được gọi là `gateway_ingress-sbox`. Kiểm tra network namespace :
```sh
ip netns list
1-1qc6vhwhae (id: 1)
ingress_sbox (id: 2)
```

Có 2 namespace. Kiểm tra interface trên namespace `ingress_sbox`, chúng ta sẽ thấy 2 interface :
```sh
ip netns exxec ingress_sbox ip addr
...
51: eth0@if52: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP
    link/ether 02:42:0a:ff:00:03 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.255.0.3/16 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:aff:feff:3/64 scope link
       valid_lft forever preferred_lft forever

53: eth1@if54: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP
    link/ether 02:42:ac:12:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 1
    inet 172.18.0.2/16 scope global eth1
       valid_lft forever preferred_lft forever
    inet6 fe80::42:acff:fe12:2/64 scope link
       valid_lft forever preferred_lft forever
```

Interfce đầu tiên `eth0` trên ingress overlay network và `eth1` trên gateway bridge network. Mục tieuencuar ingress namespace sandbox là cung cấp entry point để phơi service ra bên ngoài. Chúng ta sẽ thấy nó khi triển khai 1 số service trong cluster.

Quay trở lại với các namepace khác và kiểm tra interfaces :
```sh
ip netns exec 1-1qc6vhwhae ip addr

1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever

2: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP
    link/ether 76:18:82:e1:ff:26 brd ff:ff:ff:ff:ff:ff
    inet 10.255.0.1/16 scope global br0
       valid_lft forever preferred_lft forever
    inet6 fe80::c8cf:56ff:fec9:4d93/64 scope link
       valid_lft forever preferred_lft forever

50: vxlan1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master br0 state UNKNOWN
    link/ether a2:9e:a4:4f:3b:1b brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet6 fe80::a09e:a4ff:fe4f:3b1b/64 scope link
       valid_lft forever preferred_lft forever

52: veth2@if51: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master br0 state UP
    link/ether 76:18:82:e1:ff:26 brd ff:ff:ff:ff:ff:ff link-netnsid 1
    inet6 fe80::7418:82ff:fee1:ff26/64 scope link
       valid_lft forever preferred_lft forever
```

Namepace này là 1 bridge cho overlay network. Đó là gateway cho các container để kết nối các container khác trên cùng 1 overlay network.

Mô hình setup như sau :

![docker](/ManhDV/Docker/images/swarm.png)

Có thể thấy, swarm tạo 1 overlay network bao phủ các node trên cluster và chúng nằm trên cùng 1 mạng logical L2

## 3. Triển khai service

Trong Swarm mode, 1 service được coi là 1 abstraction để có thể khiến các container luôn truy cập được, không phụ thuộc vào node nào container đang chạy trên nó. Trong section này, chúng ta sẽ triển khia 1 ứng dụng nodejs web service đơn giản. Service này sẽ lắng nghe port 8080 và trả lời với các message từ IP được rằng buộc. Để khiến service này khả dụng với mạng bên ngoài, chúng ta sẽ map port 8080 tới port 80

Trên master node, tạo service : 

```sh
docker service create \
  --replicas 1 \
  --name nodejs \
  --publish 80:8080 \
  --constraint 'node.role==worker' \
kalise/nodejs-web-app:latest
```

Với câu lệnh trên, chúng ta sẽ nói với swarm rằng hãy tạo 1 service với tên là `nodejs` được cung cấp bởi 1 single container `replicas 1` dựa trên image ` kalise/nodejs-web-app:latest`. Chúng ta sẽ map port lắng nghe 8080 tới port bên ngoài là 80, và sẽ bắt buộc container chạy trên worker nodes.

List các service : 
```sh
docker service list
ID            NAME    MODE        REPLICAS  IMAGE
lo6p4xrhlz3n  nodejs  replicated  1/1       kalise/nodejs-web-app:latest

docker service ps nodejs
ID            NAME      IMAGE                         NODE     DESIRED STATE  CURRENT STATE           ERROR  PORTS
uf9cr96onyh6  nodejs.1  kalise/nodejs-web-app:latest  swarm01  Running        Running 2 minutes ago
```

Inspect service vừa được tạo :
```sh
dockẻ service inspect --pretty nodejs
```

```sh
ID:             lo6p4xrhlz3nnlo06yg5yn1jy
Name:           nodejs
Service Mode:   Replicated
 Replicas:      1
Placement:Contraints:   [node.role==worker]
UpdateConfig:
 Parallelism:   1
 On failure:    pause
 Max failure ratio: 0
ContainerSpec:
 Image:         kalise/nodejs-web-app:latest
Resources:
Endpoint Mode:  vip
Ports:
 PublishedPort 80
  Protocol = tcp
  TargetPort = 8080
```

Service được back bởi 1 container đang chạy trên worker node `swarm01`

Login vào worker node và inspect các namespace liên quan :
```sh
ip netns
ff939e571cc9 (id: 3)
1-1qc6vhwhae (id: 1)
ingress_sbox (id: 2)

ip netns exec 97c81c75e70a ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1450
        inet 10.255.0.7  netmask 255.255.0.0  broadcast 0.0.0.0
        inet6 fe80::42:aff:feff:7  prefixlen 64  scopeid 0x20<link>
        ether 02:42:0a:ff:00:07  txqueuelen 0  (Ethernet)

eth1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.18.0.3  netmask 255.255.0.0  broadcast 0.0.0.0
        inet6 fe80::42:acff:fe12:3  prefixlen 64  scopeid 0x20<link>
        ether 02:42:ac:12:00:03  txqueuelen 0  (Ethernet)
...
```

Container có interface `eth0` được attach tới ingress overlay network và interface eth1 được attach với gateway bridge network 
```sh
docker network list
NETWORK ID          NAME                DRIVER              SCOPE
5b47cdaf82b1        bridge              bridge              local
18ef9a1c2e08        docker_gwbridge     bridge              local
5f03d7b5240a        host                host                local
1qc6vhwhaeqn        ingress             overlay             swarm
3cd5989ee74c        none                null                local
```

Inspect ingress network
```sh
[
    {
        "Name": "ingress",
        "Id": "1qc6vhwhaeqn0n9z2hdlawz72",
        "Created": "2017-03-27T15:50:28.680582431+02:00",
        "Scope": "swarm",
        "Driver": "overlay",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "10.255.0.0/16",
                    "Gateway": "10.255.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Containers": {
            "c5ce0bf928def33a6e7680df34fe7a57aeee74f4b9f3270b8d59a39c03eacc53": {
                "Name": "nodejs.1.uf9cr96onyh6cw9ii97pgx5i4",
                "EndpointID": "4213a7e8373a538f3816d12830449aed032d442c00eb0670ee4f881f819ee2f7",
                "MacAddress": "02:42:0a:ff:00:07",
                "IPv4Address": "10.255.0.7/16",
                "IPv6Address": ""
            },
            "ingress-sbox": {
                "Name": "ingress-endpoint",
                "EndpointID": "7491408342eb79dc0030f5e1b564ab54719532449427b9f6fb57ff33daeb3336",
                "MacAddress": "02:42:0a:ff:00:04",
                "IPv4Address": "10.255.0.4/16",
                "IPv6Address": ""
            }
        },
        "Options": {
            "com.docker.network.driver.overlay.vxlanid_list": "4096"
        },
        "Labels": {},
        "Peers": [
            {
                "Name": "swarm01-5bed1d4ed40a",
                "IP": "192.168.2.61"
            },
            {
                "Name": "swarm00-a73dffac9ac0",
                "IP": "192.168.2.60"
            },
            {
                "Name": "swarm02-c97504fdbbf3",
                "IP": "192.168.2.62"
            }
        ]
    }
]
```

Gateway bridge network 
```sh
docker network inspect docker_gwbridge
```

```sh
[
    {
        "Name": "docker_gwbridge",
        "Id": "18ef9a1c2e0885ce3c0e20498277df68e30f91f5b1912068417d9d19c98e2ef7",
        "Created": "2017-03-16T17:53:58.301607018+01:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.18.0.0/16",
                    "Gateway": "172.18.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Containers": {
            "c5ce0bf928def33a6e7680df34fe7a57aeee74f4b9f3270b8d59a39c03eacc53": {
                "Name": "gateway_c5ce0bf928de",
                "EndpointID": "c8261f30b1177a2da884ce044c6e21b56f6a4e883b48c012793157ef6800aa19",
                "MacAddress": "02:42:ac:12:00:03",
                "IPv4Address": "172.18.0.3/16",
                "IPv6Address": ""
            },
            "ingress-sbox": {
                "Name": "gateway_ingress-sbox",
                "EndpointID": "406a79b5f97ea47a78eefb6c4902cf97338373a979f1ac42c16ada439852a54d",
                "MacAddress": "02:42:ac:12:00:02",
                "IPv4Address": "172.18.0.2/16",
                "IPv6Address": ""
            }
        },
        "Options": {
            "com.docker.network.bridge.enable_icc": "false",
            "com.docker.network.bridge.enable_ip_masquerade": "true",
            "com.docker.network.bridge.name": "docker_gwbridge"
        },
        "Labels": {}
    }
]
```

Network layout sẽ dạng như sau : 

![docker](/ManhDV/Docker/images/swarm-layout-01.png)

Có thể thấy gateway bridge network được coi là ingress point cho tất cả các request tới nodejs service được expose `curl http://swarm01:80`. Các request này được kiểm soát bởi ingress sandbox trên mỗi node và sau đó được gửi tới nodejs container thông qua overlay network. Tất cả quy trình được thiết lập bởi iptables chain.

Iptable trên host chặn các request và chuyển tới địa chỉ ingress sandbox trên gateway bridge network, vd `172.18.0.2:80`. Sau đó request tới sandbox gateway, nơi sẽ chuyển các request tới địa chỉ `10.255.0.4`, IP của interface trên infress overlay network.

Ở thời điểm này, request sẽ được gửi tới container đích trên overlay network. Swarm dùng các triển khai load balancer đã được nhúng trên Linux kernel được gọi là **IPVS**. Để check cấu hình của IPVS, cần cài đặt admin tool `ipvsadm`

```sh
yum install ipvsadm -y

ip netns exec ingress_sbox ipvsadm -ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
FWM  263 rr
  -> 10.255.0.7:0                 Masq    1      0          0
  -> 10.255.0.7:0                 Masq    1      0          0
```

Cấu hình trên sẽ báo IPVS load balancer để forward request tới IP `10.255.0.7`, là IP của nodejs container trên các workder node khác. Đó chỉ là backend server cho internal load balancer. Các request tới backend thông qua overlay network.

### 4. Scaling service

Điều gì sẽ xảy ra khi mở rộng service tới nhiều hơn 1 container? Swarm mode có tính năng để mở rộng các container để hỗ trợ một dịch vụ. Thử mở rộng service tới 2 container.

Trên master node :
```sh
docker service scale nodejs=2
nodejs scaled to 2

docker service ps nodejs
ID            NAME      IMAGE                         NODE     DESIRED STATE  CURRENT STATE           ERROR  PORTS
l4ubi2w5cijm  nodejs.1  kalise/nodejs-web-app:latest  swarm01  Running        Running 2 hours ago
rcftpr36e6or  nodejs.2  kalise/nodejs-web-app:latest  swarm02  Running        Running 1 minutes ago
```

Chúng ta sẽ thấy 1 container mới đang chạy trên worker node `swarm02`. Login vào node này và kiểm tra container interface 
```sh
ip netns list
83ec387bc9f1 (id: 3)
1-1qc6vhwhae (id: 1)
ingress_sbox (id: 2)

ip netns exec 83ec387bc9f1 ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1450
        inet 10.255.0.8  netmask 255.255.0.0  broadcast 0.0.0.0
        inet6 fe80::42:aff:feff:8  prefixlen 64  scopeid 0x20<link>
        ether 02:42:0a:ff:00:08  txqueuelen 0  (Ethernet)

eth1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.18.0.3  netmask 255.255.0.0  broadcast 0.0.0.0
        inet6 fe80::42:acff:fe12:3  prefixlen 64  scopeid 0x20<link>
        ether 02:42:ac:12:00:03  txqueuelen 0  (Ethernet)
...
```

Container này có IP `10.255.0.8` trên ingress overlay network.

Trở lại với master node, kiểm tra cấu hình IPVS load balancer : 
```sh
ip netns exec ingress_sbox ipvsadm -ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
FWM  263 rr
  -> 10.255.0.7:0                 Masq    1      0          0
  -> 10.255.0.8:0                 Masq    1      0          0
```

Các client request được kiểm soát bởi IPVS theo kiểu round-robin.

Layout như sau : 

![docker](/ManhDV/Docker/images/swarm-layout-02.png)

### 5. Routing Mesh
Tất cả các node tham gia 1 swarm, đều có thể định tuyến các incoming request từ ingress sandbox của nó tới service cụ thể, không cần biết là service đó đang chạy trên node nào. Tính năng này được gọi là **Rouing Mesh** và được dùng để expose 1 service ra mạng bên ngoài. VD, 1 request được expose tới nodejs service `curl http://swarm00:80` hoặc `curl http://swarm02:80` sẽ được kiểm soát bởi IPVS trên các node `swarm00` và `swarm02` tương ứng. 
```sh
netstat -natp | grep 80
tcp6       0      0 :::80                   :::*                    LISTEN      488/docker-proxy

curl 10.10.10.60:80
Hello World! from 10.255.0.7
```

```sh
netstat -natp | grep 80
tcp6       0      0 :::80                   :::*                    LISTEN      488/docker-proxy

curl 10.10.10.61:80
Hello World! from 10.255.0.7
```

```sh
netstat -natp | grep 80
tcp6       0      0 :::80                   :::*                    LISTEN      488/docker-proxy

curl 10.10.10.62:80
Hello World! from 10.255.0.7
```

Chú ý rằng **routing mesh** là lựa chọn mặc định. Nếu bạn cần service được expose chỉ trên node thật mà service đang chạy, bạn có thể force chúng với option `--publish mode=host,target=<target_port>,published=<published_port>` trong quá trình tạo service.
Ví dụ : 
```sh
docker service create  \
          --replicas 1 \
          --name nodejs \
          --publish mode=host,target=8080,published=80 \
          --constraint 'node.hostname==node01' \
          kalise/nodejs-web-app:latest
```

Chúng ta có thể thấy chỉ `node01` chạy service với port được expose.
```sh
netstat -natp | grep 80


netstat -natp | grep 80
tcp6       0      0 :::80                   :::*                    LISTEN      488/docker-proxy

netstat -natp | grep 80
```

Thông thường, routing mesh là cách được khuyến khích dùng để expose port.

### 6. Service Failover

Trong phần này chúng ta sẽ tìm hiểu cách Swarm kiểm soát việc chuyển đổi dự phòng trong trường hợp service bị fail. Single container sẽ không có thay thế nếu như nó bị fail, bị xóa vì 1 số nguyên nhân. Để phòng tránh việc này, Swarm giới thiệu khái niệm **replica** (bản sao). Replica đảm bảo rằng 1 service luôn sẽ có 1 con số nhất định container đang chạy "replica" tại bất kỳ thời điểm nào. Nói cách khác, 1 replica đảm bảo 1 service luôn luôn có các container đang chạy, dù có chuyện gì xảy ra. Nếu có quá nhiều container, swarm sẽ kill đi 1 số, nếu có quá ít, swarm sẽ bật thêm.

Tạo 1 service với số replica là 1 :
```sh
docker service create \
   --replicas 1 \
   --name nodejs \
   --publish 80:8080 \
   --constraint 'node.role==worker' \
   kalise/nodejs-web-app:latest

docker service ps nodejs
ID            NAME      IMAGE                         NODE     DESIRED STATE  CURRENT STATE           ERROR  PORTS
faapm76m2wgd  nodejs.1  kalise/nodejs-web-app:latest  swarm01  Running        Running 13 seconds ago

docker service inspect --pretty nodejs
```

```sh
ID:             9xcryw8i2i2wvslejlj6yjwry
Name:           nodejs
Service Mode:   Replicated
 Replicas:      1
Placement:Contraints:   [node.role==worker]
UpdateConfig:
 Parallelism:   1
 On failure:    pause
 Max failure ratio: 0
ContainerSpec:
 Image:         kalise/nodejs-web-app:latest
Resources:
Endpoint Mode:  vip
Ports:
 PublishedPort 80
  Protocol = tcp
  TargetPort = 8080
```

Service có số replica là 1 nghĩa là nó sẽ luôn có 1 container. Trong trường hợp container hoặc worker... bị fail, Swarm đảm bảo rằng luôn có 1 container đang chạy.

Để kiểm tra, hãy kill container đang chạy. Login vào worker node nơi mà container đang chạy và kill nó.
```sh
docker rm -f nodejs.1.faapm76m2wgdubylakmjnkwge
nodejs.1.faapm76m2wgdubylakmjnkwge

docker ps
CONTAINER ID        IMAGE   
```

Quay trở lại manager node và kiểm tra status của service 
```sh
docker service ps nodejs
ID            NAME          IMAGE                         NODE     DESIRED STATE  CURRENT STATE          ERROR                        PORTS
lmeet488s78q  nodejs.1      kalise/nodejs-web-app:latest  swarm02  Running        Running 2 minutes ago
faapm76m2wgd   \_ nodejs.1  kalise/nodejs-web-app:latest  swarm01  Shutdown       Failed 3 minutes ago   "task: non-zero exit (137)"
```

Có thể thấy swarm phát hiện ra container bị fail trên node `swarm01` và sau đó start 1 container mới trên node `swarm02` để đúng với số replica đã thiết lâp.

### 7. Service network

Mô hình overlay network cho phép swarm tạo 1 network layout phức tạp hơn. Trong phần này, chúng ta sẽ triển khai 1 service trên 1 custom internal network. Điều này có ích khi chúng ta tạo các ứng dụng multilayer được tạo bởi các service khác nhau. Các service này kết nối với nhau thông qua internal network 

Tạo 1 overlay network mới với tên là `internal`
```sh
docker network create --driver=overlay --subnet=172.30.0.0/24 --attachable internal

docker network list
NETWORK ID          NAME                DRIVER              SCOPE
6e035fb63823        bridge              bridge              local
e73fde81ef50        docker_gwbridge     bridge              local
f0c9ed46f0b6        host                host                local
1qc6vhwhaeqn        ingress             overlay             swarm
kq7sc5qu1wle        internal            overlay             swarm
eaef890efed3        none                null                local
```

Optio `--attachable` cho phép gán container bằng tay trên network này. Chúng ta dùng chúng chỉ để demo, thường không áp dụng trong thực tế.

Inspect internal network vừa tạo :
```sh
[
    {
        "Name": "internal",
        "Id": "kq7sc5qu1wlelrat8wnqob2g2",
        "Created": "0001-01-01T00:00:00Z",
        "Scope": "swarm",
        "Driver": "overlay",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.30.0.0/24",
                    "Gateway": "172.30.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": true,
        "Containers": null,
        "Options": {
            "com.docker.network.driver.overlay.vxlanid_list": "4097"
        },
        "Labels": null
    }
]
```

Sau đó start nodejs web service trên network này 
```sh
docker service create \
       --replicas 1 \
       --name nodejs \
       --network internal \
       --constraint 'node.role==worker' \
       kalise/nodejs-web-app:latest
```

Kiểm tra service đang running 
```sh
docker service ps nodejs
ID            NAME      IMAGE                         NODE     DESIRED STATE  CURRENT STATE          ERROR  PORTS
2ceiqgqalqnu  nodejs.1  kalise/nodejs-web-app:latest  swarm01  Running        Running 5 minutes ago
```

Login tới worker node nơi service đang chạy. Inspect internal network, có thể thấy service container được gán tới network : 
```sh
...
        "Internal": false,
        "Attachable": false,
        "Containers": {
            "9b2e3c6c9bcb052a61263eaf696c7352ae33a95e441c604e9929e74d85d0ea1e": {
                "Name": "nodejs.1.o6fo1nik5sh23ju7f3d6n65ki",
                "EndpointID": "1e8471d06db53c6c789189e716af3c870093814ad62d29b4d6b5dba9da6fc4e3",
                "MacAddress": "02:42:ac:1e:00:03",
                "IPv4Address": "172.30.0.3/24",
                "IPv6Address": ""
            }
        },
        "Options": {
            "com.docker.network.driver.overlay.vxlanid_list": "4097"
        },
        "Labels": {},
...
```

Mặc dù service container được gán tới gateway bridge network khi nó có thể truy cập được tới mạng ngoài thông qua NAT theo mô hình networking docker.
```sh
...
        "Internal": false,
        "Attachable": false,
        "Containers": {
            "9b2e3c6c9bcb052a61263eaf696c7352ae33a95e441c604e9929e74d85d0ea1e": {
                "Name": "gateway_9b2e3c6c9bcb",
                "EndpointID": "1a1a6e3f2f27944f9af8980f6bfa26ff8d1a38b4641ac7368d966055f7245671",
                "MacAddress": "02:42:ac:12:00:03",
                "IPv4Address": "172.18.0.3/16",
                "IPv6Address": ""
            },
            "ingress-sbox": {
                "Name": "gateway_ingress-sbox",
                "EndpointID": "686c50270cbd251dd02acc73ffabdce77067ecf5dcf9e6e01552d2ca1eef5a65",
                "MacAddress": "02:42:ac:12:00:02",
                "IPv4Address": "172.18.0.2/16",
                "IPv6Address": ""
            }
        },
        "Options": {
            "com.docker.network.bridge.enable_icc": "false",
            "com.docker.network.bridge.enable_ip_masquerade": "true",
            "com.docker.network.bridge.name": "docker_gwbridge"
        },
        "Labels": {}
...
```

Nodejs service chỉ có thể được kết nối bởi các service khác cùng chạy trên internal network. Tạo 1 service khác lá busybox trên cùng inernal network :
```sh
docker service create \
  --name busybox \
  --network internal \
  --constraint 'node.role==worker'  \
busybox:latest sleep 3000
```

Kiểm tra busybox service đang chạy trên container nào :
```sh
docker service ps busybox
ID            NAME       IMAGE           NODE     DESIRED STATE  CURRENT STATE
1swg9p3myu38  busybox.1  busybox:latest  swarm02  Running        Running 10 seconds ago
```

Login vào busybox container và truy cập tới nodejs service
```sh
docker exec -it busybox.1 sh
/ #

/ # wget 172.30.0.3:8080 -O -
Connecting to 172.30.0.3:8080 (172.30.0.3:8080)
<html><head></head><body>Hello World! from 172.30.0.3</body></html>
```

Tuy nhiên service không thể kết nối được từ bên ngoài bởi service không được expose. Để có thể truy cập từ bên ngoài, login tới manager node và expose service tới một host port.
```sh
docker service update nodejs --publish-add 80
```

Login tới worker node nơi service đang chạy. Inspect ingress network, có thể thấy service container được gán với network đó.
```sh
...
        "Internal": false,
        "Attachable": false,
        "Containers": {
            "9b2e3c6c9bcb052a61263eaf696c7352ae33a95e441c604e9929e74d85d0ea1e": {
                "Name": "nodejs.1.o6fo1nik5sh23ju7f3d6n65ki",
                "EndpointID": "ddf9d3c61e0ea8d112006ea20caf70050a71cd20609e1b216f29baf7cdd6cf98",
                "MacAddress": "02:42:0a:ff:00:08",
                "IPv4Address": "10.255.0.8/16",
                "IPv6Address": ""
            },
            "ingress-sbox": {
                "Name": "ingress-endpoint",
                "EndpointID": "5eaa13364c6edbd1541f4401aa1ef275d2df7f659bda6e39647745a3f17e9c5d",
                "MacAddress": "02:42:0a:ff:00:06",
                "IPv4Address": "10.255.0.6/16",
                "IPv6Address": ""
            }
        },
        "Options": {
            "com.docker.network.driver.overlay.vxlanid_list": "4096"
        },
...
```

Vì lý do đó chúng ta yêu cầu swarm expose service với port public là 80. Khi 1 request của user gọi tới ingress sandbox, chúng sẽ chuyển hướng request tới service container thông qua ingress network như vd trước :
```sh
wget swarm01:80 -O -
...
<html><head></head><body>Hello World! from 10.255.0.8</body></html>
```

Nếu muốn giấu service container, chỉ cần detach service từ port của host
```sh
docker service update nodejs --publish-add 80
```

### 8. Service discovery
Với mô hình docker network chỉ dung single host, docker swarm sử dụng embedded DNS để cung cấp service discovery cho các container đang chạy trên swarm. Docker engine có embedded DNS server `nameserver 127.0.0.11` cung cấp phân giải tên miền tới tất cả các container. Mỗi container có 1 bộ phân giải DNS để forwarnd các DNS query tới engine, được coi như là 1 DNS server. Docker engine sau đó kiểm tra xem DNS query thuộc về 1 container hoặc service trên mỗi network mà request container thuộc về. Nếu có thì Docker Engine tìm kiếm các IP cỉa container hoặc VIP liên quan tới service. Sau đó chúng gửi câu trả lời lại với nơi gửi request. Trong trường hợp với service, VIP được dùng cho các request load balancing tới các container replica cho việc cung cấp service.

Service discovery là network-scope, điều này có nghĩa là chỉ các container trên cùng network (bao gồm các host khác nhau) có thể sử dụng được tính năng embedded DNS. Các container không cùng 1 dải network không thể phân giải các địa chỉ khác. Nếu container đích hoặc service đích và source container không cùng mạng, Docker Engine sẽ chỏ DNS query tới external DNS server.

Để test service discovery và load balancing, tạo 1 internal overlay network khi service discovery không hoạt động trên ingress overlay network.
```sh
docker network create --driver=overlay --subnet=172.32.0.0/24 internal
```

Sau đó tạo 1 nodejs web service với 2 replica và attach chúng vào mạng trên
```sh
docker service create  \
   --replicas 2 \
   --name nodejs \
   --publish 80:8080  \
   --network internal  \
   --constraint 'node.role==worker'  \
docker.io/kalise/nodejs-web-app:latest
```

Tạo các service khác như `busybox`, trên cùng mạng :
```sh
docker service create \
  --name busybox \
  --network internal \
  --constraint 'node.role==worker'  \
busybox:latest sleep 3000
```

Kiểm tra busybox service đang chạy trên container nào
```sh
docker service ps busybox
ID            NAME       IMAGE           NODE     DESIRED STATE  CURRENT STATE
1swg9p3myu38  busybox.1  busybox:latest  swarm02  Running        Running 10 seconds ago
```

Login vào busybox container và kiểm tra việc phân giải service
```sh
docker exec -it busybox.1.1swg9p3myu38zzh1wmw8vylrd sh

/ # cat /etc/resolv.conf
search clastix.io
nameserver 127.0.0.11
options ndots:0

/ # ping nodejs
PING nodejs (172.32.0.2): 56 data bytes
64 bytes from 172.32.0.2: seq=0 ttl=64 time=0.085 ms
64 bytes from 172.32.0.2: seq=1 ttl=64 time=0.116 ms
^C
--- nodejs ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max = 0.085/0.100/0.116 ms

/ # exit
```

Request cho `nodejs` host được phân giải bởi embedded DNS server `nameserver 127.0.0.11` với VIP `172.32.0.2` của nodejs service. VIP của service này là load balancing một cách nội bộ tới từng IP container. Container name được phân giải, mặc dù được chỏ trực tiếp tới IP của họ. External hostname như `docker.com` không tồn tại như là 1 service name trên mạng internal, vì vậy reuqest được chỏ tới external DNS server mặc định trên host.

Traffic tới VIP được tự động gửi tới tất cả container relica đang chạy thông qua overlay network. Cách tiếp cận này tránh cho bất kỳ client nào phải thực hiện việc cân bằng tải bởi vì chỉ 1 IP được trả về tới client. Docker engine lo liệu toàn bộ việc định tuyến và phân tán traffic một cách cân bằng tơi service container khỏe mạnh.

Để lấy VIP của 1 service :
```sh
docker service inspect nodejs
```

```sh
...
            "Spec": {
                "Mode": "vip",
                "Ports": [
                    {
                        "Protocol": "tcp",
                        "TargetPort": 8080,
                        "PublishedPort": 80,
                        "PublishMode": "ingress"
                    }
                ]
            },
            "Ports": [
                {
                    "Protocol": "tcp",
                    "TargetPort": 8080,
                    "PublishedPort": 80,
                    "PublishMode": "ingress"
                }
            ],
            "VirtualIPs": [
                {
                    "NetworkID": "vcvqu78nxipjwqmr1ew170zzi",
                    "Addr": "10.255.0.5/16"
                },
                {
                    "NetworkID": "wh11rcyqwqgeqzfw7dfi19zrh",
                    "Addr": "172.32.0.2/24"
                }
            ]
...
```

### 9. Service Load Balancing

Với Routing mesh, swarm có thể expose service tới mỗi node của cluster. Tuy nhiên, việc load balancing chỉ giới hạn tới các cluster node. Người dùng có thể cấu hình một ứng dụng load balancer bên ngoài, ví dụ như HAProxy, để định tuyến các request từ web tới một service được publish port 80.

VD bạn có thể có cấu hình HAProxy như sau :
```sh
global
        log /dev/log    local0
        log /dev/log    local1 notice
...snip...

# Configure HAProxy to listen on port 80
frontend http_front
   bind *:80
   stats uri /haproxy?stats
   default_backend http_back

# Configure HAProxy to route requests to swarm nodes on port 80
backend http_back
   balance roundrobin
   server swarm00 10.10.10.60:80 check
   server swarm01 10.10.10.61:80 check
   server swarm02 10.10.10.62:80 check
```

Khi user truy cập tới HAProxy load balancer trên port 80, nó chỏ các reuqest tới các node trên swarm. Swarm routing mesh sẽ định tuyến các request tới các container thông qua IPV load balancer nội bộ. Nếu bất kỳ lý do gì, swarm scheduler gửi container tới nột node khác, admin không cần phải cấu hình lại load balancer.

### 10. Cluster High Avaibility

Swarm cluster với 1 manager node thì sẽ có nhược điểm là SPOF (single point of failure). Khi node manager có sự cố, các service của người dùng vẫn hoạt động được trên worker node nhưng admin không thể kiểm soát được cluster 

Một số lẻ của maanger trong swarm cluster được yêu cầu để hỗ trợ các manager node bị fail. Có 1 số lẻ để đảm bảo rằng trong thời gian bị gián đoạn, cluster quorum vẫn có thể sẵn sàng xử lý các request khi sự cố xảy ra và cluster được chia thành 2 phần riêng biệt. Ngóm swarm cluster có thể chịu được số node bị fail là (N-1)/2 và yêu cầu đa số hoặc quorum của (N/2)+1 member đồng ý về các giá trị được đề xuất cho cluster.

Theo thuật toán **Raft**, swarm cluster start 1 tiến trình quản lý leader khi cluster được vào form. Leader có trách nhiệm giữ tất cả các thay đổi trong hệ thống bằng cách đồng ý việc phân phối giữa các manager node khác.

Với single manager, chúng ta chỉ có 1 leader :
```sh
docker node list
ID                           HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
b07ejkr444dm5bsbmey4hrd2r *  swarm00   Ready   Active        Leader
q5wjauy7i4fr7ictcsu2wkl68    swarm01   Ready   Active        
r8zwpsx1rczpmoxk11i6fuk21    swarm02   Ready   Active        
```

Bằng việc thúc đẩy các node khác trở thành master node, chúng ta có các leader node khác :
```sh
docker node promote swarm01 swarm02

docker node list
ID                           HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
b07ejkr444dm5bsbmey4hrd2r *  swarm00   Ready   Active        Leader
q5wjauy7i4fr7ictcsu2wkl68    swarm01   Ready   Active        Reachable
r8zwpsx1rczpmoxk11i6fuk21    swarm02   Ready   Active        Reachable
```

Khi leader node gần nhất bị down vì 1 lý do nào đó, thì sẽ có việc bầu chọn leader node mới :
```sh
systemctl restart docker

docker node list
ID                           HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
b07ejkr444dm5bsbmey4hrd2r *  swarm00   Ready   Active        Reachable
q5wjauy7i4fr7ictcsu2wkl68    swarm01   Ready   Active        Reachable
r8zwpsx1rczpmoxk11i6fuk21    swarm02   Ready   Active        Leader
```

Không có giới hạn về số lượng của các manager node. Số node manager được cân bằng giữa hiệu năng và khả năng chịu lỗi. 