## I. Network teaming trên RHEL
### 1. Hiểu về network teaming
Việc kết hợp hay gộp chung các đường mạng để nhằm cung cáo một đường logic với mục đích thu được **network throughput**(1) cao hơn, hoặc cung cấp một **redundancy**(2), được biết với nhiều tên như "channel bonding", "Ethernet bonding", "port trunking", "channel teaming", "NIC teaming", "link aggregation"... Các định nghĩa này được hiểu chung trong Linux là "bonding". Thuật ngữ "Network Teaming" được coi là một cách biểu diễn mới, đưa ra như một sự lựa chọn và không thay thế bonding. Các driver bonding đã tồn tại sẽ không bị ảnh hưởng.

Network teaming (Team), được cung cấp một driver kernel nhỏ để kiểm soát nhanh các packet flow, và các ứng dụng user-space đa dạng. Driver có một API (Tean Netlink API), biểu diễu các giao tiếp Netlink. Các ứng dụng user-space có thể dùng API này để giao tiếp với driver. Một thư viện (lib), được cung cấp để user-space bao phủ các giao tiếp Team Netlink và RT Netlink messages. Một ứng dụng nền **teamd**, sử dụng Libteam **lib**. Một instance của **teamd** có thể kiểm soát một instance của Team driver. **Runner**, tiến trình nền phụ trách việc load-balancing và active-backup logic như là round-robin. 

**Teamdctl** cung cấp công cụ để kiểm soát instance của teamd đang sử dụng D-bus.

Team Netlink API giao tiếp với các ứng dụng user-space sử dụng Netlink messages. Thư viện user-space **libteam** không tương tác trực tiếp với API, mà dùng libnl hoặc teamnl để tương tác với driver API.

Tóm lại, các instance của Team driver, đang chạy trong kernel, không được cấu hình hoặc kiểm soát trực tiếp. Tất cả cấu hình được hoàn thành với sự giúp đỡ của các ứng dụng user-space, như là **teamd**. Ứng dụng sau đó sẽ tương tác trực tấp với một phần kernel driver.

#### 2. Hiểu về trạng thái mặc định của các interface Master và Slave.

Khi kiểm soát các teamd port interface sử dụng tiến trình NetworkManager, đặc biệt là trong quá trình tìm lỗi, hãy lưu ý các điều sau : 

 - **1**. Khởi động interface master sẽ không tự động khởi động port interface.
 - **2**. Việc start port interface luôn luôn dẫn tới việc start interface master
 - **3**. Việc stop master interface cũng dẫn tới việc stop port interface.
 - **4**. Một master không có port có thể start các kết nối IP tĩnh.
 - **5**. Một master không có port sẽ chờ các port khi khởi động DHCP connection.
 - **6**. Một master với DHCP connection sẽ chờ các port hoàn chỉnh khi một port với một carrier được thêm vào.
 - **7**. Một master với DHCP connection sẽ chờ các port sẽ tiếp tục chờ khi một port không có carrier được thêm vào.
 
**Chú ý** : Việc dùng các kết nối cap trực tiếp mà không dùng switch sẽ không được hỗ trợ bởi teaming.

#### 3. So sánh Network Teaming với Bonding

|Tính năng|Bonding|Team|
|---------|-------|----|
|broadcast Tx policy|Yes|Yes|
|round-robin Tx policy	|Yes|Yes|
|active-backup Tx policy|Yes|Yes|
|LACP (802.3ad) support	|Yes (passive only)|Yes|
|Hash-based Tx policy	|Yes|Yes|
|User can set hash function	|No|Yes|
|Tx load-balancing support (TLB)|Yes|Yes|
|LACP hash port select	|Yes|Yes|
|load-balancing for LACP support||No	|Yes|
|Ethtool link monitoring|Yes|Yes|
|ARP link monitoring|Yes|Yes|
|NS/NA (IPv6) link monitoring|No	|Yes|
|ports up/down delays	|Yes|Yes|
|port priorities and stickiness (“primary” option enhancement)|No	|Yes|
|separate per-port link monitoring setup|No	|Yes|
|multiple link monitoring setup	|Limited|Yes|
|lockless Tx/Rx path	|No (rwlock)|Yes (RCU)|
|VLAN support	|Yes	|Yes|
|user-space runtime control	|Limited|Full|
|Logic in user-space	|No	|Yes|
|Extensibility	|Hard	|Easy|
|Modular design	|No	|Yes|
|Performance overhead|Low|Very Low|
|D-Bus interface|No	|Yes|
|multiple device stacking|Yes|Yes|
|zero config using LLDP	|No	|(in planning)|
|NetworkManager|support|Yes|Yes|

#### 4. Hiểu về Network teaming deamon và "runner"

Tiến trình **teamd**, sử dụng **libteam** để kiểm soát mỗi instance của team driver. Instance này thêm các instance của driver thiết bị phần cứng thành một **team**. Một team driver biểu diễn một network interface, ví dụ như "team0". Interface được tạo từ bởi các instance của team driver được gán tên như là team0, team1...

Logic chung cho tất cả các phương pháp teaming được tạo bởi **teamd**. Các function này là độc nhất đối với các phương thức sharing và backup khác nhau, chẳng hạn round-robin, các funtion được thực hiện bởi các đơn vị mã riêng biệt, được gọi là **runner**.

Một số runner có sẵn trong thời gian ghi : 
 -	broadcast (data được truyền qua tất cả các port)
 -	round-robin (data được truyền qua tất cả các port theo vòng)
 -	active-backup (một port hoặc đường dẫn được dùng trong khi phần còn lại được dùng làm backup)
 -	loadbalance (active với Tx load-balancing và BPF-based Tx port selector)
 -	lacp (biểu diễn 802.3ad Link aggregation control protocol)

Ngoài ra, có một số link-watcher sẵn sàng : 
 -	ethtool : Libteam lib dùng ethtool để theo dõi sự thay đổi của trạng thái link.
 -	arp-ping : được dùng để giám sát các địa chỉ phần cứng ở xa dùng ARP packet.
 -	nsna-ping (Neighbor Advertisements and Neighbor Solicitation) : từ giao thức IPV6 Neighbor Discovery dùng để giám sát các địa chỉ hàng xóm.

**Chú ý** : Khi chạy lacp runner, nên dùng ethtool làm link-watcher. 

#### 5. Cài đặt và cấu hình teaming
#### 5.1 Cài đặt tiến trình network teaming.

Sử dụng quyền root, cài đặt teamd : 

```sh
yum install teamd
```

#### 5.2 Chọn các interface để dùng như các port trong network team
Để xem các interface sẵn sàng, dùng câu lệnh sau :

```sh
ip link show
1: lo:  <LOOPBACK,UP,LOWER_UP > mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: em1:  <BROADCAST,MULTICAST,UP,LOWER_UP > mtu 1500 qdisc pfifo_fast state UP mode DEFAULT qlen 1000
    link/ether 52:54:00:6a:02:8a brd ff:ff:ff:ff:ff:ff
3: em2:  <BROADCAST,MULTICAST,UP,LOWER_UP > mtu 1500 qdisc pfifo_fast state UP mode DEFAULT qlen 1000
link/ether 52:54:00:9b:6d:2a brd ff:ff:ff:ff:ff:ff
```
## 6. Các phương thức tạo network team.
Có cách phương thức tạo network team như sau : 

 -	Sử dụng công cụ text user interface của NetworkManager - **nmtui**.
 -	Sử dụng command-line tool - **nmcli**
 -	Sử dụng Team daemon - **teamd**.
 -	Sử dụng file cấu hình,
 -	Sử dụng giao người dùng bằng đồ họa.
 
Tôi sẽ hướng dẫn tạo team bằng 2 phương thức đó là sử dụng **nmcli** và file cấu hình.

#### 6.1 Tạo team bằng nmcli
#### Step1 : Thực hiện show các thiết bị sẵn sàng trên hệ thống, dùng câu lệnh : 

```sh
~]$ nmcli connection show

NAME  UUID                                  TYPE            DEVICE
eth1  0e8185a1-f0fd-4802-99fb-bedbb31c689b  802-3-ethernet  --
eth0  dfe1f57b-419d-4d1c-aaf5-245deab82487  802-3-ethernet  --
```
#### Step2 : Tạo một team interface mới, với tên là team-ServerA :

```sh
~]$ nmcli connection add type team con-name team0 ifname team0
Connection 'team0' (b954c62f-5fdd-4339-97b0-40efac734c50) successfully added.
```
NetworkManager sẽ tạo một file cấu hình trong : `/etc/sysconfig/network-scripts/ifcfg-team0`.

Để xem các giá trị khác được gán, dùng câu lệnh :

```sh
~]$ nmcli con show team-ServerA

connection.id:                          team0
connection.uuid:                        b954c62f-5fdd-4339-97b0-40efac734c50
connection.interface-name:              ServerA
connection.type:                        team
connection.autoconnect:                 yes
…
ipv4.method:                            auto
[output truncated]
```
#### Step3. Thêm interface eth0 vào team0, với tên là team0-port1 

```sh
~]$ nmcli con add type team-slave con-name team0-port1 ifname eth0 master team0
Connection 'team0-port1' (ccd87704-c866-459e-8fe7-01b06cf1cffc) successfully added.
```

Tương tự, add eth1 

```sh
~]$ nmcli con add type team-slave con-name team0-port2 ifname eth1 master team0
Connection 'team0-port2' (a89ccff8-8202-411e-8ca6-2953b7db52dd) successfully added.
```

#### Step4. Khởi động team và các port của team.

```sh
~]$ nmcli connection up team0-port1
Connection successfully activated (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/12)
~]$ nmcli connection up team0-port2
Connection successfully activated (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/13)
~]$ nmcli connection up team0
Connection successfully activated (master waiting for slaves) (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/16)

