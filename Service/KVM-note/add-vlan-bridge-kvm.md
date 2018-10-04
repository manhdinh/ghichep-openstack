## ADD vlan Bridge cho KVM

Giả sử ta có VLAN 191, 192 và muốn các vm của KVM sử dụng VLAN này thì cần setup tại 2 đầu . Port eno1 tại KVM kết nối tới SW Provider port 2/0/11, nhận được các vlan 191 và 192. 

### Switch

Port (hoặc PO) gắn với KVM cần để chế độ Trunking. VD 

```sh
interface GigabitEthernet2/0/11
 description ->KVM-1
 switchport trunk encapsulation dot1q
 switchport mode trunk
```

### Trên KVM

 - Kiểm tra port kết nối tới SW cấp vlan : 
```sh
cdpr -d eno1
```

Kết quả trả về nên là port 2/0/11 như sau :

![kvm](/Service/KVM-note/images/kvm-00.png)

Để cài cdpr check : 
```sh
yum install epel-release cdpr -y
```

Kiểm tra kêt nối đã đúng với mô hình. Setup card VLAN tagging cho port eno1.

 - Setup eno1
```sh
cp /etc/sysconfig/network-scripts/ifcfg-eno1 /etc/sysconfig/network-scripts/ifcfg-eno1.orig
rm -rf /etc/sysconfig/network-scripts/ifcfg-eno1

cat << EOF > /etc/sysconfig/network-scripts/ifcfg-eno1
TYPE=Ethernet
NAME=eno1
DEVICE=eno1
ONBOOT=yes
EOF
```

 - Setup VLAN tagging interface eno1.191 : 
```sh
cat << EOF > /etc/sysconfig/network-scripts/ifcfg-eno1.191
DEVICE=eno1.191
VLAN=yes
ONBOOT=yes
BOOTPROTO=none
EOF 
```
 - Setup bridge
```sh
brctl addbr vlan191
brctl addif vlan191 eno1.191
```

 - Setup interface bridge vlan_191
```sh
cat << EOF > /etc/sysconfig/network-scripts/ifcfg-vlan191
DEVICE=vlan191
TYPE=Bridge
BOOTPROTO=none
ONBOOT=yes
DELAY=0
EOF
```

 - Chỉnh sửa file config eno1.191
```sh
echo "BRIDGE=vlan191" >> /etc/sysconfig/network-scripts/ifcfg-eno1.191
```

 - Restart network
```sh
systemctl restart network
```