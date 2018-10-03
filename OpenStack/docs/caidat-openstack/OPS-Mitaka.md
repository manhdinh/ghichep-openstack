# Hướng dẫn cài đặt OpenStack Mitaka sử dụng Openvswitch : mix network Provider và Self-service (Không Bonding)

# Mục lục
 *	[1. Chuẩn bị](#1)
 *	[2. Cài đặt trên node Controller](#2)
	*	[2.1. Setup môi trường cài đặt](#2.1)
	*	[2.2. Cài đặt các thành phần phụ trợ](#2.2)	
		*	[2.2.1. Cài đặt NTP](#2.2.1)
		*	[2.2.2. Cài đặt và cấu hình database MySQL](#2.2.2)
		*	[2.2.3 Cài đặt và cấu hình RabbitMQ](#2.2.3)
		*	[2.2.4 Cài đặt và cấu hình Memcache](#2.2.4)
	*	[2.3. Cài đặt các thành phân lõi](#2.3)
		*	[2.3.1 Cài đặt và cấu hình Keystone](#2.3.1)
		*	[2.3.2 Cài đặt và cấu hình Glance](#2.3.2)
		*	[2.3.3 Cài đặt Nova](#2.3.3)
		*	[2.3.4 Cài đặt Neutron (OpenvSwitch)](#2.3.4)
		*	[2.3.5 Cài đặt và cấu hình Horizon](#2.3.5)
*	[3 Cài đặt trên Compute](#3)
	*	[3.1 Setup môi trường cài đặt](#3.1)
	*	[3.2 Cài đặt các thành phần phụ trợ](#3.2)
		*	[3.2.1 Cài đặt NTP](#3.2.1)
	*	[3.3 Cài đặt các thành phần lõi](#3.3)
		*	[3.3.1 Cài đặt và cấu hình Nova](#3.3.1)
		*	[3.3.2 Cài đặt và cấu hình Neutron openvSwitch](#3.3.2)
*	[4. Cài đặt mô hình network Self-service](#4)
	*	[4.1. Thực hiện trên node Controller](#4.1)
	*	[4.2 Thực hiện trên node Compute](#4.2)
	*	[4.3 Kiểm tra](#4.3)
*	[5. Tạo máy ảo theo dạng Self-service](#5)


# 1. Chuẩn bị <a name="1"> </a> 

 -	Distro : RHEL 7 đã register

 -	Mô hình
 
![ops](/ManhDV/OpenStack/images/ops-ovs-system.png)

 -	IP Planing

![ops](/ManhDV/OpenStack/images/ipplan-01.png)

# 2. Cài đặt trên CTL <a name="2"> </a> 
## 2.1 Setup môi trường cài đặt <a name="2.1"> </a> 

 -	Cấu hình file host
 
`vi /etc/hosts`
 
```sh
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
172.16.69.10    controller
172.16.69.20    compute
```

`vi /etc/resolv.conf`

```sh
nameserver 8.8.8.8
```

 - Kiểm tra ping ra Internet
 
`ping google.com`

 - Config cho các module network
 
```sh
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
echo "net.ipv4.conf.all.rp_filter=0" >> /etc/sysctl.conf
echo "net.ipv4.conf.default.rp_filter=0" >> /etc/sysctl.conf
sysctl -p
```

 - Đăng ký tài khoản RHEL (Người dùng đã phải đăng ký bằng mail trên website của RHEL)
 
```sh
subscription-manager register --username="user" --password="userpassword" --auto-attach

subscription-manager repos --enable=rhel-7-server-optional-rpms --enable=rhel-7-server-extras-rpms --enable=rhel-7-server-rh-common-rpms
```

 - Tắt firewall và selinux
```sh
systemctl disable firewalld
systemctl stop firewalld
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
```

 - Khởi động lại máy
 
`init 6`


 -  Add repo và update hệ thống

`yum install -y https://repos.fedorapeople.org/repos/openstack/openstack-mitaka/rdo-release-mitaka-6.noarch.rpm`

`yum upgrade -y`

 - Cài đặt byobu và wget 
 
`yum install -y http://dl.fedoraproject.org/pub/epel/7/x86_64/b/byobu-5.73-4.el7.noarch.rpm wget`

 - Chạy lệnh byobu
 
`byobu`

 - Cài đặt openstack-client để sử dụng các câu lệnh openstack
 
`yum install python-openstackclient -y`

 - Cài đặt gói openstack-selinux để quản lý policy cho các service openstack.
 
`yum install openstack-selinux -y`

## 2.2 Cài đặt các thành phần phụ trợ <a name="2.2"> </a> 

### 2.2.1 Cài đặt NTP Cài đặt NTP <a name="2.2.1"> </a> 
 
`yum install chrony -y `

 - Chỉnh sửa file cấu hình /etc/chrony/chrony.conf
 
```sh
sed -i "s/server 0.rhel.pool.ntp.org iburst/server vn.pool.ntp.org iburst/g" /etc/chrony.conf

sed -i 's/server 1.rhel.pool.ntp.org iburst/#server 1.rhel.pool.ntp.org iburst/g' /etc/chrony.conf

sed -i 's/server 2.rhel.pool.ntp.org iburst/#server 2.rhel.pool.ntp.org iburst/g' /etc/chrony.conf

sed -i 's/server 3.rhel.pool.ntp.org iburst/#server 3.rhel.pool.ntp.org iburst/g' /etc/chrony.conf

sed -i 's/#allow 192.168\/16/allow 172.16.69.0\/24/g' /etc/chrony.conf

```
 - Restart dịch vụ ntp
 
```sh
systemctl enable chronyd.service
systemctl start chronyd.service
```
 - Chạy lệnh kiểm tra trên 2 node CTL và COM
 
`chronyc sources`

![ops](/ManhDV/OpenStack/images/ntp.png)

### 2.2.2 Cài đặt và cấu hình database MySQL <a name="2.2.2"> </a> 

 - Cài đặt database MySQL

`yum install -y mariadb mariadb-server python2-PyMySQL `

 - Tạo file cấu hình cho dịch vụ Openstack 
 
`vi /etc/my.cnf.d/openstack.cnf`

```sh
[mysqld]

bind-address = 172.16.69.10
default-storage-engine = innodb
innodb_file_per_table
max_connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8
```

 - Start dịch vụ và cho phép khởi động dịch vụ cùng hệ thống
 
```sh
systemctl enable mariadb.service
systemctl start mariadb.service
```

 - Thực hiện security cho mysql, thực hiện theo các bước sau : 
 
`mysql_secure_installation`

```sh
Enter current password for root (enter for none): [enter]
Change the root password? [Y/n]: y
Set root password? [Y/n] y
New password:Welcome123
Re-enter new password:Welcome123
Remove anonymous users? [Y/n]: y
Disallow root login remotely? [Y/n]: y
Remove test database and access to it? [Y/n]: y
Reload privilege tables now? [Y/n]: y
```

### 2.2.3 Cài đặt và cấu hình RabbitMQ <a name="2.2.3"> </a> 

 - Cài đặt rabbitmq
 
`yum install rabbitmq-server -y `

 - Start dịch vụ và cho phép khởi động dịch vụ cùng hệ thống
 
```sh
systemctl enable rabbitmq-server.service
systemctl start rabbitmq-server.service
```

 - Thêm user **openstack**
 
`rabbitmqctl add_user openstack Welcome123`

 - Phân quyền cho user **openstack** được phép config, write, read trên rabbitmq

`rabbitmqctl set_permissions openstack ".*" ".*" ".*"`

### 2.2.4 Cài đặt và cấu hình Memcache <a name="2.2.4"> </a> 

 - Cài đặt memcache
 
`yum install -y memcached python-memcached`

 - Sao lưu cấu hình memcache
 
`cp /etc/sysconfig/memcached /etc/sysconfig/memcached.bka`

 - Chính sửa cấu hình memcache
 
`sed -i 's/OPTIONS=\"-l 127.0.0.1,::1\"/OPTIONS=\"-l 0.0.0.0,::1\"/g' /etc/sysconfig/memcached`

 - Start dịch vụ và cho phép khởi động dịch vụ cùng hệ thống
 
```sh
systemctl enable memcached.service
systemctl start memcached.service
```

## 2.3. Cài đặt các thành phân lõi <a name="2.3"> </a> 

### 2.3.1 Cài đặt và cấu hình Keystone <a name="2.3.1"> </a> 

 - Tạo database cho keystone
 
```sh
mysql -u root -pWelcome123
CREATE DATABASE keystone;
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'Welcome123';
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'Welcome123';
FLUSH PRIVILEGES;
exit
```

 - Tạo token từ câu lệnh `openssl rand -hex 10`

```sh
e27814f52b002f1e813d
```

 - Cài đặt keystone
 
`yum install -y openstack-keystone httpd mod_wsgi`

 - Sao lưu file cấu hình keystone
 
`cp /etc/keystone/keystone.conf /etc/keystone/keystone.conf.bka`

 - Chỉnh sửa file cấu hình keystone
 
`vi /etc/keystone/keystone.conf`

```sh
[DEFAULT]

admin_token = e27814f52b002f1e813d

[database]

connection = mysql+pymysql://keystone:Welcome123@172.16.69.10/keystone

[token]

provider = fernet
```

 - Tạo fernet key
 
`keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone`

 - Phân quyền thư mục log và đồng bộ databse keystone
 
```sh
chown keystone:keystone /var/log/keystone/keystone.log
su -s /bin/sh -c "keystone-manage db_sync" keystone
```

 - Chỉnh sửa file /etc/httpd/conf/httpd.conf
 
`echo 'ServerName 172.16.69.10' >> /etc/httpd/conf/httpd.conf`

 - Tạo file /etc/httpd/conf.d/wsgi-keystone.conf 
 
`vi /etc/httpd/conf.d/wsgi-keystone.conf`

```sh
Listen 5000
Listen 35357

<VirtualHost *:5000>
    WSGIDaemonProcess keystone-public processes=5 threads=1 user=keystone group=keystone display-name=%{GROUP}
    WSGIProcessGroup keystone-public
    WSGIScriptAlias / /usr/bin/keystone-wsgi-public
    WSGIApplicationGroup %{GLOBAL}
    WSGIPassAuthorization On
    ErrorLogFormat "%{cu}t %M"
    ErrorLog /var/log/httpd/keystone-error.log
    CustomLog /var/log/httpd/keystone-access.log combined

    <Directory /usr/bin>
        Require all granted
    </Directory>
</VirtualHost>

<VirtualHost *:35357>
    WSGIDaemonProcess keystone-admin processes=5 threads=1 user=keystone group=keystone display-name=%{GROUP}
    WSGIProcessGroup keystone-admin
    WSGIScriptAlias / /usr/bin/keystone-wsgi-admin
    WSGIApplicationGroup %{GLOBAL}
    WSGIPassAuthorization On
    ErrorLogFormat "%{cu}t %M"
    ErrorLog /var/log/httpd/keystone-error.log
    CustomLog /var/log/httpd/keystone-access.log combined

    <Directory /usr/bin>
        Require all granted
    </Directory>
</VirtualHost>
```

 - Restart dịch vụ và cho phép khởi động dịch vụ cùng hệ thống
 
```sh
systemctl enable httpd.service
systemctl restart httpd.service
```

 - Export các thông tin keystone
 
```sh
export OS_TOKEN=e27814f52b002f1e813d
export OS_URL=http://172.16.69.10:35357/v3
export OS_IDENTITY_API_VERSION=3
```

 - Tạo service entity cho keystone
 
`openstack service create --name keystone --description "OpenStack Identity" identity`

 - Tạo API endpoints cho keystone
 
```sh
openstack endpoint create --region RegionOne identity public http://172.16.69.10:5000/v3
  
openstack endpoint create --region RegionOne identity internal http://172.16.69.10:5000/v3
  
openstack endpoint create --region RegionOne identity admin http://172.16.69.10:35357/v3
```

 - Tạo domain default
 
`openstack domain create --description "Default Domain" default`

 - Tạo admin project
 
`openstack project create --domain default --description "Admin Project" admin`

 - Tạo user admin, nhập password là `Welcome123`
 
`openstack user create admin --domain default --password Welcome123`

 - Tạo role admin
 
`openstack role create admin`

 - Gán role admin và user và project admin
 
`openstack role add --project admin --user admin admin`

 - Tạo service project
 
```sh
openstack project create --domain default --description "Service Project" service
```

 - Tạo demo project
 
```sh
openstack project create --domain default --description "Demo Project" demo
```

 - Tạo user demo
 
```sh
openstack user create demo --domain default --password Welcome123
```

 - Tạo role demo
 
`openstack role create user`

 - Gán role demo vào project và user demo
 
`openstack role add --project demo --user demo user`

 - Sao lưu file /etc/keystone/keystone-paste.ini
 
`cp /etc/keystone/keystone-paste.ini /etc/keystone/keystone-paste.ini.bka`

 - Xóa các `admin_token_auth` khỏi các section sau :
 
```sh
[pipeline:public_api]

[pipeline:admin_api]

[pipeline:api_v3]
```

 - Unset biến 
 
`unset OS_TOKEN OS_URL`

 - Tạo user admin và demo, password là `Welcome123`
 
```sh
openstack --os-auth-url http://172.16.69.10:35357/v3 \
  --os-project-domain-name default --os-user-domain-name default \
  --os-project-name admin --os-username admin token issue
  
openstack --os-auth-url http://172.16.69.10:5000/v3 \
  --os-project-domain-name default --os-user-domain-name default \
  --os-project-name demo --os-username demo token issue  
```

Tạo các file source script admin-rc và demo-rc

`vi admin-rc`

```sh
export OS_PROJECT_DOMAIN_NAME=default
export OS_USER_DOMAIN_NAME=default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=Welcome123
export OS_AUTH_URL=http://172.16.69.10:35357/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
```

`vi demo-rc`

```sh
export OS_PROJECT_DOMAIN_NAME=default
export OS_USER_DOMAIN_NAME=default
export OS_PROJECT_NAME=demo
export OS_USERNAME=demo
export OS_PASSWORD=Welcome123
export OS_AUTH_URL=http://172.16.69.10:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
```

 - Kiểm tra script
 
`. admin-rc`

`openstack token issue`

### 2.3.2 Cài đặt và cấu hình Glance <a name="2.3.2"> </a> 

 - Tạo database cho Glance
 
```sh
mysql -u root -pWelcome123
CREATE DATABASE glance;
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY 'Welcome123';
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'Welcome123';
FLUSH PRIVILEGES;
exit
```

 - Tạo user glance

```sh
. admin-rc
openstack user create glance --domain default --password Welcome123
```

 - Gán quyền cho user glance và project service
 
`openstack role add --project service --user glance admin`

 - Tạo glance service entity
 
`openstack service create --name glance --description "OpenStack Image" image`

 - Tạo service API endpoints 
 
```sh
openstack endpoint create --region RegionOne image public http://172.16.69.10:9292

openstack endpoint create --region RegionOne image internal http://172.16.69.10:9292

openstack endpoint create --region RegionOne image admin http://172.16.69.10:9292
```

 - Cài đặt glance
 
`yum install openstack-glance -y `

 - Sao lưu file cấu hình glance-api.conf 
 
`cp /etc/glance/glance-api.conf /etc/glance/glance-api.conf.bka`

 - Sửa file cấu hình /etc/glance/glance-api.conf

```sh
[database]
connection = mysql+pymysql://glance:Welcome123@172.16.69.10/glance

[keystone_authtoken]
auth_uri = http://172.16.69.10:5000
auth_url = http://172.16.69.10:35357
memcached_servers = 172.16.69.10:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = glance
password = Welcome123

[paste_deploy]
flavor = keystone

[glance_store]
stores = file,http
default_store = file
filesystem_store_datadir = /var/lib/glance/images/
```

 - Sao lưu file cấu hình /etc/glance/glance-registry.conf
 
`cp /etc/glance/glance-registry.conf /etc/glance/glance-registry.conf.bka`

 - Sửa file cấu hình /etc/glance/glance-registry.conf
 
```sh
[database]
connection = mysql+pymysql://glance:Welcome123@172.16.69.10/glance

[keystone_authtoken]
auth_uri = http://172.16.69.10:5000
auth_url = http://172.16.69.10:35357
memcached_servers = 172.16.69.10:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = glance
password = Welcome123

[paste_deploy]
flavor = keystone
```

 - Đồng bộ glance database
 
`su -s /bin/sh -c "glance-manage db_sync" glance`

 - Start dịch vụ glance-api và glance-registry, cho phép khởi động dịch vụ cùng hệ thống
 
```sh
systemctl enable openstack-glance-api.service 
systemctl enable openstack-glance-registry.service
systemctl start openstack-glance-api.service 
systemctl start openstack-glance-registry.service
```

 - Tải image cirros
 
`wget http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-x86_64-disk.img`

 - Upload image cirros
 
`. admin-rc`
 
```sh
openstack image create "cirros" \
  --file cirros-0.3.4-x86_64-disk.img \
  --disk-format qcow2 --container-format bare \
  --public
```

 - Kiểm tra image đã upload
 
`openstack image list `

### 2.3.3 Cài đặt Nova <a name="2.3.3"> </a> 

 - Tạo database cho Nova
 
```sh
mysql -u root -pWelcome123

CREATE DATABASE nova_api;
CREATE DATABASE nova;

GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' IDENTIFIED BY 'Welcome123';
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' IDENTIFIED BY 'Welcome123';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' IDENTIFIED BY 'Welcome123';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'Welcome123';
FLUSH PRIVILEGES;
exit 
```

 - Tạo user nova
 
`. admin-rc`

`openstack user  create nova --domain default --password Welcome123`

 - Gán quyền admin cho user nova
 
`openstack role add --project service --user nova admin`

 - Tạo nova service entity
 
`openstack service create --name nova --description "OpenStack Compute" compute`

 - Tạo API endpoints cho Nova
 
```sh
openstack endpoint create --region RegionOne compute public http://172.16.69.10:8774/v2.1/%\(tenant_id\)s
  
openstack endpoint create --region RegionOne compute internal http://172.16.69.10:8774/v2.1/%\(tenant_id\)s

openstack endpoint create --region RegionOne compute admin http://172.16.69.10:8774/v2.1/%\(tenant_id\)s
```

 - Cài đặt nova
 
```sh
yum install -y openstack-nova-api openstack-nova-conductor \
  openstack-nova-console openstack-nova-novncproxy \
  openstack-nova-scheduler
```

 - Sao lưu file cấu hình /etc/nova/nova.conf 
 
`cp /etc/nova/nova.conf  /etc/nova/nova.conf.bka`

 - Sửa file cấu hình /etc/nova/nova.conf 
 
```sh
[DEFAULT]
my_ip = 172.16.69.10
enabled_apis = osapi_compute,metadata
rpc_backend = rabbit
auth_strategy = keystone
use_neutron = True
firewall_driver = nova.virt.firewall.NoopFirewallDriver

[api_database]
connection = mysql+pymysql://nova:Welcome123@172.16.69.10/nova_api

[database]
connection = mysql+pymysql://nova:Welcome123@172.16.69.10/nova

[oslo_messaging_rabbit]
rabbit_host = 172.16.69.10
rabbit_userid = openstack
rabbit_password = Welcome123

[keystone_authtoken]
auth_uri = http://172.16.69.10:5000
auth_url = http://172.16.69.10:35357
memcached_servers = 172.16.69.10:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = nova
password = Welcome123

[vnc]
vncserver_listen = $my_ip
vncserver_proxyclient_address = $my_ip

[glance]
api_servers = http://172.16.69.10:9292

[oslo_concurrency]
lock_path = /var/lib/nova/tmp
```

 - Đồng bộ database nova
 
```sh
su -s /bin/sh -c "nova-manage api_db sync" nova
su -s /bin/sh -c "nova-manage db sync" nova
```

 - Start các service nova và cho phép khởi động dịch vụ cùng hệ thống
 
```sh
systemctl enable openstack-nova-api.service \
  openstack-nova-consoleauth.service openstack-nova-scheduler.service \
  openstack-nova-conductor.service openstack-nova-novncproxy.service

systemctl start openstack-nova-api.service \
  openstack-nova-consoleauth.service openstack-nova-scheduler.service \
  openstack-nova-conductor.service openstack-nova-novncproxy.service  
```

### 2.3.4 Cài đặt Neutron (Theo dạng network OpenvSwitch) <a name="2.3.4"> </a> 

 - Tạo database cho neutron
 
```sh
mysql -u root -pWelcome123
CREATE DATABASE neutron;
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' IDENTIFIED BY 'Welcome123';
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'Welcome123';
FLUSH PRIVILEGES;
exit
```

 - Tạo user neutron
 
`vi admin-rc`
 
`openstack user create neutron --domain default --password Welcome123`

 - Gán quyền admin cho user neutron
 
`openstack role add --project service --user neutron admin`

 - Tạo neutron service entity
 
`openstack service create --name neutron --description "OpenStack Networking" network`

 - Tạo API endpoints cho neutron
 
```sh
openstack endpoint create --region RegionOne network public http://172.16.69.10:9696

openstack endpoint create --region RegionOne network internal http://172.16.69.10:9696

openstack endpoint create --region RegionOne network admin http://172.16.69.10:9696
```
 
 - Cài đặt Neutron sử dụng openvSwitch
 
`yum -y install openstack-neutron openstack-neutron-ml2  openstack-neutron-openvswitch ebtables`

 - Sao lưu file cấu hình neutron
 
`cp /etc/neutron/neutron.conf /etc/neutron/neutron.conf.bka`

 - Sửa file cấu hình /etc/neutron/neutron.conf 
 
```sh
[DEFAULT]
core_plugin = ml2
service_plugins =
rpc_backend = rabbit
auth_strategy = keystone
notify_nova_on_port_status_changes = True
notify_nova_on_port_data_changes = True
dhcp_agents_per_network = 2

[database]
connection = mysql+pymysql://neutron:Welcome123@172.16.69.10/neutron

[oslo_messaging_rabbit]
rabbit_host = 172.16.69.10
rabbit_userid = openstack
rabbit_password = Welcome123

[keystone_authtoken]
auth_uri = http://172.16.69.10:5000
auth_url = http://172.16.69.10:35357
memcached_servers = 172.16.69.10:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = Welcome123

[nova]
auth_url = http://172.16.69.10:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = nova
password = Welcome123

[oslo_concurrency]
lock_path = /var/lib/neutron/tmp
```

 - Sao lưu file cấu hình /etc/neutron/plugins/ml2/ml2_conf.ini
 
`cp /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugins/ml2/ml2_conf.ini.bka`

 - Sửa file cấu hình /etc/neutron/plugins/ml2/ml2_conf.ini
 
```sh
[ml2]
type_drivers = flat,vlan
tenant_network_types =
mechanism_drivers = openvswitch
extension_drivers = port_security

[ml2_type_flat]
flat_networks = provider

[ml2_type_vlan]
network_vlan_ranges = provider

[securitygroup]
firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
enable_security_group = True
```

 - Sao lưu file cấu hình /etc/neutron/dhcp_agent.ini
 
`cp /etc/neutron/dhcp_agent.ini /etc/neutron/dhcp_agent.ini.bka`

 - Sửa file cấu hình /etc/neutron/dhcp_agent.ini
 
```sh
[DEFAUL]
verbose = True
interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
enable_isolated_metadata = True
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
```

 - Sao lưu file cấu hình /etc/neutron/metadata_agent.ini
 
`cp /etc/neutron/metadata_agent.ini /etc/neutron/metadata_agent.ini.bka`

 - Sửa file cấu hình /etc/neutron/metadata_agent.ini
 
```sh
[DEFAULT]
nova_metadata_ip = 172.16.69.10
metadata_proxy_shared_secret = Welcome123
```

 - Sao lưu file cấu hình /etc/neutron/plugins/ml2/openvswitch_agent.ini 
 
`cp /etc/neutron/plugins/ml2/openvswitch_agent.ini /etc/neutron/plugins/ml2/openvswitch_agent.ini.bka`

 - Sửa file cấu hình /etc/neutron/plugins/ml2/openvswitch_agent.ini
 
```sh
[ovs]
bridge_mappings = provider:br-provider

[securitygroup]
firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
enable_security_group = True
```

 - Khai báo cấu hình neutron trong file cấu hình /etc/nova/nova.conf
 
```sh
[neutron]
url = http://172.16.69.10:9696
auth_url = http://172.16.69.10:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = Welcome123

service_metadata_proxy = True
metadata_proxy_shared_secret = Welcome123
```
 
 - Tạo OVS provider
 
`ovs-vsctl add-br br-provider`

 - Gán interface provider vào OVS provider
 
`ovs-vsctl add-port br-provider eno33554960`

 - Sao lưu file cấu hình ifcfg-eno33554960
 
`cp /etc/sysconfig/network-scripts/ifcfg-eno33554960 /etc/sysconfig/network-scripts/ifcfg-eno33554960.bka`

 - Tạo file cấu hình /etc/sysconfig/network-scripts/ifcfg-eno33554960 mới
 
```sh
DEVICE=eno33554960
NAME=eno33554960
DEVICETYPE=ovs
TYPE=OVSPort
OVS_BRIDGE=br-provider
ONBOOT=yes
BOOTPROTO=none
NM_CONTROLLED=no
```

 - Tạo file cấu hình /etc/sysconfig/network-scripts/ifcfg-br-provider mới

```sh
ONBOOT=yes
IPADDR=172.16.69.10
NETMASK=255.255.255.0
GATEWAY=172.16.69.1
DNS=8.8.8.8
DEVICE=br-provider
NAME=br-provider
DEVICETYPE=ovs
OVSBOOTPROTO=none
TYPE=OVSBridge
```
 - Restart network
 
`systemctl restart network`

 - Kiểm tra nếu IP trên card eno33554960 chưa mất, xóa IP bằng tay và restart network:
 
```sh
ip addr del 172.16.69.10/24 dev eno33554960
systemctl restart network
```

 - Tạo symbolic link từ ml2_conf.ini tới neutron/plugin.ini 
 
`ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini`


 - Đồng bộ database neutron
 
```sh
su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf \
  --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
```

 - Restart dịch vụ nova
 
`systemctl restart openstack-nova-api.service`

 - Start các service neutron và cho phép khởi động dịch vụ cùng hệ thống
 
```sh
systemctl enable neutron-server.service 
systemctl enable neutron-openvswitch-agent.service 
systemctl enable neutron-dhcp-agent.service 
systemctl enable neutron-metadata-agent.service

systemctl start neutron-server.service 
systemctl start neutron-openvswitch-agent.service 
systemctl start neutron-dhcp-agent.service 
systemctl start neutron-metadata-agent.service
```

 - Kiểm tra dịch vụ neutron
 
`. admin-rc`

`neutron agent-list`

![ops](/ManhDV/OpenStack/images/neutron.png)

### 2.3.5 Cài đặt và cấu hình Horizon <a name="2.3.5"> </a> 

 - Cài đặt horizon

`yum install openstack-dashboard -y`

 - Sao lưu file cấu hình cho dashboard
 
`cp /etc/openstack-dashboard/local_settings  /etc/openstack-dashboard/local_settings.bka`

 - Sửa file cấu hình /etc/openstack-dashboard/local_settings
 
```sh
OPENSTACK_HOST = "172.16.69.10"

ALLOWED_HOSTS = ['*', ]

SESSION_ENGINE = 'django.contrib.sessions.backends.cache'

CACHES = {
    'default': {
         'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
         'LOCATION': '172.16.69.10:11211',
    }
}

OPENSTACK_KEYSTONE_URL = "http://%s:5000/v3" % OPENSTACK_HOST

OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True

OPENSTACK_API_VERSIONS = {
    "identity": 3,
    "image": 2,
    "volume": 2,
}

OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = "default"

OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"

OPENSTACK_NEUTRON_NETWORK = {
    ...
    'enable_router': False,
    'enable_quotas': False,
    'enable_distributed_router': False,
    'enable_ha_router': False,
    'enable_lb': False,
    'enable_firewall': False,
    'enable_vpn': False,
    'enable_fip_topology_check': False,
}

```

 - Restart dịch vụ

`systemctl restart httpd.service memcached.service`


# 3 Cài đặt trên Compute  <a name="3"> </a> 
## 3.1 Setup môi trường cài đặt <a name="3.1"> </a> 

 - Setup bonding cho node COM. Tham khảo link [sau](https://github.com/meditechopen/mdt-technical/blob/master/ManhDV/OpenStack/Caidat-bonding.md)
 
 - Cấu hình file hosts
 
`vi /etc/hosts`
 
```sh
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
172.16.69.10    controller
172.16.69.20    compute
```

`vi /etc/resolv.conf`

```sh
nameserver 8.8.8.8
```

 - Kiểm tra ping ra Internet
 
`ping google.com`

 - Config cho các module network
 
```sh
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
echo "net.ipv4.conf.all.rp_filter=0" >> /etc/sysctl.conf
echo "net.ipv4.conf.default.rp_filter=0" >> /etc/sysctl.conf
sysctl -p
```

 - Đăng ký tài khoản RHEL (Người dùng đã phải đăng ký bằng mail trên website của RHEL)
 
```sh
subscription-manager register --username="user" --password="userpassword" --auto-attach

subscription-manager repos --enable=rhel-7-server-optional-rpms --enable=rhel-7-server-extras-rpms --enable=rhel-7-server-rh-common-rpms
```

 - Tắt firewall và selinux
```sh
systemctl disable firewalld
systemctl stop firewalld
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
```

 - Khởi động lại máy
 
`init 6`


 -  Add repo và update hệ thống

`yum install -y https://repos.fedorapeople.org/repos/openstack/openstack-mitaka/rdo-release-mitaka-6.noarch.rpm`

`yum upgrade -y`

 - Cài đặt byobu và wget 
 
`yum install -y http://dl.fedoraproject.org/pub/epel/7/x86_64/b/byobu-5.73-4.el7.noarch.rpm wget`

 - Chạy lệnh byobu
 
`byobu`

 - Cài đặt openstack-client để sử dụng các câu lệnh openstack
 
`yum install python-openstackclient -y`

 - Cài đặt gói openstack-selinux để quản lý policy cho các service openstack.
 
`yum install openstack-selinux -y`

## 3.2 Cài đặt các thành phần phụ trợ <a name="3.2"> </a> 

### 3.2.1 Cài đặt NTP <a name="3.2.1"> </a> 
 
`yum install chrony -y `

 - Chỉnh sửa file cấu hình /etc/chrony/chrony.conf
 
```sh
sed -i "s/server 0.rhel.pool.ntp.org iburst/server 172.16.69.10 iburst/g" /etc/chrony.conf

sed -i 's/server 1.rhel.pool.ntp.org iburst/#server 1.rhel.pool.ntp.org iburst/g' /etc/chrony.conf

sed -i 's/server 2.rhel.pool.ntp.org iburst/#server 2.rhel.pool.ntp.org iburst/g' /etc/chrony.conf

sed -i 's/server 3.rhel.pool.ntp.org iburst/#server 3.rhel.pool.ntp.org iburst/g' /etc/chrony.conf

```
 - Restart dịch vụ ntp
 
```sh
systemctl enable chronyd.service
systemctl start chronyd.service
```
 - Chạy lệnh kiểm tra trên COM
 
`chronyc sources`

![ops](/ManhDV/OpenStack/images/ntp.png)

## 3.3 Cài đặt các thành phần lõi <a name="3.3"> </a> 

### 3.3.1 Cài đặt và cấu hình Nova <a name="3.3.1"> </a> 
 
`yum install openstack-nova-compute -y`

 - Sao lưu file cấu hình /etc/nova/nova.conf
 
`cp /etc/nova/nova.conf /etc/nova/nova.conf.bka`

 - Sửa file cấu hình /etc/nova/nova.conf
 
```sh
[DEFAULT]
my_ip = 172.16.69.20
rpc_backend = rabbit
auth_strategy = keystone
use_neutron = True
firewall_driver = nova.virt.firewall.NoopFirewallDriver
compute_driver=libvirt.LibvirtDriver

[oslo_messaging_rabbit]
rabbit_host = 172.16.69.10
rabbit_userid = openstack
rabbit_password = Welcome123

[libvirt]
virt_type=kvm
cpu_mode=none

[keystone_authtoken]
auth_uri = http://172.16.69.10:5000
auth_url = http://172.16.69.10:35357
memcached_servers = 172.16.69.10:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = nova
password = Welcome123

[vnc]
enabled = True
vncserver_listen = 0.0.0.0
vncserver_proxyclient_address = $my_ip
novncproxy_base_url = http://172.16.69.10:6080/vnc_auto.html

[glance]
api_servers = http://172.16.69.10:9292

[oslo_concurrency]
lock_path = /var/lib/nova/tmp
```

 - Đối với môi trường ảo hóa vmware, kiểm tra xem máy compute có hỗ trợ kvm không.
 
`egrep -c '(vmx|svm)' /proc/cpuinfo`

Nếu giá trị trả về = 0 thì sửa trong file /etc/nova/nova.conf 

```sh
[libvirt]
virt_type = qemu
```

 - Start service nova và cho phép khởi động dịch vụ cùng hệ thống
 
```sh
systemctl enable libvirtd.service openstack-nova-compute.service
systemctl start libvirtd.service openstack-nova-compute.service
```

 - Quay lại máy Controller, thực hiện kiểm tra nova
 
`. admin-rc`

`openstack compute service list`

![ops](/ManhDV/OpenStack/images/nova.png)

### 3.3.2 Cài đặt và cấu hình Neutron openvSwitch <a name="3.3.2"> </a> 

 - Cài đặt neutron openvswitch
 
`yum install -y openstack-neutron-openvswitch ebtables ipset`

 - Sao lưu file cấu hình /etc/neutron/neutron.conf
 
`cp /etc/neutron/neutron.conf /etc/neutron/neutron.conf.bka `

 - Sửa file cấu hình /etc/neutron/neutron.conf 
 
```sh
[DEFAULT]
rpc_backend = rabbit
auth_strategy = keystone
core_plugin = ml2 
notify_nova_on_port_status_changes = true
notify_nova_on_port_data_changes = true

[oslo_messaging_rabbit]
rabbit_host = 172.16.69.10
rabbit_userid = openstack
rabbit_password = Welcome123

[keystone_authtoken]
auth_uri = http://172.16.69.10:5000
auth_url = http://172.16.69.10:35357
memcached_servers = 172.16.69.10:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = Welcome123

[oslo_concurrency]
lock_path = /var/lib/neutron/tmp
```

 - Sao lưu file cấu hình openvswitch_agent.ini
 
`cp /etc/neutron/plugins/ml2/openvswitch_agent.ini /etc/neutron/plugins/ml2/openvswitch_agent.ini.bka`

 - Sửa file cấu hình openvswitch_agent.ini
 
```sh
[ovs]
bridge_mappings = provider:br-provider

[securitygroup]
firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
enable_security_group = True
```

 - Cấu hình neutron tròn file /etc/nova/nova.conf
 
```sh
[neutron]
url = http://172.16.69.10:9696
auth_url = http://172.16.69.10:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = Welcome123 
```
 
 - Restart dịch vụ nova
 
`systemctl restart openstack-nova-compute.service`

 - Start neutron và cho phép khởi động dịch vụ trực tiếp cùng hệ thống.
 
```sh
systemctl enable neutron-openvswitch-agent.service
systemctl start neutron-openvswitch-agent.service
```
 - Tạo OVS provider
 
`ovs-vsctl add-br br-provider`

 - Gán interface provider vào OVS provider
 
`ovs-vsctl add-port br-provider eno33554960`

 - Sao lưu file cấu hình ifcfg-eno33554960
 
`cp /etc/sysconfig/network-scripts/ifcfg-eno33554960 /etc/sysconfig/network-scripts/ifcfg-eno33554960.bka`

 - Tạo file cấu hình /etc/sysconfig/network-scripts/ifcfg-eno33554960 mới
 
```sh
DEVICE=eno33554960
NAME=eno33554960
DEVICETYPE=ovs
TYPE=OVSPort
OVS_BRIDGE=br-provider
ONBOOT=yes
BOOTPROTO=none
NM_CONTROLLED=no
```

 - Tạo file cấu hình /etc/sysconfig/network-scripts/ifcfg-br-provider mới

```sh
ONBOOT=yes
IPADDR=172.16.69.20
NETMASK=255.255.255.0
GATEWAY=172.16.69.1
DNS=8.8.8.8
DEVICE=br-provider
NAME=br-provider
DEVICETYPE=ovs
OVSBOOTPROTO=none
TYPE=OVSBridge
```
 - Restart network
 
`systemctl restart network`

cd

 - Restart dịch vụ OVS agent
 
`systemctl restart neutron-openvswitch-agent.service`

 - Quay lại kiểm tra trên Controller
 
`. admin-rc`

`openstack network agent list`

 - Tạo máy ảo với image cirros, flavor tiny và dải mạng external_network. Quay lại node CTL.
 
`. admin-rc`
 
```sh
openstack server create mdt-cirros --image cirros  --flavor m1.tiny --nic net-id=external_network
```

 - Setup **Default rules` cho các vm
 
![ops](/ManhDV/OpenStack/images/default-rule.png) 
 
 - Kiểm tra trên dashboard, đăng nhập và kiểm tra PING và SSH vào máy đã tạo.
 
![ops](/ManhDV/OpenStack/images/test-vm-01.png) 

![ops](/ManhDV/OpenStack/images/test-vm-02.png) 
 

# 4 Cài đặt mô hình network Self-service <a name="4"> </a> 


 - Tắt 2 máy CTL và COM, sau đó add thêm card VMnet2 cho cả 2 máy. Xuất hiện card ens39 thuộc VMnet2.

## 4.1 Thực hiện trên node Controller <a name="4.1"> </a> 

 - Sửa cấu hình file /etc/neutron/neutron.conf 
 
```sh
[DEFAULT]
verbose = True
core_plugin = ml2
service_plugins = router
allow_overlapping_ips = True
```
 
 - Sửa file cấu hình /etc/neutron/plugins/ml2/ml2_conf.ini
 
```sh
[ml2]
type_drivers = flat,vlan,gre,vxlan
tenant_network_types = vlan,gre,vxlan
mechanism_drivers = openvswitch,l2population
extension_drivers = port_security

[ml2_type_flat]
flat_networks = provider

[ml2_type_vlan]
network_vlan_ranges = provider,vlan:1:100

[ml2_type_gre]
tunnel_id_ranges = 1:100

[ml2_type_vxlan]
vni_ranges = 1:100

[securitygroup]
enable_ipset = True
```

 - Sửa file /etc/neutron/plugins/ml2/openvswitch_agent.ini
 
```sh
[ovs]
local_ip = 192.168.11.10
bridge_mappings = vlan:br-vlan,provider:br-provider

[agent]
tunnel_types = gre,vxlan
l2_population = True
prevent_arp_spoofing = True

[securitygroup]
firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
enable_security_group = True
```

 - Sửa file /etc/neutron/l3_agent.ini
 
```sh
[DEFAULT]
verbose = True
interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
use_namespaces = True
external_network_bridge =
```

 - Sửa file /etc/neutron/dhcp_agent.ini
 
```sh
[DEFAULT]
verbose = True
interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = True
dnsmasq_config_file = /etc/neutron/dnsmasq-neutron.conf
```

 - Tạo file /etc/neutron/dnsmasq-neutron.conf
 
`vi /etc/neutron/dnsmasq-neutron.conf`
 
```sh
dhcp-option-force=26,1450
```

 - Tạo OVS br-vlan
 
`ovs-vsctl add-br br-vlan`

 - Gán interface vlan vào OVS br-vlan
 
`ovs-vsctl add-port br-vlan ens39`

 - Sao lưu file cấu hình ifcfg-ens39
 
`cp /etc/sysconfig/network-scripts/ifcfg-ens39 /etc/sysconfig/network-scripts/ifcfg-ens39.bka`

 - Tạo file cấu hình /etc/sysconfig/network-scripts/ifcfg-ens39 mới
 
```sh
DEVICE=ens39
NAME=ens39
DEVICETYPE=ovs
TYPE=OVSPort
OVS_BRIDGE=br-vlan
ONBOOT=yes
BOOTPROTO=none
NM_CONTROLLED=no
```

 - Tạo file cấu hình /etc/sysconfig/network-scripts/ifcfg-br-vlan mới

```sh
ONBOOT=yes
DEVICE=br-vlan
NAME=br-vlan
DEVICETYPE=ovs
OVSBOOTPROTO=none
TYPE=OVSBridge
```
 - Restart network
 
`systemctl restart network`

 - Khởi động lại các dịch vụ :
 
```sh
systemctl restart openvswitch.service
systemctl restart neutron-server.service 
systemctl restart neutron-openvswitch-agent.service 
systemctl restart neutron-dhcp-agent.service 
systemctl restart neutron-metadata-agent.service
systemctl restart neutron-l3-agent
```

## 4.2 Thực hiện trên node Compute <a name="4.2"> </a> 

 - Sửa file /etc/neutron/plugins/ml2/openvswitch_agent.ini
 
```sh
[ovs]
local_ip = 192.168.11.20
bridge_mappings = vlan:br-vlan

[agent]
tunnel_types = gre,vxlan
l2_population = True
prevent_arp_spoofing = True

[securitygroup]
firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
enable_security_group = True
```

 - Tạo OVS br-vlan
 
`ovs-vsctl add-br br-vlan`

 - Gán interface vlan vào OVS br-vlan
 
`ovs-vsctl add-port br-vlan ens39`

 - Sao lưu file cấu hình ifcfg-ens39
 
`cp /etc/sysconfig/network-scripts/ifcfg-ens39 /etc/sysconfig/network-scripts/ifcfg-ens39.bka`

 - Tạo file cấu hình /etc/sysconfig/network-scripts/ifcfg-ens39 mới
 
```sh
DEVICE=ens39
NAME=ens39
DEVICETYPE=ovs
TYPE=OVSPort
OVS_BRIDGE=br-vlan
ONBOOT=yes
BOOTPROTO=none
NM_CONTROLLED=no
```

 - Tạo file cấu hình /etc/sysconfig/network-scripts/ifcfg-br-vlan mới

```sh
ONBOOT=yes
DEVICE=br-vlan
NAME=br-vlan
DEVICETYPE=ovs
OVSBOOTPROTO=none
TYPE=OVSBridge
```
 - Restart network
 
`systemctl restart network`

 - Khởi động lại các dịch vụ :
 
```sh
systemctl restart openvswitch.service
systemctl restart neutron-openvswitch-agent.service 
```

## 4.3 Kiểm tra <a name="4.3"> </a> 

 - Đúng trên CTL, `source admin-rc`, sau đó kiểm tra neutron
 
`neutron agent-list`

![ops](/ManhDV/OpenStack/images/neutron-agent-list.png)


# 5. Tạo máy ảo <a name="5"> </a> 

 - Tạo network public
 
```sh
neutron net-create external_network --provider:network_type flat \
--provider:physical_network provider  \
--router:external \
--shared
```

 - Tạo subnet trong network public
 
```sh
neutron subnet-create --name public_subnet \
--enable_dhcp=True --dns-nameserver 8.8.8.8 \
--allocation-pool=start=172.16.69.80,end=172.16.69.100 \
--gateway=172.16.69.1 external_network 172.16.69.0/24
```

 - Tạo network private

```sh
neutron net-create private_network --provider:network_type vxlan

neutron subnet-create --name private_subnet private_network 10.0.0.0/24 \
--dns-nameserver 8.8.8.8
```

 - Tạo router và addd các interface

```sh
neutron router-create router
neutron router-gateway-set router external_network
neutron router-interface-add router private_subnet
```

 - Kiểm tra các network đang có
 
`ip netns`

```
qrouter-4d7928a0-4a3c-4b99-b01b-97da2f97e279
qdhcp-353f5937-a2d3-41ba-8225-fa1af2538141
```

 - Kiểm tra các dải mạng đang gắn vào router
 
`neutron router-port-list router`

![ops](/ManhDV/OpenStack/images/router.png)

 - Tạo máy ảo và ping đến gateway ip của router để kiểm tra
 
```sh
openstack server create mdt --image cirros  --flavor m1.tiny --nic net-id=e111bed7-aa46-4489-b52b-de7d3baf5614 
```

 - Ping ra gateway 10.0.0.1 và ping ra Internet để kiểm tra
 
 - Tạo floating IP cho VM
 
`neutron floatingip-create external_network`
![ops](/ManhDV/OpenStack/images/floatingip.png)

 - Gán IP floating cho vm
 
`nova floating-ip-associate vm1 172.16.69.84`

 - Đăng nhập SSH với IP này kiểm tra.

