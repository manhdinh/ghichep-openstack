# Cài đặt TripleO trên Centos 7

Triển khai Triple O với mô hình đơn giản bao gồm : 
 - 01 node Director
 - 01 node Controller
 - 01 node Compute
 - 01 node Storage

## I. Mô hình cài đặt và IP-Planning

Mô hình cài đặt Triple O đơn giản 

![tripleo](/OpenStack/images/tripleo-04.png)

IP-PLANNING cho hệ thống

![tripleo](/OpenStack/images/tripleo-05.png)


## II. Cài đặt 

### 1. Chuẩn bị môi trường

### 1.1. Cài đặt DNS Server cho hệ thống

Director yêu cầu FQDN (fully qualified domain name) cho việc cài đặt và cấu hình. Vì vậy ta cần một DNS server nội bộ cho tripleO. Ở đây tôi sẽ dựng 1 DNS server tại 1 VPS riêng biệt với IP `10.10.30.52`.

Setup hostname cho DNS server 

```sh
hostnamectl set-hostname dns-server.example
```

Cài đặt dịch vụ DNS :

```sh
yum install bind bind-utils bind-chroot caching-nameserver -y
```

Stop và disable tạm thời dịch vụ

```sh
systemctl stop named
systemctl disable named
```

Copy các file cấu hình mặc định vào thư mục /var/named/chroot/etc/

```sh
cp -rpvf /usr/share/doc/bind-9.11.4/sample/etc/* /var/named/chroot/etc/
```

![tripleo](/OpenStack/images/tripleo-06.png)

Lưu ý : thư mục `bind` có thể thay đổi tùy vào phiên bản được cài đặt.

Copy các file liên quan tới zone vào thư mục mới :

```sh
cp -rpvf /usr/share/doc/bind-9.11.4/sample/var/named/* /var/named/chroot/var/named/
```

![tripleo](/OpenStack/images/tripleo-07.png)

Chỉnh sửa cấu hình file /var/named/chroot/etc/named.conf

```sh
cp /var/named/chroot/etc/named.conf /var/named/chroot/etc/named.conf.orig
rm -rf /var/named/chroot/etc/named.conf

cat << EOF >> /var/named/chroot/etc/named.conf
options {
        listen-on port 53 { 127.0.0.1; any; };
#       listen-on-v6 port 53 { ::1; };
        directory       "/var/named";
        dump-file       "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";
        allow-query     { localhost; any; };
        allow-query-cache { localhost; any; };
};
logging {
        channel default_debug {
                file "data/named.run";
                severity dynamic;
        };
};

view my_resolver {
        match-clients      { localhost; any; };
        recursion yes;
        include "/etc/named.rfc1912.zones";
};
EOF
```

Thêm file cấu hình `named.rfc1912.zones`. Trong đó dải cần thực hiện là `10.10.30.0/24`. DNS server là 10.10.30.52 và node `10.10.30.41` được gán tương ứng với tên miền `undercloud-director.example`.

```sh
cp /var/named/chroot/etc/named.rfc1912.zones /var/named/chroot/etc/named.rfc1912.zones.orig
rm -rf /var/named/chroot/etc/named.rfc1912.zones

cat << EOF >>
zone "example" IN {
        type master;
        file "example.zone";
        allow-update { none; };
};
zone "30.10.10.in-addr.arpa" IN {
        type master;
        file "example.rzone";
        allow-update { none; };
};
EOF 
```

Các `zone` có nội dung thông tin phải được thêm tại `/var/named/chroot/etc/named.rfc1912.zones`

```sh
cd /var/named/chroot/var/named/
cp -p named.localhost  example.zone
cp -p named.loopback example.rzone
chown root:named *
chown -R  named:named data
```

Thêm các file phân giải tên miền dành cho IP `10.10.30.41` với tên miền `undercloud-director.example`.

```sh
cat << EOF >> /var/named/chroot/var/named/example.zone
$TTL 1D
@       IN SOA   example. root (
                                        0       ; serial
                                        1D      ; refresh
                                        1H      ; retry
                                        1W      ; expire
                                        3H )    ; minimum
        IN NS   example.

                        IN A 10.10.30.52
undercloud-director     IN A 10.10.30.41
EOF

cat << EOF >> /var/named/chroot/var/named/example.rzone
$TTL 1D
@       IN SOA  example. root.example. (
                                        0       ; serial
                                        1D      ; refresh
                                        1H      ; retry
                                        1W      ; expire
                                        3H )    ; minimum

                IN NS   example.
41              IN PTR  undercloud-director.example.
EOF
```

Kiểm tra việc cấu hình đã thành công hay chưa : 

```sh
named-checkzone undercloud-director.example example.zone
named-checkzone undercloud-director.example example.rzone
named-checkconf -t /var/named/chroot/ /etc/named.conf
echo $?
```
![tripleo](/OpenStack/images/tripleo-08.png)

Kết quả như ảnh trên là đã thành công.



Thực hiện restart dịch vụ `named-chroot` :

```sh
systemctl restart named-chroot
```

Tại các node cần phân giải tên miền, trong file `/etc/resolv.conf` cần chứa cấu hình sau : 

```sh
search example
nameserver 10.10.30.52
```

Trong đó `10.10.30.52` là địa chỉ IP của  DNS server.

Tại node cần phân giải, kiểm tra việc phân giải tên miền như sau :

```sh
dig -x 10.10.30.41
```

Kết quả như sau : 

![tripleo](/OpenStack/images/tripleo-09.png)

Tiếp tục kiểm tra phân giải tên miền :

```sh
nslookup undercloud-director.example
```

![tripleo](/OpenStack/images/tripleo-10.png)

Như vậy là việc chuẩn bi DNS server đã xong.

### 1.2. (Optional) Enable KVM-Nested trên KVM

Đối với môi trường LAB, máy Director là máy ảo và ta cần bật chế độ Nested Virtualization trên KVM như sau : 

- Kiểm tra xem chế độ Nested Virtualization đã được bật chưa 
```sh
cat /sys/module/kvm_intel/parameters/nested
N
```

Nếu output là `N` nghĩa là Nested KVM đang bị tắt. Ta cần bật lên như sau : 
```sh
cat << EOF >> /etc/modprobe.d/kvm-nested.conf
options kvm-intel nested=1
options kvm-intel enable_shadow_vmcs=1
options kvm-intel enable_apicv=1
options kvm-intel ept=1

modprobe -r kvm_intel
modprobe -a kvm_intel
```

- Kiểm tra lại chế độ Nested Virtualization đã được bật chưa 
```sh
cat /sys/module/kvm_intel/parameters/nested
Y
```

Nếu output là `Y` nghĩa là Nested KVM đang được bật thành công.

## 2. Cài đặt node UnderCloud (Director) (Thực hiện trên node Undercloud)

### 2.1. Bước 1 : Setup stack user và cài đặt 1 số gói cần thiết

Chú ý : Để Selinux tại các node ở chế độ `Enforcing` thay vì `Permissve` hoặc `Disabled`.

Do TripleO cần can thiệp tới file `/etc/hostname` do vậy cần thực hiện restore nội dunng file :
```sh
restorecon -R -v /etc/hostname
```
Tạo một user non-root là `stack` và setup quyền sudo cho user đó.
TripleO sẽ sử dụng user `stack` để thực hiện việc cài đặt và cấu hình.

```sh
useradd stack
echo "welcome123" | passwd --stdin stack
echo "stack ALL=(root) NOPASSWD:ALL" | sudo tee -a /etc/sudoers.d/stack
chmod 0440 /etc/sudoers.d/stack
su - stack
```

Enable và cấu hình repo cho undercloud :

```sh

sudo yum -y install yum-plugin-priorities
sudo yum install epel-release -y
wget https://trunk.rdoproject.org/centos7/current/python2-tripleo-repos-0.0.1-0.20200409224957.8bac392.el7.noarch.rpm
sudo yum install python-requests -y
sudo rpm -ivh python2-tripleo-repos-0.0.1-0.20200409224957.8bac392.el7.noarch.rpm
sudo -E tripleo-repos -b train current
sudo yum install python-tripleoclient -y
sudo yum install wget byobu -y
```

Copy file cấu hình mặc định của tripleO và chỉnh sửa :

```sh
cp /usr/share/python-tripleoclient/undercloud.conf.sample ~/undercloud.conf
```

Chỉnh sửa nội dung file `undercloud.conf` theo IP-PLANNING như sau : 
```sh
[DEFAULT]
local_ip = 10.10.34.41/24
undercloud_public_host = 10.10.34.55
undercloud_admin_host = 10.10.34.56
undercloud_hostname = undercloud-director.example
[ctlplane-subnet]
gateway = 10.10.34.41
local_interface = eth3
cidr = 10.10.34.0/24
masquerade_network = 10.10.34.0/24
dhcp_start = 10.10.34.42
dhcp_end = 10.10.34.45
inspection_iprange = 10.10.34.46,10.10.34.50
```

Thực hiện hạ phiên bản của leatherman để tương thích với phiên bản puppet : 

```sh
sudo yum downgrade leatherman -y
```

Thực hiện cài đặt undercloud :
```sh
byobu
openstack undercloud install
```
![tripleo](/OpenStack/images/tripleo-11.png)

### 2.2. Bước 2 : Upload các image OpenStack

Tải các image và upload lên OpenStack

```sh
sudo wget https://images.rdoproject.org/train/rdo_trunk/current-tripleo-rdo/overcloud-full.tar --no-check-certificate
sudo wget https://images.rdoproject.org/train/rdo_trunk/current-tripleo-rdo/ironic-python-agent.tar --no-check-certificate
mkdir ~/images
tar -xpvf ironic-python-agent.tar -C ~/images/
tar -xpvf overcloud-full.tar -C ~/images/
source ~/stackrc
openstack overcloud image upload --image-path ~/images/
```

![tripleo](/OpenStack/images/tripleo-12.png)

Kiểm tra các image đã được upload :
```sh
openstack image list
```
![tripleo](/OpenStack/images/tripleo-13.png)


### 2.3. Bước 3 : Thêm DNS server vào subnet

Kiểm tra subnet hiện có của TripleO

```sh
openstack subnet list
```
![tripleo](/OpenStack/images/tripleo-14.png)

Thêm DNS server cho subnet :

```sh
neutron subnet-update 2ee5a9b5-03a6-4de8-826a-490b885c71f3 --dns-nameserver 10.10.34.52
```

Kiểm tra lại subnet được thêm DNS server :
```sh
openstack subnet show 2ee5a9b5-03a6-4de8-826a-490b885c71f3
```

![tripleo](/OpenStack/images/tripleo-15.png)

### 2.4. Bước 4 : Tạo vm cho Controller và Compute (Thực hiện trên KVM)

Trên node KVM, thực hiện tạo các máy ảo Controller và Compute1, Compute2.

Tạo các file disk của máy ảo

```sh
qemu-img create -f qcow2 -o preallocation=metadata /kvm/overcloud-controller.qcow2 60G
qemu-img create -f qcow2 -o preallocation=metadata /kvm/overcloud-compute1.qcow2 60G
qemu-img create -f qcow2 -o preallocation=metadata /kvm/overcloud-compute2.qcow2 60G
qemu-img create -f qcow2 -o preallocation=metadata /kvm/test01-overcloud-ceph-root.qcow2 60G
qemu-img create -f qcow2 -o preallocation=metadata /kvm/test01-overcloud-ceph-osd1.qcow2 100G
qemu-img create -f qcow2 -o preallocation=metadata /kvm/test01-overcloud-ceph-osd2.qcow2 100G
```

Tạo các file XML chứa thông tin máy ảo

```sh
virt-install --ram 8192 --vcpus 8 --os-variant rhel7 --disk path=/kvm/overcloud-controller.qcow2,device=disk,bus=virtio,format=qcow2 --noautoconsole --vnc --network network:vlan634 --name overcloud-controller --cpu host-model,+vmx --dry-run --print-xml > /tmp/overcloud-controller.xml

virt-install --ram 8192 --vcpus 8 --os-variant rhel7 --disk path=/kvm/overcloud-compute1.qcow2,device=disk,bus=virtio,format=qcow2 --noautoconsole --vnc --network network:vlan634 --name overcloud-compute1 --cpu host-model,+vmx --dry-run --print-xml > /tmp/overcloud-compute1.xml

virt-install --ram 8192 --vcpus 8 --os-variant rhel7 --disk path=/kvm/overcloud-compute2.qcow2,device=disk,bus=virtio,format=qcow2 --noautoconsole --vnc --network network:vlan634 --name overcloud-compute2 --cpu host-model,+vmx --dry-run --print-xml > /tmp/overcloud-compute2.xml
```
Define file XML của máy ảo

```sh
virsh define --file /tmp/overcloud-controller.xml
virsh define --file /tmp/overcloud-compute1.xml
virsh define --file /tmp/overcloud-compute2.xml
```

Kiểm tra tình trạng của máy ảo được tạo :

```sh
virsh list --all | grep overcloud*
```

![tripleo](/OpenStack/images/tripleo-16.png)


### 2.5. Bước 5 : Cài đặt và cấu hình VBMC (Virtual BMC) trên node UnderCloud

Cài đặt gói `vbmc` và `ipmitool` :
```sh
sudo yum install python-virtualbmc ipmitool -y
```

Copy SSH Key từ máy ảo Undercloud tới máy KVM :
```sh
ssh-copy-id root@$KVM_IP
```

Thêm các máy ảo Controller và Compute vào `VBMC` với các câu lệnh : 
```sh
vbmc add overcloud-compute1 --port 6001 --username admin --password password --libvirt-uri qemu+ssh://root@KVM_IP/system
vbmc add overcloud-compute2 --port 6002 --username admin --password password --libvirt-uri qemu+ssh://root@KVM_IP/system
vbmc add overcloud-controller  --port 6003 --username admin --password password --libvirt-uri qemu+ssh://root@KVM_IP/system

vbmc start overcloud-controller 
vbmc start overcloud-compute1 
vbmc start overcloud-compute2
```

Kiểm tra list của VM :
```sh
vbmc list
```
![tripleo](/OpenStack/images/tripleo-17.png)

Kiểm tra trạng thái của máy ảo trên node Undercloud với IMPI 
```sh
ipmitool -I lanplus -U admin -P password -H 127.0.0.1 -p 6001 power status
ipmitool -I lanplus -U admin -P password -H 127.0.0.1 -p 6002 power status
ipmitool -I lanplus -U admin -P password -H 127.0.0.1 -p 6003 power status
```

![tripleo](/OpenStack/images/tripleo-18.png)

### 2.6. Bước 6 : Tạo và Import Inventory cho Overcloud node

Tại KVM, thực hiện lấy MAC của các máy ảo Controller và Compute

```sh
virsh domiflist overcloud-compute1 | grep vlan634
virsh domiflist overcloud-compute2 | grep vlan634
virsh domiflist overcloud-controller | grep vlan634
```

![tripleo](/OpenStack/images/tripleo-19.png)

Trên Undercloud node, tạo một file JSON với tên `overcloud-stackenv.json`

```sh
vi overcloud-stackenv.json
{
  "nodes": [
    {
      "arch": "x86_64",
      "disk": "60",
      "memory": "8192",
      "name": "overcloud-compute1",
      "pm_user": "admin",
      "pm_addr": "127.0.0.1",
      "pm_password": "password",
      "pm_port": "6001",
      "pm_type": "pxe_ipmitool",
      "mac": [
        "52:54:00:42:7f:df"
      ],
      "cpu": "8"
    },
    {
      "arch": "x86_64",
      "disk": "60",
      "memory": "8192",
      "name": "overcloud-compute2",
      "pm_user": "admin",
      "pm_addr": "127.0.0.1",
      "pm_password": "password",
      "pm_port": "6002",
      "pm_type": "pxe_ipmitool",
      "mac": [
        "52:54:00:f6:22:d7"
      ],
      "cpu": "8"
    },
    {
      "arch": "x86_64",
      "disk": "60",
      "memory": "8192",
      "name": "overcloud-controller",
      "pm_user": "admin",
      "pm_addr": "127.0.0.1",
      "pm_password": "password",
      "pm_port": "6003",
      "pm_type": "pxe_ipmitool",
      "mac": [
        "52:54:00:be:79:89"
      ],
      "cpu": "8"
    }
  ]
}
```

Thay thế MAC phù hợp với các node Controller và Compute ở trên.

Import node và thực hiện quá trình `introspection` với các node như sau :
```sh
source stackrc
openstack overcloud node import --introspect --provide overcloud-stackenv.json
```

Thực hiện set Role cho các node. Các máy ảo `overcloud-compute1` và `overcloud-compute2` sẽ có role `Compute`. Máy ảo `overcloud-controler` sẽ có role `Control`.

```sh
openstack baremetal node set --property capabilities='profile:compute,boot_option:local' $compute1_id
openstack baremetal node set --property capabilities='profile:compute,boot_option:local' $compute2_id
openstack baremetal node set --property capabilities='profile:control,boot_option:local' $controller_id
```

Kiểm tra với câu lệnh :
```sh
openstack overcloud profiles list
```

![tripleo](/OpenStack/images/tripleo-20.png)

Trạng thái của `Provision State` cần là `available` như trên.

