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