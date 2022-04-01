## Install Manila 

1. Chuẩn bị Database
mysql -u root -px9OaxZllwg
CREATE DATABASE manila;
GRANT ALL PRIVILEGES ON manila.* TO 'manila'@'localhost' IDENTIFIED BY 'kOSCHwsnnfOeIbWVWYJ4';
GRANT ALL PRIVILEGES ON manila.* TO 'manila'@'%' IDENTIFIED BY 'kOSCHwsnnfOeIbWVWYJ4';
quit;


2. Thêm các endpoint

openstack user create --domain default --password-prompt manila
openstack role add --project service --user manila admin
openstack service create --name manila --description "OpenStack Shared File Systems" share
openstack service create --name manilav2 --description "OpenStack Shared File Systems V2" sharev2

openstack endpoint create --region regionOne share public http://172.18.0.49:8786/v1/%\(tenant_id\)s
openstack endpoint create --region regionOne share internal http://172.18.0.49:8786/v1/%\(tenant_id\)s
openstack endpoint create --region regionOne share admin http://172.18.0.49:8786/v1/%\(tenant_id\)s
openstack endpoint create --region regionOne sharev2 public http://172.18.0.49:8786/v2/%\(tenant_id\)s
openstack endpoint create --region regionOne sharev2 internal http://172.18.0.49:8786/v2/%\(tenant_id\)s
openstack endpoint create --region regionOne sharev2 admin http://172.18.0.49:8786/v2/%\(tenant_id\)s



internal  | http://172.18.0.10:8776/v2/%(tenant_id)s       
public    | https://hn.foxcloud.vn:13776/v2/%(tenant_id)s  
admin     | http://172.18.0.10:8776/v2/%(tenant_id)s       



3. Cài đặt và cấu hình Manila
Thêm vào file host của manila : 

echo "172.20.0.51   glusterfs-01     glusterfs-01.example.com" >> /etc/hosts
echo "172.20.0.52   glusterfs-02     glusterfs-02.example.com" >> /etc/hosts
echo "172.20.0.53   glusterfs-03     glusterfs-03.example.com" >> /etc/hosts
echo "172.20.0.54   glusterfs-04     glusterfs-04.example.com" >> /etc/hosts


sudo dnf install -y sudo dnf install -y https://trunk.rdoproject.org/centos8/component/tripleo/current/python3-tripleo-repos-0.1.1-0.20200909062930.1c4a717.el8.noarch.rpm

sudo -E tripleo-repos -b ussuri current ceph
sudo dnf install -y https://trunk.rdoproject.org/centos8/component/tripleo/current/python3-tripleo-repos-0.1.1-0.20200909062930.1c4a717.el8.noarch.rpm
sudo -E tripleo-repos -b ussuri current ceph
sudo dnf install -y python3-tripleoclient
yum install openstack-manila python3-manilaclient openstack-manila-share python2-PyMySQL -y

/etc/manila/manila.conf 
[DEFAULT]
state_path=/var/lib/manila
host=hostgroup
storage_availability_zone=nova
default_share_type=default
rootwrap_config=/etc/manila/rootwrap.conf
auth_strategy=keystone
enabled_share_backends = glusterfs
enabled_share_protocols = NFS,CIFS,GLUSTERFS
network_api_class=manila.network.neutron.neutron_network_plugin.NeutronNetworkPlugin
network_plugin_ipv4_enabled=True
network_plugin_ipv6_enabled=False
osapi_share_listen=172.18.0.49
osapi_share_workers=12
debug=True
log_dir=/var/log/manila
transport_url=rabbit://guest:CZEzedT35fv27xJuI5VAS9bNh@cas-hn-controller-01.internalapi.localdomain:5672,guest:CZEzedT35fv27xJuI5VAS9bNh@cas-hn-controller-02.internalapi.localdomain:5672,guest:CZEzedT35fv27xJuI5VAS9bNh@cas-hn-controller-03.internalapi.localdomain:5672/?ssl=0
control_exchange=openstack
api_paste_config=/etc/manila/api-paste.ini
[cinder]
[cors]
[database]
connection=mysql+pymysql://manila:kOSCHwsnnfOeIbWVWYJ4@172.18.0.10/manila?read_default_file=/etc/my.cnf.d/tripleo.cnf&read_default_group=tripleo
max_retries=-1
db_max_retries=-1
[healthcheck]
[keystone_authtoken]
www_authenticate_uri=http://172.18.0.10:5000
region_name=regionOne
memcached_servers=172.18.0.11:11211,172.18.0.12:11211,172.18.0.13:11211
auth_type=password
auth_url=http://172.18.0.10:5000
username=manila
password = kOSCHwsnnfOeIbWVWYJ4
user_domain_name=Default
project_name=service
project_domain_name=Default
[neutron]
region_name=regionOne
auth_url=http://172.18.0.10:5000
auth_type=password
password=kOSCHwsnnfOeIbWVWYJ4
project_domain_name=Default
project_name=service
user_domain_name=Default
username=manila
[nova]
region_name=regionOne
auth_url=http://172.18.0.10:5000
auth_type=password
password=kOSCHwsnnfOeIbWVWYJ4
project_domain_name=Default
project_name=service
user_domain_name=Default
username=manila
[oslo_concurrency]
lock_path=/tmp/manila/manila_locks
[oslo_messaging_amqp]
container_name=guest
idle_timeout=0
trace=False
server_request_prefix=exclusive
broadcast_prefix=broadcast
group_request_prefix=unicast
allow_insecure_clients=False
[oslo_messaging_kafka]
[oslo_messaging_notifications]
driver=noop
transport_url=rabbit://guest:CZEzedT35fv27xJuI5VAS9bNh@cas-hn-controller-01.internalapi.localdomain:5672,guest:CZEzedT35fv27xJuI5VAS9bNh@cas-hn-controller-02.internalapi.localdomain:5672,guest:CZEzedT35fv27xJuI5VAS9bNh@cas-hn-controller-03.internalapi.localdomain:5672/?ssl=0
[oslo_messaging_rabbit]
[oslo_middleware]
enable_proxy_headers_parsing=True
[oslo_policy]
policy_file=/etc/manila/policy.json
[ssl]
[glusterfs]
driver_handles_share_servers = False
share_backend_name = glusterfs
share_driver = manila.share.drivers.glusterfs.GlusterfsShareDriver
#glusterfs_target = root@172.20.0.51:/manila
#glusterfs_mount_point_base = $state_path/mnt
#glusterfs_share_layout = layout_directory.GlusterfsDirectoryMappedLayout
glusterfs_nfs_server_type = Gluster
glusterfs_server_password = Cas@2020
glusterfs_servers = root@172.20.0.51
glusterfs_volume_pattern = manila
glusterfs_share_layout = layout_volume.GlusterfsVolumeMappedLayout



Sync dữ liệu database
su -s /bin/sh -c "manila-manage db sync" manila

4. Restart service
systemctl enable openstack-manila-api.service openstack-manila-scheduler.service openstack-manila-share.service
systemctl restart openstack-manila-api.service openstack-manila-scheduler.service openstack-manila-share.service