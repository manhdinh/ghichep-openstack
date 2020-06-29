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

### 1.1. Bước 1 : Setup stack user và cài đặt 1 số gói cần thiết

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
local_ip = 10.0.13.80/24
undercloud_admin_host = 10.0.13.120
undercloud_public_host = 10.0.13.121
[ctlplane-subnet]
cidr = 10.0.13.0/24
dhcp_end = 10.0.13.110
dhcp_start = 10.0.13.84
dns_nameservers = 8.8.8.8
gateway = 10.0.13.1
inspection_iprange = 10.0.13.122,10.0.13.148
```

Đối với môi trường LAB trên ESXI, thực hiện thêm dòng cấu hình sau để ebabke :

```sh
enabled_hardware_types=idrac,ilo,ipmi,redfish,manual-management
```

Thực hiện hạ phiên bản của leatherman để tương thích với phiên bản puppet : 

```sh
sudo yum downgrade leatherman -y
```

Thực hiện cài đặt undercloud :
```sh
byobu
restorecon -R -v /etc/hostname
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


### 2.3. Bước 3 : Tạo vm cho Controller và Compute (Thực hiện trên KVM)

Trên node ESXI, thực hiện tạo các máy ảo Controller1,2,3 và Compute1,2,3.

Cấu trúc phần cứng của node Controller :

![tripleo](/OpenStack/images/tripleo-esxi-00.png)

Cấu trúc phần cứng của node Compute :

![tripleo](/OpenStack/images/tripleo-esxi-01.png)

### 2.5. Bước 5 : Tạo và Import Inventory cho Overcloud node

Tại KVM, thực hiện lấy MAC của các máy ảo Controller và Compute với VLAN 13 - PROVISION. (`NODE=> Configure => Networking => Virtual switches => VLAN13`)

![tripleo](/OpenStack/images/tripleo-esxi-02.png)

Trên Undercloud node, tạo một file JSON với tên `overcloud-stackenv.json`

```sh
vi nodes.json
{
      "nodes": [
        {
          "name": "controller01",
          "memory": "8192",
          "mac": [
            "00:50:56:aa:60:f8"
          ],
          "pm_type": "manual-management",
          "disk": "100",
          "arch": "x86_64",
          "cpu": "8"
        },
        {
          "name": "controller02",
          "memory": "8192",
          "mac": [
            "00:50:56:aa:ea:5f"
          ],
          "pm_type": "manual-management",
          "disk": "100",
          "arch": "x86_64",
          "cpu": "8"
        },
        {
          "name": "controller03",
          "memory": "8192",
          "mac": [
            "00:50:56:aa:61:6b"
          ],
          "pm_type": "manual-management",
          "disk": "100",
          "arch": "x86_64",
          "cpu": "8"
        },
        {
          "name": "compute01",
          "memory": "12288",
          "mac": [
            "00:50:56:aa:d6:63"
          ],
          "pm_type": "manual-management",
          "disk": "100",
          "arch": "x86_64",
          "cpu": "12"
        },
        {
          "name": "compute02",
          "memory": "12288",
          "mac": [
            "00:50:56:aa:b9:be"
          ],
          "pm_type": "manual-management",
          "disk": "100",
          "arch": "x86_64",
          "cpu": "12"
        },
        {
          "name": "compute03",
          "memory": "12288",
          "mac": [
            "00:50:56:aa:f0:76"
          ],
          "pm_type": "manual-management",
          "disk": "100",
          "arch": "x86_64",
          "cpu": "12"
         }
      ]
}

```

Thay thế MAC phù hợp với các node Controller và Compute ở trên.

Thực hiện check cú pháp khai báo của file :

```sh
source stack
openstack overcloud node import --validate-only ~/nodes.json
```

Sau khi cú pháp đã OK. Thực hiện import file

```sh
openstack overcloud node import ~/nodes.json
```

Do TripleO dùng `fake_pxe` driver để quản lý các node. Do vậy bạn cần xử lý việc bật tắt các node bằng tay. 

Sau khi import thành công, các node sẽ chuyển trạng thái là `manageable`. Lúc đó thực hiện bật các node để load các image kernel và image vmzlinux.

![tripleo](/OpenStack/images/tripleo-esxi-04.png)

![tripleo](/OpenStack/images/tripleo-esxi-03.png)

Thực hiện chạy kiểm tra trước khi thực hiện inspect các node

```sh
source stackrc
openstack tripleo validator run --group pre-introspection
```
![tripleo](/OpenStack/images/tripleo-esxi-05.png)

Thực hiện quá trình `introspection` với các node như sau :

```sh
openstack overcloud node introspect --all-manageable --provide
```

![tripleo](/OpenStack/images/tripleo-esxi-06.png)


Kiểm tra trạng thái của các node, khi trạng thái các node chuyển sang `power-on` thì bật các node lên. (Nên bật lần lượt từng node, đợi tới khi node đã load xong image thì bật node tiếp theo)

![tripleo](/OpenStack/images/tripleo-esxi-07.png)

Sau khi tripleo inspect thành công các node thì trạng thái các node sẽ chuyển sang `power off` và `available`. Lúc này shutdown các máy.

![tripleo](/OpenStack/images/tripleo-esxi-08.png)

Chú ý : Việc download image và boot tới các node được thực hiện qua đường Provision. Trong môi trường LAB nếu nhiều node cùng thực hiện download sẽ dẫn tới timeout tại console. Khi có hiện tượng cần reset node để download lại.

Kiểm tra trạng thái các node như sau :

```sh
openstack baremetal node list
```

![tripleo](/OpenStack/images/tripleo-esxi-09.png)

Thực hiện set Role cho các node. 

```sh
openstack baremetal node set --property capabilities='node:controller-0,profile:control,boot_option:local' $controller1_ID
openstack baremetal node set --property capabilities='node:controller-1,profile:control,boot_option:local' $controller2_ID
openstack baremetal node set --property capabilities='node:controller-2,profile:control,boot_option:local' $controller3_ID
openstack baremetal node set --property capabilities='node:compute-0,profile:compute,boot_option:local' $compute1_ID
openstack baremetal node set --property capabilities='node:compute-1,profile:compute,boot_option:local' $compute2_ID
openstack baremetal node set --property capabilities='node:compute-2,profile:compute,boot_option:local' $compute3_ID

```


Kiểm tra với câu lệnh :
```sh
openstack overcloud profiles list
```

![tripleo](/OpenStack/images/tripleo-20.png)

Trạng thái của `Provision State` cần là `available` như trên.

## 3 Xử lý template

Tạo thư mục cho overcloud-templates :

```sh
mkdir /home/stack/custom-templates/
```

Render toàn bộ các file dạng Jinja2 sang dạng YAML 

```sh
cd /usr/share/openstack-tripleo-heat-templates
./tools/process-templates.py -o ~/openstack-tripleo-heat-templates-rendered
```


### 3.1. Template `01-roles-data.yaml`


Copy file template sang thư mục config

```sh
cp /usr/share/openstack-tripleo-heat-templates/roles_data.yaml /home/stack/custom-templates/01-roles-data.yaml
```

Chỉnh sửa giá trị `CountDefault` tại section Controller và Compute thành `3`


### 3.2. Template `02-node-info.yaml`

Tạo file `/home/stack/custom-templates/02-node-info.yaml` :


```sh
parameter_defaults:
  OvercloudControllerFlavor: baremetal
  OvercloudComputeFlavor: baremetal
  ControllerCount: 3
  ComputeCount: 3
```

### 3.3. Tạo file `03-network-isolation.yaml`

```sh
# Enable the creation of Neutron networks for isolated Overcloud
# traffic and configure each role to assign ports (related
# to that role) on these networks.
resource_registry:
  # networks as defined in network_data.yaml
  OS::TripleO::Network::Storage: /usr/share/openstack-tripleo-heat-templates/network/storage.yaml
  #OS::TripleO::Network::StorageMgmt: /usr/share/openstack-tripleo-heat-templates/network/storage_mgmt.yaml
  OS::TripleO::Network::InternalApi: /usr/share/openstack-tripleo-heat-templates/network/internal_api.yaml
  OS::TripleO::Network::Tenant: /usr/share/openstack-tripleo-heat-templates/network/tenant.yaml
  OS::TripleO::Network::External: /usr/share/openstack-tripleo-heat-templates/network/external.yaml
  #OS::TripleO::Network::Management: ../network/management.yaml

  # Port assignments for the VIPs
  OS::TripleO::Network::Ports::StorageVipPort: /usr/share/openstack-tripleo-heat-templates/network/ports/storage.yaml
  #OS::TripleO::Network::Ports::StorageMgmtVipPort: /usr/share/openstack-tripleo-heat-templates/network/ports/storage_mgmt.yaml
  OS::TripleO::Network::Ports::InternalApiVipPort: /usr/share/openstack-tripleo-heat-templates/network/ports/internal_api.yaml
  OS::TripleO::Network::Ports::ExternalVipPort: /usr/share/openstack-tripleo-heat-templates/network/ports/external.yaml
  OS::TripleO::Network::Ports::RedisVipPort: /usr/share/openstack-tripleo-heat-templates/network/ports/vip.yaml
  #OS::TripleO::Network::Ports::OVNDBsVipPort: /usr/share/openstack-tripleo-heat-templates/network/ports/vip.yaml

  # Port assignments by role, edit role definition to assign networks to roles.
  # Port assignments for the Controller
  OS::TripleO::Controller::Ports::RedisVipPort: /usr/share/openstack-tripleo-heat-templates/network/ports/vip.yaml
  OS::TripleO::Controller::Ports::StoragePort: /usr/share/openstack-tripleo-heat-templates/network/ports/storage.yaml
  #OS::TripleO::Controller::Ports::StorageMgmtPort: /usr/share/openstack-tripleo-heat-templates/network/ports/storage_mgmt.yaml
  OS::TripleO::Controller::Ports::InternalApiPort: /usr/share/openstack-tripleo-heat-templates/network/ports/internal_api.yaml
  OS::TripleO::Controller::Ports::TenantPort: /usr/share/openstack-tripleo-heat-templates/network/ports/tenant.yaml
  OS::TripleO::Controller::Ports::ExternalPort: /usr/share/openstack-tripleo-heat-templates/network/ports/external.yaml
  # Port assignments for the Compute
  OS::TripleO::Compute::Ports::StoragePort: /usr/share/openstack-tripleo-heat-templates/network/ports/storage.yaml
  OS::TripleO::Compute::Ports::InternalApiPort: /usr/share/openstack-tripleo-heat-templates/network/ports/internal_api.yaml
  OS::TripleO::Compute::Ports::TenantPort: /usr/share/openstack-tripleo-heat-templates/network/ports/tenant.yaml
  # Port assignments for the BlockStorage
  OS::TripleO::BlockStorage::Ports::StoragePort: /usr/share/openstack-tripleo-heat-templates/network/ports/storage.yaml
  #OS::TripleO::BlockStorage::Ports::StorageMgmtPort: /usr/share/openstack-tripleo-heat-templates/network/ports/storage_mgmt.yaml
  OS::TripleO::BlockStorage::Ports::InternalApiPort: /usr/share/openstack-tripleo-heat-templates/network/ports/internal_api.yaml
  # Port assignments for the ObjectStorage
  OS::TripleO::ObjectStorage::Ports::StoragePort: /usr/share/openstack-tripleo-heat-templates/network/ports/storage.yaml
  #OS::TripleO::ObjectStorage::Ports::StorageMgmtPort: /usr/share/openstack-tripleo-heat-templates/network/ports/storage_mgmt.yaml
  OS::TripleO::ObjectStorage::Ports::InternalApiPort: /usr/share/openstack-tripleo-heat-templates/network/ports/internal_api.yaml
```

### 3.4. Tạo file `04-overcloud_images.yaml`

Tạo file `/home/stack/custom-templates/04-overcloud_images.yaml`

```sh
openstack overcloud container image prepare \
  --push-destination=10.0.13.80:8787 \
  --namespace=docker.io/tripleotrain/ \
  --output-env-file=/home/stack/custom-templates/04-overcloud_images.yaml \
  --prefix=openstack- \
  --output-images-file /home/stack/custom-templates/local_registry_images.yaml
```

Nội dung file sẽ như sau : 

```sh
# Generated with the following on 2020-06-29T17:57:06.899964
#
#   openstack overcloud container image prepare --push-destination=10.0.13.80:8787 --namespace=docker.io/tripleotrain/ --output-env-file=/home/stack/custom-templates/04-overcloud_images.yaml --prefix=openstack- --output-images-file /home/stack/custom-templates/local_registry_images.yaml
#

parameter_defaults:
  AlertManagerContainerImage: 10.0.13.80:8787/prom/alertmanager:v0.16.2
  ContainerAodhApiImage: 10.0.13.80:8787/tripleotrain//openstack-aodh-api:current-tripleo
  ContainerAodhConfigImage: 10.0.13.80:8787/tripleotrain//openstack-aodh-api:current-tripleo
  ContainerAodhEvaluatorImage: 10.0.13.80:8787/tripleotrain//openstack-aodh-evaluator:current-tripleo
  ContainerAodhListenerImage: 10.0.13.80:8787/tripleotrain//openstack-aodh-listener:current-tripleo
  ContainerAodhNotifierImage: 10.0.13.80:8787/tripleotrain//openstack-aodh-notifier:current-tripleo
  ContainerBarbicanApiImage: 10.0.13.80:8787/tripleotrain//openstack-barbican-api:current-tripleo
  ContainerBarbicanConfigImage: 10.0.13.80:8787/tripleotrain//openstack-barbican-api:current-tripleo
  ContainerBarbicanKeystoneListenerConfigImage: 10.0.13.80:8787/tripleotrain//openstack-barbican-keystone-listener:current-tripleo
  ContainerBarbicanKeystoneListenerImage: 10.0.13.80:8787/tripleotrain//openstack-barbican-keystone-listener:current-tripleo
  ContainerBarbicanWorkerConfigImage: 10.0.13.80:8787/tripleotrain//openstack-barbican-worker:current-tripleo
  ContainerBarbicanWorkerImage: 10.0.13.80:8787/tripleotrain//openstack-barbican-worker:current-tripleo
  ContainerCeilometerCentralImage: 10.0.13.80:8787/tripleotrain//openstack-ceilometer-central:current-tripleo
  ContainerCeilometerComputeImage: 10.0.13.80:8787/tripleotrain//openstack-ceilometer-compute:current-tripleo
  ContainerCeilometerConfigImage: 10.0.13.80:8787/tripleotrain//openstack-ceilometer-central:current-tripleo
  ContainerCeilometerNotificationImage: 10.0.13.80:8787/tripleotrain//openstack-ceilometer-notification:current-tripleo
  ContainerCephDaemonImage: 10.0.13.80:8787/ceph/daemon:v4.0.12-stable-4.0-nautilus-centos-7-x86_64
  ContainerCinderApiImage: 10.0.13.80:8787/tripleotrain//openstack-cinder-api:current-tripleo
  ContainerCinderBackupImage: 10.0.13.80:8787/tripleotrain//openstack-cinder-backup:current-tripleo
  ContainerCinderConfigImage: 10.0.13.80:8787/tripleotrain//openstack-cinder-api:current-tripleo
  ContainerCinderSchedulerImage: 10.0.13.80:8787/tripleotrain//openstack-cinder-scheduler:current-tripleo
  ContainerCinderVolumeImage: 10.0.13.80:8787/tripleotrain//openstack-cinder-volume:current-tripleo
  ContainerClustercheckConfigImage: 10.0.13.80:8787/tripleotrain//openstack-mariadb:current-tripleo
  ContainerClustercheckImage: 10.0.13.80:8787/tripleotrain//openstack-mariadb:current-tripleo
  ContainerCollectdConfigImage: 10.0.13.80:8787/tripleotrain//openstack-collectd:current-tripleo
  ContainerCollectdImage: 10.0.13.80:8787/tripleotrain//openstack-collectd:current-tripleo
  ContainerCrondConfigImage: 10.0.13.80:8787/tripleotrain//openstack-cron:current-tripleo
  ContainerCrondImage: 10.0.13.80:8787/tripleotrain//openstack-cron:current-tripleo
  ContainerDesignateApiImage: 10.0.13.80:8787/tripleotrain//openstack-designate-api:current-tripleo
  ContainerDesignateBackendBIND9Image: 10.0.13.80:8787/tripleotrain//openstack-designate-backend-bind9:current-tripleo
  ContainerDesignateCentralImage: 10.0.13.80:8787/tripleotrain//openstack-designate-central:current-tripleo
  ContainerDesignateConfigImage: 10.0.13.80:8787/tripleotrain//openstack-designate-worker:current-tripleo
  ContainerDesignateMDNSImage: 10.0.13.80:8787/tripleotrain//openstack-designate-mdns:current-tripleo
  ContainerDesignateProducerImage: 10.0.13.80:8787/tripleotrain//openstack-designate-producer:current-tripleo
  ContainerDesignateSinkImage: 10.0.13.80:8787/tripleotrain//openstack-designate-sink:current-tripleo
  ContainerDesignateWorkerImage: 10.0.13.80:8787/tripleotrain//openstack-designate-worker:current-tripleo
  ContainerEc2ApiConfigImage: 10.0.13.80:8787/tripleotrain//openstack-ec2-api:current-tripleo
  ContainerEc2ApiImage: 10.0.13.80:8787/tripleotrain//openstack-ec2-api:current-tripleo
  ContainerEtcdConfigImage: 10.0.13.80:8787/tripleotrain//openstack-etcd:current-tripleo
  ContainerEtcdImage: 10.0.13.80:8787/tripleotrain//openstack-etcd:current-tripleo
  ContainerGlanceApiConfigImage: 10.0.13.80:8787/tripleotrain//openstack-glance-api:current-tripleo
  ContainerGlanceApiImage: 10.0.13.80:8787/tripleotrain//openstack-glance-api:current-tripleo
  ContainerGnocchiApiImage: 10.0.13.80:8787/tripleotrain//openstack-gnocchi-api:current-tripleo
  ContainerGnocchiConfigImage: 10.0.13.80:8787/tripleotrain//openstack-gnocchi-api:current-tripleo
  ContainerGnocchiMetricdImage: 10.0.13.80:8787/tripleotrain//openstack-gnocchi-metricd:current-tripleo
  ContainerGnocchiStatsdImage: 10.0.13.80:8787/tripleotrain//openstack-gnocchi-statsd:current-tripleo
  ContainerHAProxyConfigImage: 10.0.13.80:8787/tripleotrain//openstack-haproxy:current-tripleo
  ContainerHAProxyImage: 10.0.13.80:8787/tripleotrain//openstack-haproxy:current-tripleo
  ContainerHeatApiCfnConfigImage: 10.0.13.80:8787/tripleotrain//openstack-heat-api-cfn:current-tripleo
  ContainerHeatApiCfnImage: 10.0.13.80:8787/tripleotrain//openstack-heat-api-cfn:current-tripleo
  ContainerHeatApiConfigImage: 10.0.13.80:8787/tripleotrain//openstack-heat-api:current-tripleo
  ContainerHeatApiImage: 10.0.13.80:8787/tripleotrain//openstack-heat-api:current-tripleo
  ContainerHeatConfigImage: 10.0.13.80:8787/tripleotrain//openstack-heat-api:current-tripleo
  ContainerHeatEngineImage: 10.0.13.80:8787/tripleotrain//openstack-heat-engine:current-tripleo
  ContainerHorizonConfigImage: 10.0.13.80:8787/tripleotrain//openstack-horizon:current-tripleo
  ContainerHorizonImage: 10.0.13.80:8787/tripleotrain//openstack-horizon:current-tripleo
  ContainerIronicApiConfigImage: 10.0.13.80:8787/tripleotrain//openstack-ironic-api:current-tripleo
  ContainerIronicApiImage: 10.0.13.80:8787/tripleotrain//openstack-ironic-api:current-tripleo
  ContainerIronicConductorImage: 10.0.13.80:8787/tripleotrain//openstack-ironic-conductor:current-tripleo
  ContainerIronicConfigImage: 10.0.13.80:8787/tripleotrain//openstack-ironic-pxe:current-tripleo
  ContainerIronicInspectorConfigImage: 10.0.13.80:8787/tripleotrain//openstack-ironic-inspector:current-tripleo
  ContainerIronicInspectorImage: 10.0.13.80:8787/tripleotrain//openstack-ironic-inspector:current-tripleo
  ContainerIronicNeutronAgentImage: 10.0.13.80:8787/tripleotrain//openstack-ironic-neutron-agent:current-tripleo
  ContainerIronicPxeImage: 10.0.13.80:8787/tripleotrain//openstack-ironic-pxe:current-tripleo
  ContainerIscsidConfigImage: 10.0.13.80:8787/tripleotrain//openstack-iscsid:current-tripleo
  ContainerIscsidImage: 10.0.13.80:8787/tripleotrain//openstack-iscsid:current-tripleo
  ContainerKeepalivedConfigImage: 10.0.13.80:8787/tripleotrain//openstack-keepalived:current-tripleo
  ContainerKeepalivedImage: 10.0.13.80:8787/tripleotrain//openstack-keepalived:current-tripleo
  ContainerKeystoneConfigImage: 10.0.13.80:8787/tripleotrain//openstack-keystone:current-tripleo
  ContainerKeystoneImage: 10.0.13.80:8787/tripleotrain//openstack-keystone:current-tripleo
  ContainerManilaApiImage: 10.0.13.80:8787/tripleotrain//openstack-manila-api:current-tripleo
  ContainerManilaConfigImage: 10.0.13.80:8787/tripleotrain//openstack-manila-api:current-tripleo
  ContainerManilaSchedulerImage: 10.0.13.80:8787/tripleotrain//openstack-manila-scheduler:current-tripleo
  ContainerManilaShareImage: 10.0.13.80:8787/tripleotrain//openstack-manila-share:current-tripleo
  ContainerMemcachedConfigImage: 10.0.13.80:8787/tripleotrain//openstack-memcached:current-tripleo
  ContainerMemcachedImage: 10.0.13.80:8787/tripleotrain//openstack-memcached:current-tripleo
  ContainerMetricsQdrConfigImage: 10.0.13.80:8787/tripleotrain//openstack-qdrouterd:current-tripleo
  ContainerMetricsQdrImage: 10.0.13.80:8787/tripleotrain//openstack-qdrouterd:current-tripleo
  ContainerMistralApiImage: 10.0.13.80:8787/tripleotrain//openstack-mistral-api:current-tripleo
  ContainerMistralConfigImage: 10.0.13.80:8787/tripleotrain//openstack-mistral-api:current-tripleo
  ContainerMistralEngineImage: 10.0.13.80:8787/tripleotrain//openstack-mistral-engine:current-tripleo
  ContainerMistralEventEngineImage: 10.0.13.80:8787/tripleotrain//openstack-mistral-event-engine:current-tripleo
  ContainerMistralExecutorImage: 10.0.13.80:8787/tripleotrain//openstack-mistral-executor:current-tripleo
  ContainerMultipathdConfigImage: 10.0.13.80:8787/tripleotrain//openstack-multipathd:current-tripleo
  ContainerMultipathdImage: 10.0.13.80:8787/tripleotrain//openstack-multipathd:current-tripleo
  ContainerMysqlClientConfigImage: 10.0.13.80:8787/tripleotrain//openstack-mariadb:current-tripleo
  ContainerMysqlConfigImage: 10.0.13.80:8787/tripleotrain//openstack-mariadb:current-tripleo
  ContainerMysqlImage: 10.0.13.80:8787/tripleotrain//openstack-mariadb:current-tripleo
  ContainerNeutronApiImage: 10.0.13.80:8787/tripleotrain//openstack-neutron-server-ovn:current-tripleo
  ContainerNeutronConfigImage: 10.0.13.80:8787/tripleotrain//openstack-neutron-server-ovn:current-tripleo
  ContainerNeutronDHCPImage: 10.0.13.80:8787/tripleotrain//openstack-neutron-dhcp-agent:current-tripleo
  ContainerNeutronL3AgentImage: 10.0.13.80:8787/tripleotrain//openstack-neutron-l3-agent:current-tripleo
  ContainerNeutronMetadataImage: 10.0.13.80:8787/tripleotrain//openstack-neutron-metadata-agent:current-tripleo
  ContainerNovaApiImage: 10.0.13.80:8787/tripleotrain//openstack-nova-api:current-tripleo
  ContainerNovaComputeImage: 10.0.13.80:8787/tripleotrain//openstack-nova-compute:current-tripleo
  ContainerNovaComputeIronicImage: 10.0.13.80:8787/tripleotrain//openstack-nova-compute-ironic:current-tripleo
  ContainerNovaConductorImage: 10.0.13.80:8787/tripleotrain//openstack-nova-conductor:current-tripleo
  ContainerNovaConfigImage: 10.0.13.80:8787/tripleotrain//openstack-nova-api:current-tripleo
  ContainerNovaLibvirtConfigImage: 10.0.13.80:8787/tripleotrain//openstack-nova-compute:current-tripleo
  ContainerNovaLibvirtImage: 10.0.13.80:8787/tripleotrain//openstack-nova-libvirt:current-tripleo
  ContainerNovaMetadataConfigImage: 10.0.13.80:8787/tripleotrain//openstack-nova-api:current-tripleo
  ContainerNovaMetadataImage: 10.0.13.80:8787/tripleotrain//openstack-nova-api:current-tripleo
  ContainerNovaSchedulerImage: 10.0.13.80:8787/tripleotrain//openstack-nova-scheduler:current-tripleo
  ContainerNovaVncProxyImage: 10.0.13.80:8787/tripleotrain//openstack-nova-novncproxy:current-tripleo
  ContainerOctaviaApiImage: 10.0.13.80:8787/tripleotrain//openstack-octavia-api:current-tripleo
  ContainerOctaviaConfigImage: 10.0.13.80:8787/tripleotrain//openstack-octavia-api:current-tripleo
  ContainerOctaviaDriverAgentConfigImage: 10.0.13.80:8787/tripleotrain//openstack-octavia-api:current-tripleo
  ContainerOctaviaDriverAgentImage: 10.0.13.80:8787/tripleotrain//openstack-octavia-api:current-tripleo
  ContainerOctaviaHealthManagerImage: 10.0.13.80:8787/tripleotrain//openstack-octavia-health-manager:current-tripleo
  ContainerOctaviaHousekeepingImage: 10.0.13.80:8787/tripleotrain//openstack-octavia-housekeeping:current-tripleo
  ContainerOctaviaRsyslogImage: 10.0.13.80:8787/tripleotrain//openstack-rsyslog:current-tripleo
  ContainerOctaviaWorkerImage: 10.0.13.80:8787/tripleotrain//openstack-octavia-worker:current-tripleo
  ContainerOpenvswitchImage: 10.0.13.80:8787/tripleotrain//openstack-neutron-openvswitch-agent:current-tripleo
  ContainerOvnControllerConfigImage: 10.0.13.80:8787/tripleotrain//openstack-ovn-controller:current-tripleo
  ContainerOvnControllerImage: 10.0.13.80:8787/tripleotrain//openstack-ovn-controller:current-tripleo
  ContainerOvnDbsConfigImage: 10.0.13.80:8787/tripleotrain//openstack-ovn-northd:current-tripleo
  ContainerOvnDbsImage: 10.0.13.80:8787/tripleotrain//openstack-ovn-northd:current-tripleo
  ContainerOvnMetadataImage: 10.0.13.80:8787/tripleotrain//openstack-neutron-metadata-agent-ovn:current-tripleo
  ContainerOvnNbDbImage: 10.0.13.80:8787/tripleotrain//openstack-ovn-nb-db-server:current-tripleo
  ContainerOvnNorthdImage: 10.0.13.80:8787/tripleotrain//openstack-ovn-northd:current-tripleo
  ContainerOvnSbDbImage: 10.0.13.80:8787/tripleotrain//openstack-ovn-sb-db-server:current-tripleo
  ContainerPankoApiImage: 10.0.13.80:8787/tripleotrain//openstack-panko-api:current-tripleo
  ContainerPankoConfigImage: 10.0.13.80:8787/tripleotrain//openstack-panko-api:current-tripleo
  ContainerPlacementConfigImage: 10.0.13.80:8787/tripleotrain//openstack-placement-api:current-tripleo
  ContainerPlacementImage: 10.0.13.80:8787/tripleotrain//openstack-placement-api:current-tripleo
  ContainerQdrouterdConfigImage: 10.0.13.80:8787/tripleotrain//openstack-qdrouterd:current-tripleo
  ContainerQdrouterdImage: 10.0.13.80:8787/tripleotrain//openstack-qdrouterd:current-tripleo
  ContainerRabbitmqConfigImage: 10.0.13.80:8787/tripleotrain//openstack-rabbitmq:current-tripleo
  ContainerRabbitmqImage: 10.0.13.80:8787/tripleotrain//openstack-rabbitmq:current-tripleo
  ContainerRedisConfigImage: 10.0.13.80:8787/tripleotrain//openstack-redis:current-tripleo
  ContainerRedisImage: 10.0.13.80:8787/tripleotrain//openstack-redis:current-tripleo
  ContainerRsyslogConfigImage: 10.0.13.80:8787/tripleotrain//openstack-rsyslog:current-tripleo
  ContainerRsyslogImage: 10.0.13.80:8787/tripleotrain//openstack-rsyslog:current-tripleo
  ContainerRsyslogSidecarConfigImage: 10.0.13.80:8787/tripleotrain//openstack-rsyslog:current-tripleo
  ContainerRsyslogSidecarImage: 10.0.13.80:8787/tripleotrain//openstack-rsyslog:current-tripleo
  ContainerSaharaApiImage: 10.0.13.80:8787/tripleotrain//openstack-sahara-api:current-tripleo
  ContainerSaharaConfigImage: 10.0.13.80:8787/tripleotrain//openstack-sahara-api:current-tripleo
  ContainerSaharaEngineImage: 10.0.13.80:8787/tripleotrain//openstack-sahara-engine:current-tripleo
  ContainerSkydiveAgentImage: 10.0.13.80:8787/tripleotrain//openstack-skydive-agent:current-tripleo
  ContainerSkydiveAnalyzerImage: 10.0.13.80:8787/tripleotrain//openstack-skydive-analyzer:current-tripleo
  ContainerSwiftAccountImage: 10.0.13.80:8787/tripleotrain//openstack-swift-account:current-tripleo
  ContainerSwiftConfigImage: 10.0.13.80:8787/tripleotrain//openstack-swift-proxy-server:current-tripleo
  ContainerSwiftContainerImage: 10.0.13.80:8787/tripleotrain//openstack-swift-container:current-tripleo
  ContainerSwiftObjectImage: 10.0.13.80:8787/tripleotrain//openstack-swift-object:current-tripleo
  ContainerSwiftProxyImage: 10.0.13.80:8787/tripleotrain//openstack-swift-proxy-server:current-tripleo
  ContainerZaqarConfigImage: 10.0.13.80:8787/tripleotrain//openstack-zaqar-wsgi:current-tripleo
  ContainerZaqarImage: 10.0.13.80:8787/tripleotrain//openstack-zaqar-wsgi:current-tripleo
  DockerInsecureRegistryAddress:
  - 10.0.13.80:8787
  GrafanaContainerImage: 10.0.13.80:8787/grafana/grafana:5.2.4
  NodeExporterContainerImage: 10.0.13.80:8787/prom/node-exporter:v0.17.0
  PrometheusContainerImage: 10.0.13.80:8787/prom/prometheus:v2.7.2
```

Nội dung file `local_registry_images.yaml`

```sh
container_images:
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-aodh-api:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-aodh-evaluator:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-aodh-listener:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-aodh-notifier:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-barbican-api:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-barbican-keystone-listener:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-barbican-worker:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-ceilometer-central:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-ceilometer-compute:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-ceilometer-notification:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-cinder-api:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-cinder-backup:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-cinder-scheduler:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-cinder-volume:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-collectd:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-cron:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-designate-api:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-designate-backend-bind9:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-designate-central:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-designate-mdns:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-designate-producer:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-designate-sink:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-designate-worker:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-ec2-api:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-etcd:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-glance-api:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-gnocchi-api:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-gnocchi-metricd:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-gnocchi-statsd:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-haproxy:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-heat-api-cfn:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-heat-api:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-heat-engine:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-horizon:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-ironic-api:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-ironic-conductor:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-ironic-inspector:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-ironic-pxe:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-ironic-neutron-agent:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-iscsid:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-keepalived:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-keystone:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-manila-api:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-manila-scheduler:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-manila-share:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-mariadb:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-memcached:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-mistral-api:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-mistral-engine:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-mistral-executor:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-mistral-event-engine:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-multipathd:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-neutron-dhcp-agent:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-neutron-l3-agent:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-neutron-metadata-agent:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-neutron-openvswitch-agent:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-neutron-server-ovn:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-neutron-metadata-agent-ovn:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-nova-api:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-nova-compute-ironic:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-nova-compute:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-nova-conductor:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-nova-libvirt:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-nova-novncproxy:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-nova-scheduler:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-octavia-api:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-octavia-health-manager:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-octavia-housekeeping:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-octavia-worker:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-ovn-controller:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-ovn-nb-db-server:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-ovn-northd:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-ovn-sb-db-server:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-panko-api:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-placement-api:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-qdrouterd:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-rabbitmq:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-redis:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-sahara-api:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-sahara-engine:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-skydive-agent:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-skydive-analyzer:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-swift-account:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-swift-container:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-swift-object:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-swift-proxy-server:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-zaqar-wsgi:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: ceph
  imagename: docker.io/ceph/daemon:v4.0.12-stable-4.0-nautilus-centos-7-x86_64
  push_destination: 10.0.13.80:8787
- image_source: prom
  imagename: docker.io/prom/prometheus:v2.7.2
  push_destination: 10.0.13.80:8787
- image_source: prom
  imagename: docker.io/prom/alertmanager:v0.16.2
  push_destination: 10.0.13.80:8787
- image_source: prom
  imagename: docker.io/prom/node-exporter:v0.17.0
  push_destination: 10.0.13.80:8787
- image_source: grafana
  imagename: docker.io/grafana/grafana:5.2.4
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain//openstack-rsyslog:current-tripleo
  push_destination: 10.0.13.80:8787
```

### 3.5. Tạo file `05-network-environment.yaml`

```sh

resource_registry:
  OS::TripleO::Controller::Net::SoftwareConfig: /home/stack/templates/custom-nics/controller.yaml
  OS::TripleO::Compute::Net::SoftwareConfig: /home/stack/templates/custom-nics/compute.yaml


parameter_defaults:
  EC2MetadataIp: 10.0.13.80
  ControlPlaneDefaultRoute: 10.0.13.80

  StorageNetCidr: '10.0.15.0/24'
  StorageAllocationPools: [{'start': '10.0.15.131', 'end': '10.0.15.199'}]
  StorageNetworkVlanID: 15

  InternalApiNetCidr: '10.0.11.0/24'
  InternalApiAllocationPools: [{'start': '10.0.15.131', 'end': '10.0.11.199'}]
  InternalApiNetworkVlanID: 11

  TenantNetCidr: '10.0.12.0/24'
  TenantAllocationPools: [{'start': '10.0.12.131', 'end': '10.0.12.199'}]
  TenantNetworkVlanID: 12
  TenantNetPhysnetMtu: 1500 

  ExternalNetCidr: '10.0.17.0/24'
  ExternalAllocationPools: [{'start': '10.0.17.131', 'end': '10.0.17.199'}]
  ExternalInterfaceDefaultRoute: '10.0.17.1'
  ExternalNetworkVlanID: 17

  ControlFixedIPs: [{'ip_address':'10.0.13.130'}]
  InternalApiVirtualFixedIPs: [{'ip_address':'10.0.11.130'}]
  PublicVirtualFixedIPs: [{'ip_address':'10.0.17.130'}]
  StorageVirtualFixedIPs: [{'ip_address':'10.0.15.130'}]

  DnsServers: []
  NeutronNetworkType: 'geneve,vlan'
  NeutronNetworkVLANRanges: 'datacentre:1:1000'
  NtpServer: 10.0.13.1

  # The OVS logical->physical bridge mappings to use.
  NeutronBridgeMappings: 'provider:br-provider'
  # The Neutron ML2 and OpenVSwitch vlan mapping range to support.
  NeutronNetworkVLANRanges: 'provider:2:4094'  
  # Enable isolated metadata agent on controllers
  NeutronEnableIsolatedMetadata: 'True'
  ```


Copy file config NIC vào thư mục chứa file câu hình của Controller và Compute : 
```sh
cp /usr/share/openstack-tripleo-heat-templates/network/scripts/run-os-net-config.sh /home/stack/templates/custom-nics/
```

Tại thư mục custom-nics chứa 2 file cấu hình của Controller và Compute

```sh
/home/stack/custom-templates/custom-nics/controller.yaml
/home/stack/custom-templates/custom-nics/compute.yaml
```

Tạo file cấu hình `/home/stack/templates/custom-nics/controller.yaml`

```sh

heat_template_version: rocky
description: >
  Software Config to drive os-net-config to configure multiple interfaces for the Controller role.
parameters:
  ControlPlaneIp:
    default: ''
    description: IP address/subnet on the ctlplane network
    type: string
  ControlPlaneSubnetCidr:
    default: ''
    type: string
  ControlPlaneDefaultRoute:
    default: ''
    type: string
  ControlPlaneStaticRoutes:
    default: []
    type: json
  ControlPlaneMtu:
    default: 1500
    type: number

  StorageIpSubnet:
    default: ''
    description: IP address/subnet on the storage network
    type: string
  StorageNetworkVlanID:
    default: 15
    description: Vlan ID for the storage network traffic.
    type: number
  StorageMtu:
    default: 1500
    type: number
  StorageInterfaceRoutes:
    default: []
    type: json

  InternalApiIpSubnet:
    default: ''
    description: IP address/subnet on the internal_api network
    type: string
  InternalApiNetworkVlanID:
    default: 11
    description: Vlan ID for the internal_api network traffic.
    type: number
  InternalApiMtu:
    default: 1500
    type: number
  InternalApiInterfaceRoutes:
    default: []
    type: json

  TenantIpSubnet:
    default: ''
    description: IP address/subnet on the tenant network
    type: string
  TenantNetworkVlanID:
    default: 12
    description: Vlan ID for the tenant network traffic.
    type: number
  TenantMtu:
    default: 1500
    type: number
  TenantInterfaceRoutes:
    default: []
    type: json

  ExternalIpSubnet:
    default: ''
    description: IP address/subnet on the external network
    type: string
  ExternalNetworkVlanID:
    default: 17
    description: Vlan ID for the external network traffic.
    type: number
  ExternalMtu:
    default: 1500
    type: number
  ExternalInterfaceDefaultRoute:
    default: ''
    description: default route for the external network
    type: string
  ExternalInterfaceRoutes:
    default: []
    type: json

  DnsServers: # Override this via parameter_defaults
    default: []
    type: comma_delimited_list
  DnsSearchDomains: # Override this via parameter_defaults
    default: []
    description: A list of DNS search domains to be added (in order) to resolv.conf.
    type: comma_delimited_list
resources:
  OsNetConfigImpl:
    type: OS::Heat::SoftwareConfig
    properties:
      group: script
      config:
        str_replace:
          template:
            get_file: /home/stack/templates/custom-nics/run-os-net-config.sh
          params:
            $network_config:
              network_config:
              - type: interface
                name: ens192
                mtu:
                  get_param: ControlPlaneMtu
                use_dhcp: false
                dns_servers:
                  get_param: DnsServers
                domain:
                  get_param: DnsSearchDomains
                addresses:
                - ip_netmask:
                    list_join:
                    - /
                    - - get_param: ControlPlaneIp
                      - get_param: ControlPlaneSubnetCidr
                routes:
                  list_concat_unique:
                - ip_netmask: 169.254.169.254/32
                  next_hop: {get_param: EC2MetadataIp}                               
              - type: interface
                name: ens161
                mtu:
                  get_param: StorageMtu
                addresses:
                - ip_netmask:
                    get_param: StorageIpSubnet
                routes:
                  list_concat_unique:
                    - get_param: StorageInterfaceRoutes
              - type: interface
                name: ens256
                mtu:
                  get_param: InternalApiMtu
                addresses:
                - ip_netmask:
                    get_param: InternalApiIpSubnet
                routes:
                  list_concat_unique:
                    - get_param: InternalApiInterfaceRoutes
              - type: ovs_bridge
                name: br-tenant
                mtu:
                  get_param: TenantMtu
                dns_servers:
                  get_param: DnsServers
                addresses:
                - ip_netmask:
                    get_param: TenantIpSubnet
                routes:
                  list_concat_unique:
                    - get_param: TenantInterfaceRoutes
                members:
                - type: interface
                  name: ens224
                  mtu:
                    get_param: TenantMtu
                  primary: true
              - type: ovs_bridge
                name: br-ext
                mtu:
                  get_param: ExternalMtu
                dns_servers:
                  get_param: DnsServers
                use_dhcp: false
                addresses:
                - ip_netmask:
                    get_param: ExternalIpSubnet
                routes:
                  list_concat_unique:
                    - get_param: ExternalInterfaceRoutes
                    - - default: true
                        next_hop:
                          get_param: ExternalInterfaceDefaultRoute
                members:
                - type: interface
                  name: ens193
                  mtu:
                    get_param: ExternalMtu
                  primary: true
outputs:
  OS::stack_id:
    description: The OsNetConfigImpl resource.
    value:
      get_resource: OsNetConfigImpl
```

Bước 5 : Tạo file `/home/stack/templates/custom-nics/compute.yaml`

```sh

heat_template_version: rocky
description: >
  Software Config to drive os-net-config to configure multiple interfaces for the Controller role.
parameters:
  ControlPlaneIp:
    default: ''
    description: IP address/subnet on the ctlplane network
    type: string
  ControlPlaneSubnetCidr:
    default: ''
    type: string
  ControlPlaneDefaultRoute:
    default: ''
    type: string
  ControlPlaneStaticRoutes:
    default: []
    type: json
  ControlPlaneMtu:
    default: 1500
    type: number

  StorageIpSubnet:
    default: ''
    description: IP address/subnet on the storage network
    type: string
  StorageNetworkVlanID:
    default: 15
    description: Vlan ID for the storage network traffic.
    type: number
  StorageMtu:
    default: 1500
    type: number
  StorageInterfaceRoutes:
    default: []
    type: json
  InternalApiIpSubnet:
    default: ''
    description: IP address/subnet on the internal_api network
    type: string
  InternalApiNetworkVlanID:
    default: 11
    description: Vlan ID for the internal_api network traffic.
    type: number
  InternalApiMtu:
    default: 1500
    type: number
  InternalApiInterfaceRoutes:
    default: []
    type: json
  TenantIpSubnet:
    default: ''
    description: IP address/subnet on the tenant network
    type: string
  TenantNetworkVlanID:
    default: 12
    description: Vlan ID for the tenant network traffic.
    type: number
  TenantMtu:
    default: 1500
    type: number
  TenantInterfaceRoutes:
    default: []
    type: json

  ExternalMtu:
    default: 1500
    type: number

  DnsServers: # Override this via parameter_defaults
    default: []
    type: comma_delimited_list
  DnsSearchDomains: # Override this via parameter_defaults
    default: []
    type: comma_delimited_list
resources:
  OsNetConfigImpl:
    type: OS::Heat::SoftwareConfig
    properties:
      group: script
      config:
        str_replace:
          template:
            get_file: /home/stack/templates/custom-nics/run-os-net-config.sh 
          params:
            $network_config:
              network_config:
              - type: interface
                name: ens192
                mtu: 1500
                use_dhcp: false
                dns_servers:
                  get_param: DnsServers
                domain:
                  get_param: DnsSearchDomains
                addresses:
                - ip_netmask:
                    list_join:
                    - /
                    - - get_param: ControlPlaneIp
                      - get_param: ControlPlaneSubnetCidr
                routes:
                  list_concat_unique:
                    - get_param: ControlPlaneStaticRoutes
                    - - default: true
                - ip_netmask: 169.254.169.254/32
                  next_hop: {get_param: EC2MetadataIp}
              - type: interface
                name: ens161
                mtu:
                  get_param: StorageMtu
                addresses:
                - ip_netmask:
                    get_param: StorageIpSubnet
                routes:
                  list_concat_unique:
                    - get_param: StorageInterfaceRoutes
              - type: interface
                name: ens256
                mtu: 1500
                addresses:
                - ip_netmask:
                    get_param: InternalApiIpSubnet
                routes:
                  list_concat_unique:
                    - get_param: InternalApiInterfaceRoutes
              - type: ovs_bridge
                name: br-tenant
                mtu: 1500
                dns_servers:
                  get_param: DnsServers
                addresses:
                - ip_netmask:
                    get_param: TenantIpSubnet
                routes:
                  list_concat_unique:
                    - get_param: TenantInterfaceRoutes
                members:
                - type: interface
                  name: ens224
                  mtu: 1500
                  primary: true           
              - type: ovs_bridge
                name: br-provider
                mtu: 1500
                members:
                - type: interface
                  name: ens193
                  mtu: 1500
                  primary: true                          
outputs:
  OS::stack_id:
    description: The OsNetConfigImpl resource.
    value:
      get_resource: OsNetConfigImpl
```

Tạo file `/home/stack/custom-templates/06-extra-parameters.yaml`

```sh
parameter_defaults:
  #CeilometerMeterDispatcher: 'database' # does not exist in OSP13
  MysqlMaxConnections: 8944
  SwiftWorkers: 4
  RabbitFDLimit: 65536
  TimeZone: 'Asia/Ho_Chi_Minh'

  ExtraConfig:
    # Increase Qemu max_files param further to 131072
    # Default in base template is 32768
    nova::compute::libvirt::qemu::max_files: 131072

    # CPU/Memory Overcommitment Ratio
    nova::cpu_allocation_ratio: 8.0
    nova::block_device_allocate_retries: 600

  ControllerExtraConfig:
    # SSL Hardening
    # The cipher suite to use in HAProxy
    tripleo::haproxy::ssl_cipher_suite: ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA:ECDHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA256:DHE-RSA-AES256-SHA:ECDHE-ECDSA-DES-CBC3-SHA:ECDHE-RSA-DES-CBC3-SHA:EDH-RSA-DES-CBC3-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:DES-CBC3-SHA:!DSS

    # The SSL/TLS rules to use in HAProxy
    tripleo::haproxy::ssl_options: no-sslv3 no-tls-tickets

    # Expire Ceilometer data after 14 days
    # Default is no-limit
    ceilometer::event_time_to_live: 1209600
    ceilometer::metering_time_to_live: 1209600

    # HAProxy maxconn for MariaDB/Galera
    # Default is 4096
    tripleo::haproxy::haproxy_default_maxconn: 8944
```



Gắn cụ thể các IP cố định tới các node `/home/stack/custom-templates/07-predictive-ips.yaml`

```sh
resource_registry:
  OS::TripleO::Controller::Ports::ExternalPort: /usr/share/openstack-tripleo-heat-templates/network/ports/external_from_pool.yaml
  OS::TripleO::Controller::Ports::InternalApiPort: /usr/share/openstack-tripleo-heat-templates/network/ports/internal_api_from_pool.yaml
  OS::TripleO::Controller::Ports::StoragePort: /usr/share/openstack-tripleo-heat-templates/network/ports/storage_from_pool.yaml
  OS::TripleO::Controller::Ports::TenantPort: /usr/share/openstack-tripleo-heat-templates/network/ports/tenant_from_pool.yaml

  OS::TripleO::Compute::Ports::InternalApiPort: /usr/share/openstack-tripleo-heat-templates/network/ports/internal_api_from_pool.yaml
  OS::TripleO::Compute::Ports::StoragePort: /usr/share/openstack-tripleo-heat-templates/network/ports/storage_from_pool.yaml
  OS::TripleO::Compute::Ports::TenantPort: /usr/share/openstack-tripleo-heat-templates/network/ports/tenant_from_pool.yaml

parameter_defaults:
  ControllerIPs:
    ctlplane:
    - 10.0.13.131
    - 10.0.13.132
    - 10.0.13.133
    internal_api:
    - 10.0.11.131
    - 10.0.11.132
    - 10.0.11.133
    tenant:
    - 10.0.12.131
    - 10.0.12.132
    - 10.0.12.133
    storage:
    - 10.0.15.131
    - 10.0.15.132
    - 10.0.15.133
    external:
    - 10.0.17.131
    - 10.0.17.132
    - 10.0.17.133
  ComputeIPs:
    ctlplane:
    - 10.0.13.134
    - 10.0.13.135
    - 10.0.13.136
    internal_api:
    - 10.0.11.134
    - 10.0.11.135
    - 10.0.11.136
    tenant:
    - 10.0.12.134
    - 10.0.12.135
    - 10.0.12.136
    storage:
    - 10.0.15.134
    - 10.0.15.135
    - 10.0.15.136

```

Đặt file map hostname của các node `/home/stack/custom-templates/08-scheduler-hints-env.yaml`

```sh
parameter_defaults:
  ControllerSchedulerHints:
    'capabilities:node': 'controller-%index%'
  ComputeSchedulerHints:
    'capabilities:node': 'compute-%index%'
  HostnameMap:
    overcloud-controller-0: overcloud-controller-test-0
    overcloud-controller-1: overcloud-controller-test-1
    overcloud-controller-2: overcloud-controller-test-2
    overcloud-compute-0: overcloud-compute-test-0
    overcloud-compute-1: overcloud-compute-test-1
    overcloud-compute-2: overcloud-compute-test-2
```

Tạo file `/home/stack/custom-templates/09-no-tls-endpoint-public.yaml`

```sh
# *******************************************************************
# This file was created automatically by the sample environment
# generator. Developers should use `tox -e genconfig` to update it.
# Users are recommended to make changes to a copy of the file instead
# of the original, if any customizations are needed.
# *******************************************************************
# title: Deploy All Endpoints without TLS and with IP addresses
# description: |
#   Use this environment when deploying an overcloud where all the endpoints not
#   using TLS and are using IP addresses.
parameter_defaults:
  # Whether to enable TLS on the public interface or not.
  # Type: boolean
  EnablePublicTLS: False

  # Mapping of service endpoint -> protocol. Typically set via parameter_defaults in the resource registry.
  # Type: json
  EndpointMap:
    AodhAdmin: {protocol: http, port: '8042', host: IP_ADDRESS}
    AodhInternal: {protocol: http, port: '8042', host: IP_ADDRESS}
    AodhPublic: {protocol: http, port: '8042', host: IP_ADDRESS}
    BarbicanAdmin: {protocol: http, port: '9311', host: IP_ADDRESS}
    BarbicanInternal: {protocol: http, port: '9311', host: IP_ADDRESS}
    BarbicanPublic: {protocol: http, port: '9311', host: IP_ADDRESS}
    CephDashboardInternal: {protocol: http, port: '8444', host: IP_ADDRESS}
    CephGrafanaInternal: {protocol: http, port: '3100', host: IP_ADDRESS}
    CephRgwAdmin: {protocol: http, port: '8080', host: IP_ADDRESS}
    CephRgwInternal: {protocol: http, port: '8080', host: IP_ADDRESS}
    CephRgwPublic: {protocol: http, port: '8080', host: IP_ADDRESS}
    CinderAdmin: {protocol: http, port: '8776', host: IP_ADDRESS}
    CinderInternal: {protocol: http, port: '8776', host: IP_ADDRESS}
    CinderPublic: {protocol: http, port: '8776', host: IP_ADDRESS}
    DesignateAdmin: {protocol: 'http', port: '9001', host: IP_ADDRESS}
    DesignateInternal: {protocol: 'http', port: '9001', host: IP_ADDRESS}
    DesignatePublic: {protocol: 'http', port: '9001', host: IP_ADDRESS}
    DockerRegistryInternal: {protocol: http, port: '8787', host: IP_ADDRESS}
    Ec2ApiAdmin: {protocol: http, port: '8788', host: IP_ADDRESS}
    Ec2ApiInternal: {protocol: http, port: '8788', host: IP_ADDRESS}
    Ec2ApiPublic: {protocol: http, port: '8788', host: IP_ADDRESS}
    GaneshaInternal: {protocol: nfs, port: '2049', host: IP_ADDRESS}
    GlanceAdmin: {protocol: http, port: '9292', host: IP_ADDRESS}
    GlanceInternal: {protocol: http, port: '9292', host: IP_ADDRESS}
    GlancePublic: {protocol: http, port: '9292', host: IP_ADDRESS}
    GnocchiAdmin: {protocol: http, port: '8041', host: IP_ADDRESS}
    GnocchiInternal: {protocol: http, port: '8041', host: IP_ADDRESS}
    GnocchiPublic: {protocol: http, port: '8041', host: IP_ADDRESS}
    HeatAdmin: {protocol: http, port: '8004', host: IP_ADDRESS}
    HeatInternal: {protocol: http, port: '8004', host: IP_ADDRESS}
    HeatPublic: {protocol: http, port: '8004', host: IP_ADDRESS}
    HeatUIConfig: {protocol: http, port: '3000', host: IP_ADDRESS}
    HeatCfnAdmin: {protocol: http, port: '8000', host: IP_ADDRESS}
    HeatCfnInternal: {protocol: http, port: '8000', host: IP_ADDRESS}
    HeatCfnPublic: {protocol: http, port: '8000', host: IP_ADDRESS}
    HorizonPublic: {protocol: http, port: '80', host: IP_ADDRESS}
    IronicAdmin: {protocol: http, port: '6385', host: IP_ADDRESS}
    IronicInternal: {protocol: http, port: '6385', host: IP_ADDRESS}
    IronicPublic: {protocol: http, port: '6385', host: IP_ADDRESS}
    IronicUIConfig: {protocol: http, port: '3000', host: IP_ADDRESS}
    IronicInspectorAdmin: {protocol: http, port: '5050', host: IP_ADDRESS}
    IronicInspectorInternal: {protocol: http, port: '5050', host: IP_ADDRESS}
    IronicInspectorPublic: {protocol: http, port: '5050', host: IP_ADDRESS}
    IronicInspectorUIConfig: {protocol: http, port: '3000', host: IP_ADDRESS}
    KeystoneAdmin: {protocol: http, port: '35357', host: IP_ADDRESS}
    KeystoneInternal: {protocol: http, port: '5000', host: IP_ADDRESS}
    KeystonePublic: {protocol: http, port: '5000', host: IP_ADDRESS}
    KeystoneUIConfig: {protocol: http, port: '3000', host: IP_ADDRESS}
    ManilaAdmin: {protocol: http, port: '8786', host: IP_ADDRESS}
    ManilaInternal: {protocol: http, port: '8786', host: IP_ADDRESS}
    ManilaPublic: {protocol: http, port: '8786', host: IP_ADDRESS}
    MetricsQdrPublic: {protocol: 'amqp', port: '5666', host: IP_ADDRESS}
    MistralAdmin: {protocol: http, port: '8989', host: IP_ADDRESS}
    MistralInternal: {protocol: http, port: '8989', host: IP_ADDRESS}
    MistralPublic: {protocol: http, port: '8989', host: IP_ADDRESS}
    MistralUIConfig: {protocol: http, port: '3000', host: IP_ADDRESS}
    MysqlInternal: {protocol: mysql+pymysql, port: '3306', host: IP_ADDRESS}
    NeutronAdmin: {protocol: http, port: '9696', host: IP_ADDRESS}
    NeutronInternal: {protocol: http, port: '9696', host: IP_ADDRESS}
    NeutronPublic: {protocol: http, port: '9696', host: IP_ADDRESS}
    NovaAdmin: {protocol: http, port: '8774', host: IP_ADDRESS}
    NovaInternal: {protocol: http, port: '8774', host: IP_ADDRESS}
    NovaPublic: {protocol: http, port: '8774', host: IP_ADDRESS}
    NovajoinAdmin: {protocol: http, port: '9090', host: IP_ADDRESS}
    NovajoinInternal: {protocol: http, port: '9090', host: IP_ADDRESS}
    NovajoinPublic: {protocol: http, port: '9090', host: IP_ADDRESS}
    NovaMetadataInternal: {protocol: http, port: '8775', host: IP_ADDRESS}
    NovaUIConfig: {protocol: http, port: '3000', host: IP_ADDRESS}
    PlacementAdmin: {protocol: http, port: '8778', host: IP_ADDRESS}
    PlacementInternal: {protocol: http, port: '8778', host: IP_ADDRESS}
    PlacementPublic: {protocol: http, port: '8778', host: IP_ADDRESS}
    NovaVNCProxyAdmin: {protocol: http, port: '6080', host: IP_ADDRESS}
    NovaVNCProxyInternal: {protocol: http, port: '6080', host: IP_ADDRESS}
    NovaVNCProxyPublic: {protocol: http, port: '6080', host: IP_ADDRESS}
    OctaviaAdmin: {protocol: http, port: '9876', host: IP_ADDRESS}
    OctaviaInternal: {protocol: http, port: '9876', host: IP_ADDRESS}
    OctaviaPublic: {protocol: http, port: '9876', host: IP_ADDRESS}
    PankoAdmin: {protocol: http, port: '8977', host: IP_ADDRESS}
    PankoInternal: {protocol: http, port: '8977', host: IP_ADDRESS}
    PankoPublic: {protocol: http, port: '8977', host: IP_ADDRESS}
    SaharaAdmin: {protocol: http, port: '8386', host: IP_ADDRESS}
    SaharaInternal: {protocol: http, port: '8386', host: IP_ADDRESS}
    SaharaPublic: {protocol: http, port: '8386', host: IP_ADDRESS}
    SwiftAdmin: {protocol: http, port: '8080', host: IP_ADDRESS}
    SwiftInternal: {protocol: http, port: '8080', host: IP_ADDRESS}
    SwiftPublic: {protocol: http, port: '8080', host: IP_ADDRESS}
    SwiftUIConfig: {protocol: http, port: '3000', host: IP_ADDRESS}
    ZaqarAdmin: {protocol: http, port: '8888', host: IP_ADDRESS}
    ZaqarInternal: {protocol: http, port: '8888', host: IP_ADDRESS}
    ZaqarPublic: {protocol: http, port: '8888', host: IP_ADDRESS}
    ZaqarWebSocketAdmin: {protocol: ws, port: '9000', host: IP_ADDRESS}
    ZaqarWebSocketInternal: {protocol: ws, port: '9000', host: IP_ADDRESS}
    ZaqarWebSocketPublic: {protocol: ws, port: '9000', host: IP_ADDRESS}
    ZaqarWebSocketUIConfig: {protocol: ws, port: '3000', host: IP_ADDRESS}
```

Tạo file `disable-panko.yaml`

```sh
# This heat environment can be used to disable panko services.
# Panko should not be used in most deployement, but we can't yet remove it from
# the default setup.

resource_registry:
  OS::TripleO::Services::PankoApi: OS::Heat::None

parameter_defaults:
  CeilometerEnablePanko: false
  EventPipelinePublishers:
    - gnocchi://?archive_policy=high
```

Tạo file `disable-telemetry.yaml`

```sh
# This heat environment can be used to disable all of the telemetry services.
# It is most useful in a resource constrained environment or one in which
# telemetry is not needed.

resource_registry:
  OS::TripleO::Services::CeilometerAgentCentral: OS::Heat::None
  OS::TripleO::Services::CeilometerAgentNotification: OS::Heat::None
  OS::TripleO::Services::CeilometerAgentIpmi: OS::Heat::None
  OS::TripleO::Services::ComputeCeilometerAgent: OS::Heat::None
  OS::TripleO::Services::GnocchiApi: OS::Heat::None
  OS::TripleO::Services::GnocchiMetricd: OS::Heat::None
  OS::TripleO::Services::GnocchiStatsd: OS::Heat::None
  OS::TripleO::Services::AodhApi: OS::Heat::None
  OS::TripleO::Services::AodhEvaluator: OS::Heat::None
  OS::TripleO::Services::AodhNotifier: OS::Heat::None
  OS::TripleO::Services::AodhListener: OS::Heat::None
  OS::TripleO::Services::PankoApi: OS::Heat::None

parameter_defaults:
  NotificationDriver: 'noop'
  GnocchiRbdPoolName: ''
```


Câu lệnh boot : 

	openstack --debug overcloud deploy --templates \
	-r /home/stack/overcloud-template/00-roles-data.yaml \
	-e /home/stack/overcloud-template/01-node-info.yaml \
	-e /home/stack/overcloud-template/02-network-isolation.yaml \

	--log-file overcloud_deployment.log --ntp-server 10.0.13.1
	
	--update-plan-only