# Hướng dẫn cài đặt Openstack Mitaka bằng packstack với mô hình Network Bonding

# 1. Chuẩn bị

 -	Distro : RHEL 7 đã register

 -	Mô hình

![ops](/ManhDV/OpenStack/images/ops-bonding.png)

 -	IP Planing

![ops](/ManhDV/OpenStack/images/ipplan.png)

## 2. Cài đặt và cấu hình Bonding (Thực hiện trên 2 node CTL và COM)

### 2.1. Kiểm tra mô hình bonding
 
![ops](/ManhDV/OpenStack/images/cardvm.png) 

Dùng 4 card mạng, chia thành 2 cặp, NIC1 và NIC2 chọn VMnet1 (Host-only), NIC3 và NIC4 chọn VMnet8 (NAT).

 -	Phân bố các NIC
	-	bond0 : eno16777736 & eno33554960, sử dụng VMnet1
	-	bond1 : eno50332184 & eno67109408, sử dụng VMnet8 (NAT)

**Lưu ý** : Trong quá trình thêm card mới, thêm lần lượt từng card một, không add nhiều card 1 lúc, sẽ dễ gây hiện tượng nhảy card mạng.
	
 -	Kiểm tra trạng thái card mạng
 
Trong quá trình cài đặt OS, nếu không bật chế độ **Connected** cho các NIC, thì các file cấu hình sẽ không tự sinh ra, mà ta phải tạo file cấu hình cho các NIC bằng lệnh. Trước tiên, kiểm tra trạng thái của các NIC và xem các NIC đã có file cấu hình hay chưa.

 -	Kiểm tra xem đã có file cấu hình cho NIC hay chưa
 
	```sh
	ls -alh /etc/sysconfig/network-scripts/
	```
	
![ops](/ManhDV/OpenStack/images/nic-fileconfig.png)  
 
 -	Kiểm tra trạng thái các NIC

	```sh
	nmcli device status
	```
	
![ops](/ManhDV/OpenStack/images/nic-status.png) 

Khi cài đặt OS, tôi đã chọn chế độ **Connected** cho cả 4 NIC, vì vậy có đủ 4 file cấu hình cho 4 NIC. Nếu chưa chọn chế độ **Connected**, thực hiện lệnh tạo file cấu hình còn thiếu cho các NIC.

```sh
nmcli device connect enoXXX
```
	
Thay thế enoXXX với tên NIC chưa có file cấu hình. Kiểm tra lại xem file cấu hình đã sinh. 

 - Các bước cấu hình
	-	Thực hiện lệnh để nạp chế độ bonding cho OS trên các máy cần cấu hình.
	
	```sh
	modprobe bonding
	```

	-	Kiểm tra xem chế độ bonding đã được hay chưa.
	```sh
	modinfo bonding
	```
	
	-	Kết quả sẽ hiển thị như sau, trong đó chưa dòng `description:    Ethernet Channel Bonding Driver, v3.7.1`
	
	```sh
	filename:       /lib/modules/3.10.0-327.el7.x86_64/kernel/drivers/net/bonding/bonding.ko
	author:         Thomas Davis, tadavis@lbl.gov and many others
	description:    Ethernet Channel Bonding Driver, v3.7.1
	version:        3.7.1
	license:        GPL
	alias:          rtnl-link-bond
	rhelversion:    7.2
	srcversion:     5ACA30A256544B84A47E606
	depends:
	intree:         Y
	vermagic:       3.10.0-327.el7.x86_64 SMP mod_unload modversions
	signer:         Red Hat Enterprise Linux kernel signing key
	sig_key:        BC:73:C3:CE:E8:9E:5E:AE:99:4A:E5:0A:0D:B1:F0:FE:E3:FC:09:13
	sig_hashalgo:   sha256
	parm:           max_bonds:Max number of bonded devices (int)
	parm:           tx_queues:Max number of transmit queues (default = 16) (int)
	parm:           num_grat_arp:Number of peer notifications to send on failover event (alias of num_unsol_na) (int)
	parm:           num_unsol_na:Number of peer notifications to send on failover event (alias of num_grat_arp) (int)
	parm:           miimon:Link check interval in milliseconds (int)
	parm:           updelay:Delay before considering link up, in milliseconds (int)
	parm:           downdelay:Delay before considering link down, in milliseconds (int)
	parm:           use_carrier:Use netif_carrier_ok (vs MII ioctls) in miimon; 0 for off, 1 for on (default) (int)
	parm:           mode:Mode of operation; 0 for balance-rr, 1 for active-backup, 2 for balance-xor, 3 for broadcast, 4 for 802.3ad, 5 for balance-tlb, 6 for balance-alb (charp)
	parm:           primary:Primary network device to use (charp)
	parm:           primary_reselect:Reselect primary slave once it comes up; 0 for always (default), 1 for only if speed of primary is better, 2 for only on active slave failure (charp)
	parm:           lacp_rate:LACPDU tx rate to request from 802.3ad partner; 0 for slow, 1 for fast (charp)
	parm:           ad_select:803.ad aggregation selection logic; 0 for stable (default), 1 for bandwidth, 2 for count (charp)
	parm:           min_links:Minimum number of available links before turning on carrier (int)
	parm:           xmit_hash_policy:balance-xor and 802.3ad hashing method; 0 for layer 2 (default), 1 for layer 3+4, 2 for layer 2+3, 3 for encap layer 2+3, 4 for encap layer 3+4 (charp)
	parm:           arp_interval:arp interval in milliseconds (int)
	parm:           arp_ip_target:arp targets in n.n.n.n form (array of charp)
	parm:           arp_validate:validate src/dst of ARP probes; 0 for none (default), 1 for active, 2 for backup, 3 for all (charp)
	parm:           arp_all_targets:fail on any/all arp targets timeout; 0 for any (default), 1 for all (charp)
	parm:           fail_over_mac:For active-backup, do not set all slaves to the same MAC; 0 for none (default), 1 for active, 2 for follow (charp)
	parm:           all_slaves_active:Keep all frames received on an interface by setting active flag for all slaves; 0 for never (default), 1 for always. (int)
	parm:           resend_igmp:Number of IGMP membership reports to send on link failure (int)
	parm:           packets_per_slave:Packets to send per slave in balance-rr mode; 0 for a random slave, 1 packet per slave (default), >1 packets per slave. (int)
	parm:           lp_interval:The number of seconds between instances where the bonding driver sends learning packets to each slaves peer switch. The default is 1. (uint)
	```
	
### 2.2. Cấu hình bond0
 -	Bước 1 : Tạo bond0 cho 2 interface eno16777736 & eno33554960
	```sh
	cat << EOF> /etc/sysconfig/network-scripts/ifcfg-bond0
	DEVICE=bond0
	TYPE=Bond
	NAME=bond0
	BONDING_MASTER=yes
	BOOTPROTO=none
	ONBOOT=yes
	IPADDR=192.168.11.10
	NETMASK=255.255.255.0
	BONDING_OPTS="mode=1 miimon=100"
	EOF
	```
	
 -	Bước 2 : Sửa file cấu hình các interface thuộc bond0
	-	Sao lưu file cấu hình interface eno16777736
	
	```sh
	cp /etc/sysconfig/network-scripts/ifcfg-eno16777736 /etc/sysconfig/network-scripts/ifcfg-eno16777736.orig
	```
	
	-	Sửa dòng với giá trị mới nếu đã có dòng đó và thêm các dòng nếu thiếu trong file /etc/sysconfig/network-scripts/ifcfg-eno16777736
		
	```sh
	BOOTPROTO=none
	ONBOOT=yes
	MASTER=bond0
	SLAVE=yes
	```
		
	-	Sao lưu file cấu hình của interface eno33554952
	```sh
	cp /etc/sysconfig/network-scripts/ifcfg-eno33554952 /etc/sysconfig/network-scripts/ifcfg-eno33554952.orig
	```
	
	-	Sửa dòng với giá trị mới nếu đã có dòng đó và thêm các dòng nếu thiếu trong file /etc/sysconfig/network-scripts/ifcfg-eno33554952
	
	```sh
	BOOTPROTO=none
	ONBOOT=yes
	MASTER=bond0
	SLAVE=yes
	```
	
	-	Khởi động lại network sau khi cấu hình bond0
	
	```sh
	nmcli con reload
	systemctl restart network
	```
	-	Kiểm tra với câu lệnh `ip a` sẽ thấy `bond0` đã có địa chỉ IP và state `UP`
	
### 2.3. Cấu hình bond1
 -	Bước 1 : Tạo bond1 cho 2 interface eno50332184 & eno67109408
	```sh
	cat << EOF> /etc/sysconfig/network-scripts/ifcfg-bond1
	DEVICE=bond1
	TYPE=Bond
	NAME=bond1
	BONDING_MASTER=yes
	BOOTPROTO=none
	ONBOOT=yes
	IPADDR=172.16.69.10
	NETMASK=255.255.255.0
	GATEWAY=172.16.69.1
	DNS=8.8.8.8
	BONDING_OPTS="mode=1 miimon=100"
	EOF
	```
	
 -	Bước 2 : Sửa file cấu hình các interface thuộc bond1
	-	Sao lưu file cấu hình interface eno50332184
	
	```sh
	cp /etc/sysconfig/network-scripts/ifcfg-eno50332184 /etc/sysconfig/network-scripts/ifcfg-eno50332184.orig
	```
	
	-	Sửa dòng với giá trị mới nếu đã có dòng đó và thêm các dòng nếu thiếu trong file /etc/sysconfig/network-scripts/ifcfg-eno50332184
		
	```sh
	BOOTPROTO=none
	ONBOOT=yes
	MASTER=bond1
	SLAVE=yes
	```
		
	-	Sao lưu file cấu hình của interface eno67109408
	```sh
	cp /etc/sysconfig/network-scripts/ifcfg-eno67109408 /etc/sysconfig/network-scripts/ifcfg-eno67109408.orig
	```
	
	-	Sửa dòng với giá trị mới nếu đã có dòng đó và thêm các dòng nếu thiếu trong file /etc/sysconfig/network-scripts/ifcfg-eno67109408
	
	```sh
	BOOTPROTO=none
	ONBOOT=yes
	MASTER=bond1
	SLAVE=yes
	```
	
	-	Khởi động lại network sau khi cấu hình bond1
	
	```sh
	nmcli con reload
	systemctl restart network
	```
	-	Kiểm tra với câu lệnh `ip a` sẽ thấy `bond1` đã có địa chỉ IP và state `UP`. Kiểm tra thông tin các bond với câu lệnh `cat /proc/net/bonding/bond0`, kết quả như sau :
	
	```sh
	Ethernet Channel Bonding Driver: v3.7.1 (April 27, 2011)

	Bonding Mode: fault-tolerance (active-backup)
	Primary Slave: None
	Currently Active Slave: eno16777736
	MII Status: up
	MII Polling Interval (ms): 100
	Up Delay (ms): 0
	Down Delay (ms): 0

	Slave Interface: eno16777736
	MII Status: up
	Speed: 1000 Mbps
	Duplex: full
	Link Failure Count: 0
	Permanent HW addr: 00:0c:29:d5:89:f4
	Slave queue ID: 0

	Slave Interface: eno33554960
	MII Status: up
	Speed: 1000 Mbps
	Duplex: full
	Link Failure Count: 0
	Permanent HW addr: 00:0c:29:d5:89:fe
	```
	
	-	Sử dụng lệnh watch với netstat, sau đó đứng từ một máy khác ping đến và quan sát kết quả.
	
	```sh
	watch -d -n1 netstat -i
	```
	
	-	Kết quả cuối cùng, 2 máy CTL và COM phải có 2 card bond0 và bond1 với cấu hình network như trên IP Plan.
	
## 3. Cài đặt Packstack trên node CTL

 -	Tắt firewall, NetworkManager và disable Selinux

	```sh
	systemctl disable firewalld
	systemctl stop firewalld
	systemctl disable NetworkManager
	systemctl stop NetworkManager
	systemctl enable network
	systemctl start network
	sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
	```

 -	Reboot lại máy CTL

	```sh
	init 6
	```

 -	Đăng ký tài khoản RHEL cho máy và upgrade (thêm `--force` nếu đã đăng ký)
 
	```sh
	subscription-manager register --username="manh02@meditech.vn" --password="meditech2017"  --auto-attach
	subscription-manager repos --enable=rhel-7-server-optional-rpms --enable=rhel-7-server-extras-rpms --enable=rhel-7-server-rh-common-rpms
	yum upgrade -y
	```
	
 -	Cài đặt Repo Mitaka, Byobu, Packstack

	```sh
	yum install -y https://repos.fedorapeople.org/repos/openstack/openstack-mitaka/rdo-release-mitaka-6.noarch.rpm 
	yum install -y http://dl.fedoraproject.org/pub/epel/7/x86_64/b/byobu-5.73-4.el7.noarch.rpm
	yum update -y
	yum install -y wget crudini
	yum install -y openstack-packstack
	```
 -	Reboot lại máy CTL
	```sh
	init 6
	```
	
 -	Cài đặt packstack
 
	```sh
	packstack --allinone \
		--default-password=Welcome123 \
		--os-cinder-install=y \
		--os-ceilometer-install=n \
		--os-trove-install=n \
		--os-ironic-install=n \
		--nagios-install=n \
		--os-swift-install=n \
		--os-gnocchi-install=n \
		--os-aodh-install=n \
		--os-neutron-ovs-bridge-mappings=extnet:br-ex \
		--os-neutron-ovs-bridge-interfaces=br-ex:bond1 \
		--os-neutron-ovs-bridges-compute=br-ex \
		--os-neutron-ml2-type-drivers=vxlan,flat \
		--os-controller-host=172.16.69.10 \
		--os-compute-hosts=172.16.69.20 \
		--os-neutron-ovs-tunnel-if=bond0 \
		--provision-demo=n
	```

 -	Chạy source admin
 
	```sh
	source ~/keystonerc_admin
	```

 -	Upload images

	```sh
	curl http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-x86_64-disk.img | glance \
	image-create --name='cirros' \
	--visibility=public \
	--container-format=bare \
	--disk-format=qcow2	
	```
	
 -	Tạo network public
 
	```sh
	neutron net-create external_network --provider:network_type flat \
	--provider:physical_network extnet  \
	--router:external \
	--shared
	```

 -	Tạo subnet trong network public
 
	```sh
	neutron subnet-create --name public_subnet \
	--enable_dhcp=True \
	--allocation-pool=start=172.16.69.30,end=172.16.69.50 \
	--gateway=172.16.69.1 external_network 172.16.69.0/24
	```
	
	
 -	Tạo network private
 
	```sh
	neutron net-create private_network
	neutron subnet-create --name private_subnet private_network 10.0.0.0/24 \
	--dns-nameserver 8.8.8.8
	```

 -	Tạo router và addd các interface

	```sh
	neutron router-create router
	neutron router-gateway-set router external_network
	neutron router-interface-add router private_subnet	
	```
 -	Chỉnh sửa file /etc/nova/nova.conf
	```sh
	[vnc]
	...
	vncserver_listen=172.16.69.10
	vncserver_proxyclient_address=172.16.69.10
	```
 -	Restart service
 
	```sh
	systemctl restart openstack-nova-api.service 
	systemctl restart openstack-nova-consoleauth.service openstack-nova-scheduler.service 
	systemctl restart openstack-nova-conductor.service openstack-nova-novncproxy.service
	```
 -	Sau khi thực hiện chỉnh sửa `nova.conf` trên cả CTL và COM, vào dashboard và tạo máy ảo.
## 4. Cài đặt packstack trên COM

 -	Tắt firewall, NetworkManager và disable Selinux

	```sh
	systemctl disable firewalld
	systemctl stop firewalld
	systemctl disable NetworkManager
	systemctl stop NetworkManager
	systemctl enable network
	systemctl start network
	sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
	```

 -	Reboot lại máy CTL

	```sh
	init 6
	```

 -	Đăng ký tài khoản RHEL cho máy và upgrade (thêm `--force` nếu đã đăng ký)
 
	```sh
	subscription-manager register --username="manh02@meditech.vn" --password="meditech2017"  --auto-attach
	subscription-manager repos --enable=rhel-7-server-optional-rpms --enable=rhel-7-server-extras-rpms --enable=rhel-7-server-rh-common-rpms
	yum upgrade -y
	```
	
 -	Cài đặt Repo Mitaka, Byobu, Packstack

	```sh
	yum install -y https://repos.fedorapeople.org/repos/openstack/openstack-mitaka/rdo-release-mitaka-6.noarch.rpm 
	yum install -y http://dl.fedoraproject.org/pub/epel/7/x86_64/b/byobu-5.73-4.el7.noarch.rpm
	yum update -y
	yum install -y wget crudini
	yum install -y openstack-packstack
	```
 -	Reboot lại máy CTL
	```sh
	init 6
	```
 -	Sau khi cài xong packstack trên CTL, chỉnh sửa file /etc/nova/nova.conf
	```sh
	[libvirt]
	...
	virt_type=kvm
	cpu_mode=none

	[vnc]
	...
	vncserver_proxyclient_address=172.16.69.20
	novncproxy_base_url=http://172.16.69.10:6080/vnc_auto.html
	```
 -	Restart service
	```sh
	systemctl restart openstack-nova-compute
	```