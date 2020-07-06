# Cài đặt TripleO trên Centos 7

Triển khai Triple O với mô hình đơn giản bao gồm : 
 - 01 node Director
 - 03 node Controller
 - 03 node Compute
 - 01 node Storage (External, cài riêng)

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
enabled_hardware_types=idrac,ilo,ipmi,redfish,manual-management
undercloud_ntp_servers = 10.0.13.1
local_ip = 10.0.13.80/24
undercloud_admin_host = 10.0.13.120
undercloud_public_host = 10.0.13.121
generate_service_certificate = true
certificate_generation_ca = local
[ctlplane-subnet]
cidr = 10.0.13.0/24
dhcp_end = 10.0.13.106
dhcp_start = 10.0.13.81
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

Trên Undercloud node, tạo một file JSON với tên `~/nodes.json`

```sh
vi nodes.json
{
      "nodes": [
        {
          "name": "controller01",
          "memory": "16384",
          "mac": [
            "00:50:56:aa:70:cf"
          ],
          "pm_type": "manual-management",
          "disk": "100",
          "arch": "x86_64",
          "cpu": "16"
        },
        {
          "name": "controller02",
          "memory": "16384",
          "mac": [
            "00:50:56:aa:29:d6"
          ],
          "pm_type": "manual-management",
          "disk": "100",
          "arch": "x86_64",
          "cpu": "16"
        },
        {
          "name": "controller03",
          "memory": "16384",
          "mac": [
            "00:50:56:aa:e5:b4"
          ],
          "pm_type": "manual-management",
          "disk": "100",
          "arch": "x86_64",
          "cpu": "16"
        },
        {
          "name": "compute01",
          "memory": "24576",
          "mac": [
            "00:50:56:aa:bd:f0"
          ],
          "pm_type": "manual-management",
          "disk": "200",
          "arch": "x86_64",
          "cpu": "24"
        },
        {
          "name": "compute02",
          "memory": "24576",
          "mac": [
            "00:50:56:aa:86:a4"
          ],
          "pm_type": "manual-management",
          "disk": "200",
          "arch": "x86_64",
          "cpu": "24"
        },
        {
          "name": "compute03",
          "memory": "24576",
          "mac": [
            "00:50:56:aa:76:f4"
          ],
          "pm_type": "manual-management",
          "disk": "200",
          "arch": "x86_64",
          "cpu": "24"
         }
      ]
}
```

Thay thế MAC với MAC của vlan 13 (dải control plane) với các node Controller và Compute ở trên.

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

Sau khi trạng thái các node là `power off` và `available`. Ta thực hiện shutdown các máy bằng tay.

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

### 3.1. Template `01-roles-data.yaml`


Copy file template sang thư mục config

```sh
cp /usr/share/openstack-tripleo-heat-templates/roles_data.yaml /home/stack/custom-templates/01-roles-data.yaml
```

Chỉnh sửa giá trị `CountDefault` tại section Controller và Compute thành `3`. Loại bỏ các node không cần thiết

```sh
###############################################################################
# Role: Controller                                                            #
###############################################################################
- name: Controller
  CountDefault: 3
  tags:
    - primary
    - controller
  networks:
    External:
      subnet: external_subnet
    InternalApi:
      subnet: internal_api_subnet
    Storage:
      subnet: storage_subnet
    Tenant:
      subnet: tenant_subnet
  # For systems with both IPv4 and IPv6, you may specify a gateway network for
  # each, such as ['ControlPlane', 'External']
  default_route_networks: ['External']
  HostnameFormatDefault: '%stackname%-controller-%index%'
  # Deprecated & backward-compatible values (FIXME: Make parameters consistent)
  # Set uses_deprecated_params to True if any deprecated params are used.
  uses_deprecated_params: True
  deprecated_param_extraconfig: 'controllerExtraConfig'
  deprecated_param_flavor: 'OvercloudControlFlavor'
  deprecated_param_image: 'controllerImage'
  deprecated_nic_config_name: 'controller.yaml'
  update_serial: 1
  ServicesDefault:
    - OS::TripleO::Services::Aide
    - OS::TripleO::Services::AodhApi
    - OS::TripleO::Services::AodhEvaluator
    - OS::TripleO::Services::AodhListener
    - OS::TripleO::Services::AodhNotifier
    - OS::TripleO::Services::AuditD
    - OS::TripleO::Services::BarbicanApi
    - OS::TripleO::Services::BarbicanBackendSimpleCrypto
    - OS::TripleO::Services::BarbicanBackendDogtag
    - OS::TripleO::Services::BarbicanBackendKmip
    - OS::TripleO::Services::BarbicanBackendPkcs11Crypto
    - OS::TripleO::Services::BootParams
    - OS::TripleO::Services::CACerts
    - OS::TripleO::Services::CeilometerAgentCentral
    - OS::TripleO::Services::CeilometerAgentNotification
    - OS::TripleO::Services::CephExternal
    - OS::TripleO::Services::CephGrafana
    - OS::TripleO::Services::CephMds
    - OS::TripleO::Services::CephMgr
    - OS::TripleO::Services::CephMon
    - OS::TripleO::Services::CephRbdMirror
    - OS::TripleO::Services::CephRgw
    - OS::TripleO::Services::CertmongerUser
    - OS::TripleO::Services::CinderApi
    - OS::TripleO::Services::CinderBackendDellPs
    - OS::TripleO::Services::CinderBackendDellSc
    - OS::TripleO::Services::CinderBackendDellEMCPowermax
    - OS::TripleO::Services::CinderBackendDellEMCSc
    - OS::TripleO::Services::CinderBackendDellEMCUnity
    - OS::TripleO::Services::CinderBackendDellEMCVMAXISCSI
    - OS::TripleO::Services::CinderBackendDellEMCVNX
    - OS::TripleO::Services::CinderBackendDellEMCVxFlexOS
    - OS::TripleO::Services::CinderBackendDellEMCXtremio
    - OS::TripleO::Services::CinderBackendDellEMCXTREMIOISCSI
    - OS::TripleO::Services::CinderBackendNetApp
    - OS::TripleO::Services::CinderBackendPure
    - OS::TripleO::Services::CinderBackendScaleIO
    - OS::TripleO::Services::CinderBackendVRTSHyperScale
    - OS::TripleO::Services::CinderBackendNVMeOF
    - OS::TripleO::Services::CinderBackup
    - OS::TripleO::Services::CinderHPELeftHandISCSI
    - OS::TripleO::Services::CinderScheduler
    - OS::TripleO::Services::CinderVolume
    - OS::TripleO::Services::Clustercheck
    - OS::TripleO::Services::Collectd
    - OS::TripleO::Services::ContainerImagePrepare
    - OS::TripleO::Services::DesignateApi
    - OS::TripleO::Services::DesignateCentral
    - OS::TripleO::Services::DesignateProducer
    - OS::TripleO::Services::DesignateWorker
    - OS::TripleO::Services::DesignateMDNS
    - OS::TripleO::Services::DesignateSink
    - OS::TripleO::Services::Docker
    - OS::TripleO::Services::Ec2Api
    - OS::TripleO::Services::Etcd
    - OS::TripleO::Services::GlanceApi
    - OS::TripleO::Services::GnocchiApi
    - OS::TripleO::Services::GnocchiMetricd
    - OS::TripleO::Services::GnocchiStatsd
    - OS::TripleO::Services::HAproxy
    - OS::TripleO::Services::HeatApi
    - OS::TripleO::Services::HeatApiCloudwatch
    - OS::TripleO::Services::HeatApiCfn
    - OS::TripleO::Services::HeatEngine
    - OS::TripleO::Services::Horizon
    - OS::TripleO::Services::IpaClient
    - OS::TripleO::Services::Ipsec
    - OS::TripleO::Services::IronicApi
    - OS::TripleO::Services::IronicConductor
    - OS::TripleO::Services::IronicInspector
    - OS::TripleO::Services::IronicPxe
    - OS::TripleO::Services::IronicNeutronAgent
    - OS::TripleO::Services::Iscsid
    - OS::TripleO::Services::Keepalived
    - OS::TripleO::Services::Kernel
    - OS::TripleO::Services::Keystone
    - OS::TripleO::Services::LoginDefs
    - OS::TripleO::Services::ManilaApi
    - OS::TripleO::Services::ManilaBackendCephFs
    - OS::TripleO::Services::ManilaBackendIsilon
    - OS::TripleO::Services::ManilaBackendNetapp
    - OS::TripleO::Services::ManilaBackendUnity
    - OS::TripleO::Services::ManilaBackendVNX
    - OS::TripleO::Services::ManilaBackendVMAX
    - OS::TripleO::Services::ManilaScheduler
    - OS::TripleO::Services::ManilaShare
    - OS::TripleO::Services::Memcached
    - OS::TripleO::Services::MetricsQdr
    - OS::TripleO::Services::MistralApi
    - OS::TripleO::Services::MistralEngine
    - OS::TripleO::Services::MistralExecutor
    - OS::TripleO::Services::MistralEventEngine
    - OS::TripleO::Services::Multipathd
    - OS::TripleO::Services::MySQL
    - OS::TripleO::Services::MySQLClient
    - OS::TripleO::Services::NeutronApi
    - OS::TripleO::Services::NeutronBgpVpnApi
    - OS::TripleO::Services::NeutronSfcApi
    - OS::TripleO::Services::NeutronCorePlugin
    - OS::TripleO::Services::NeutronDhcpAgent
    - OS::TripleO::Services::NeutronL2gwAgent
    - OS::TripleO::Services::NeutronL2gwApi
    - OS::TripleO::Services::NeutronL3Agent
    - OS::TripleO::Services::NeutronLinuxbridgeAgent
    - OS::TripleO::Services::NeutronMetadataAgent
    - OS::TripleO::Services::NeutronML2FujitsuCfab
    - OS::TripleO::Services::NeutronML2FujitsuFossw
    - OS::TripleO::Services::NeutronOvsAgent
    - OS::TripleO::Services::NeutronVppAgent
    - OS::TripleO::Services::NeutronAgentsIBConfig
    - OS::TripleO::Services::NovaApi
    - OS::TripleO::Services::NovaConductor
    - OS::TripleO::Services::NovaIronic
    - OS::TripleO::Services::NovaMetadata
    - OS::TripleO::Services::NovaScheduler
    - OS::TripleO::Services::NovaVncProxy
    - OS::TripleO::Services::ContainersLogrotateCrond
    - OS::TripleO::Services::OctaviaApi
    - OS::TripleO::Services::OctaviaDeploymentConfig
    - OS::TripleO::Services::OctaviaHealthManager
    - OS::TripleO::Services::OctaviaHousekeeping
    - OS::TripleO::Services::OctaviaWorker
    - OS::TripleO::Services::OpenStackClients
    - OS::TripleO::Services::OVNDBs
    - OS::TripleO::Services::OVNController
    - OS::TripleO::Services::Pacemaker
    - OS::TripleO::Services::PankoApi
    - OS::TripleO::Services::PlacementApi
    - OS::TripleO::Services::OsloMessagingRpc
    - OS::TripleO::Services::OsloMessagingNotify
    - OS::TripleO::Services::Podman
    - OS::TripleO::Services::Rear
    - OS::TripleO::Services::Redis
    - OS::TripleO::Services::Rhsm
    - OS::TripleO::Services::Rsyslog
    - OS::TripleO::Services::RsyslogSidecar
    - OS::TripleO::Services::SaharaApi
    - OS::TripleO::Services::SaharaEngine
    - OS::TripleO::Services::Securetty
    - OS::TripleO::Services::SkydiveAgent
    - OS::TripleO::Services::SkydiveAnalyzer
    - OS::TripleO::Services::Snmp
    - OS::TripleO::Services::Sshd
    - OS::TripleO::Services::Timesync
    - OS::TripleO::Services::Timezone
    - OS::TripleO::Services::TripleoFirewall
    - OS::TripleO::Services::TripleoPackages
    - OS::TripleO::Services::Tuned
    - OS::TripleO::Services::Vpp
    - OS::TripleO::Services::Zaqar
###############################################################################
# Role: Compute                                                               #
###############################################################################
- name: Compute
  description: |
    Basic Compute Node role
  CountDefault: 3
  # Create external Neutron bridge (unset if using ML2/OVS without DVR)
  tags:
    - external_bridge
  networks:
    InternalApi:
      subnet: internal_api_subnet
    Tenant:
      subnet: tenant_subnet
    Storage:
      subnet: storage_subnet
  HostnameFormatDefault: '%stackname%-novacompute-%index%'
  RoleParametersDefault:
    TunedProfileName: "virtual-host"
  # Deprecated & backward-compatible values (FIXME: Make parameters consistent)
  # Set uses_deprecated_params to True if any deprecated params are used.
  # These deprecated_params only need to be used for existing roles and not for
  # composable roles.
  uses_deprecated_params: True
  deprecated_param_image: 'NovaImage'
  deprecated_param_extraconfig: 'NovaComputeExtraConfig'
  deprecated_param_metadata: 'NovaComputeServerMetadata'
  deprecated_param_scheduler_hints: 'NovaComputeSchedulerHints'
  deprecated_param_ips: 'NovaComputeIPs'
  deprecated_server_resource_name: 'NovaCompute'
  deprecated_nic_config_name: 'compute.yaml'
  update_serial: 25
  ServicesDefault:
    - OS::TripleO::Services::Aide
    - OS::TripleO::Services::AuditD
    - OS::TripleO::Services::BootParams
    - OS::TripleO::Services::CACerts
    - OS::TripleO::Services::CephClient
    - OS::TripleO::Services::CephExternal
    - OS::TripleO::Services::CertmongerUser
    - OS::TripleO::Services::Collectd
    - OS::TripleO::Services::ComputeCeilometerAgent
    - OS::TripleO::Services::ComputeNeutronCorePlugin
    - OS::TripleO::Services::ComputeNeutronL3Agent
    - OS::TripleO::Services::ComputeNeutronMetadataAgent
    - OS::TripleO::Services::ComputeNeutronOvsAgent
    - OS::TripleO::Services::Docker
    - OS::TripleO::Services::IpaClient
    - OS::TripleO::Services::Ipsec
    - OS::TripleO::Services::Iscsid
    - OS::TripleO::Services::Kernel
    - OS::TripleO::Services::LoginDefs
    - OS::TripleO::Services::MetricsQdr
    - OS::TripleO::Services::Multipathd
    - OS::TripleO::Services::MySQLClient
    - OS::TripleO::Services::NeutronBgpVpnBagpipe
    - OS::TripleO::Services::NeutronLinuxbridgeAgent
    - OS::TripleO::Services::NeutronVppAgent
    - OS::TripleO::Services::NovaAZConfig
    - OS::TripleO::Services::NovaCompute
    - OS::TripleO::Services::NovaLibvirt
    - OS::TripleO::Services::NovaLibvirtGuests
    - OS::TripleO::Services::NovaMigrationTarget
    - OS::TripleO::Services::ContainersLogrotateCrond
    - OS::TripleO::Services::Podman
    - OS::TripleO::Services::Rear
    - OS::TripleO::Services::Rhsm
    - OS::TripleO::Services::Rsyslog
    - OS::TripleO::Services::RsyslogSidecar
    - OS::TripleO::Services::Securetty
    - OS::TripleO::Services::SkydiveAgent
    - OS::TripleO::Services::Snmp
    - OS::TripleO::Services::Sshd
    - OS::TripleO::Services::Timesync
    - OS::TripleO::Services::Timezone
    - OS::TripleO::Services::TripleoFirewall
    - OS::TripleO::Services::TripleoPackages
    - OS::TripleO::Services::Tuned
    - OS::TripleO::Services::Vpp
    - OS::TripleO::Services::OVNController
    - OS::TripleO::Services::OVNMetadataAgent
```

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

```

### 3.4. Tạo file `04-overcloud_images.yaml`
Tạo file `containers-prepare-parameter.yaml`

```sh
openstack tripleo container image prepare default \
  --local-push-destination \
  --output-env-file /home/stack/containers-prepare-parameter.yaml
```

Nội dung file `containers-prepare-parameter.yaml` sẽ như sau :

```sh
# Generated with the following on 2020-06-30T08:53:32.406551
#
#   openstack tripleo container image prepare default --local-push-destination --output-env-file containers-prepare-parameter.yaml
#

parameter_defaults:
  ContainerImagePrepare:
  - push_destination: 10.0.13.80:8787
    set:
      ceph_alertmanager_image: alertmanager
      ceph_alertmanager_namespace: docker.io/prom
      ceph_alertmanager_tag: v0.16.2
      ceph_grafana_image: grafana
      ceph_grafana_namespace: docker.io/grafana
      ceph_grafana_tag: 5.2.4
      ceph_image: daemon
      ceph_namespace: docker.io/ceph
      ceph_node_exporter_image: node-exporter
      ceph_node_exporter_namespace: docker.io/prom
      ceph_node_exporter_tag: v0.17.0
      ceph_prometheus_image: prometheus
      ceph_prometheus_namespace: docker.io/prom
      ceph_prometheus_tag: v2.7.2
      ceph_tag: v4.0.12-stable-4.0-nautilus-centos-7-x86_64
      name_prefix: centos-binary-
      name_suffix: ''
      namespace: docker.io/tripleotrain
      neutron_driver: ovn
      rhel_containers: false
      tag: current-tripleo
    tag_from_label: rdo_version

```

Tạo file `/home/stack/custom-templates/04-overcloud_images.yaml`

```sh
openstack overcloud container image prepare --push-destination=10.0.13.80:8787 --environment-file /home/stack/containers-prepare-parameter.yaml  --output-env-file /home/stack/custom-templates/04-overcloud_images.yaml --output-images-file /home/stack/custom-templates/local_registry_images.yaml

```

Nội dung file sẽ như sau : 

```sh
# Generated with the following on 2020-07-02T20:40:56.908793
#
#   openstack overcloud container image prepare --push-destination=10.0.13.80:8787 --environment-file containers-prepare-parameter.yaml --output-env-file /home/stack/custom-templates/04-overcloud_images.yaml --output-images-file /home/stack/custom-templates/local_registry_images.yaml
#

parameter_defaults:
  AlertManagerContainerImage: 10.0.13.80:8787/prom/alertmanager:v0.16.2
  ContainerAodhApiImage: 10.0.13.80:8787/tripleotrain/centos-binary-aodh-api:current-tripleo
  ContainerAodhConfigImage: 10.0.13.80:8787/tripleotrain/centos-binary-aodh-api:current-tripleo
  ContainerAodhEvaluatorImage: 10.0.13.80:8787/tripleotrain/centos-binary-aodh-evaluator:current-tripleo
  ContainerAodhListenerImage: 10.0.13.80:8787/tripleotrain/centos-binary-aodh-listener:current-tripleo
  ContainerAodhNotifierImage: 10.0.13.80:8787/tripleotrain/centos-binary-aodh-notifier:current-tripleo
  ContainerBarbicanApiImage: 10.0.13.80:8787/tripleotrain/centos-binary-barbican-api:current-tripleo
  ContainerBarbicanConfigImage: 10.0.13.80:8787/tripleotrain/centos-binary-barbican-api:current-tripleo
  ContainerBarbicanKeystoneListenerConfigImage: 10.0.13.80:8787/tripleotrain/centos-binary-barbican-keystone-listener:current-tripleo
  ContainerBarbicanKeystoneListenerImage: 10.0.13.80:8787/tripleotrain/centos-binary-barbican-keystone-listener:current-tripleo
  ContainerBarbicanWorkerConfigImage: 10.0.13.80:8787/tripleotrain/centos-binary-barbican-worker:current-tripleo
  ContainerBarbicanWorkerImage: 10.0.13.80:8787/tripleotrain/centos-binary-barbican-worker:current-tripleo
  ContainerCeilometerCentralImage: 10.0.13.80:8787/tripleotrain/centos-binary-ceilometer-central:current-tripleo
  ContainerCeilometerComputeImage: 10.0.13.80:8787/tripleotrain/centos-binary-ceilometer-compute:current-tripleo
  ContainerCeilometerConfigImage: 10.0.13.80:8787/tripleotrain/centos-binary-ceilometer-central:current-tripleo
  ContainerCeilometerNotificationImage: 10.0.13.80:8787/tripleotrain/centos-binary-ceilometer-notification:current-tripleo
  ContainerCephDaemonImage: 10.0.13.80:8787/ceph/daemon:v4.0.12-stable-4.0-nautilus-centos-7-x86_64
  ContainerCinderApiImage: 10.0.13.80:8787/tripleotrain/centos-binary-cinder-api:current-tripleo
  ContainerCinderBackupImage: 10.0.13.80:8787/tripleotrain/centos-binary-cinder-backup:current-tripleo
  ContainerCinderConfigImage: 10.0.13.80:8787/tripleotrain/centos-binary-cinder-api:current-tripleo
  ContainerCinderSchedulerImage: 10.0.13.80:8787/tripleotrain/centos-binary-cinder-scheduler:current-tripleo
  ContainerCinderVolumeImage: 10.0.13.80:8787/tripleotrain/centos-binary-cinder-volume:current-tripleo
  ContainerClustercheckConfigImage: 10.0.13.80:8787/tripleotrain/centos-binary-mariadb:current-tripleo
  ContainerClustercheckImage: 10.0.13.80:8787/tripleotrain/centos-binary-mariadb:current-tripleo
  ContainerCollectdConfigImage: 10.0.13.80:8787/tripleotrain/centos-binary-collectd:current-tripleo
  ContainerCollectdImage: 10.0.13.80:8787/tripleotrain/centos-binary-collectd:current-tripleo
  ContainerCrondConfigImage: 10.0.13.80:8787/tripleotrain/centos-binary-cron:current-tripleo
  ContainerCrondImage: 10.0.13.80:8787/tripleotrain/centos-binary-cron:current-tripleo
  ContainerDesignateApiImage: 10.0.13.80:8787/tripleotrain/centos-binary-designate-api:current-tripleo
  ContainerDesignateBackendBIND9Image: 10.0.13.80:8787/tripleotrain/centos-binary-designate-backend-bind9:current-tripleo
  ContainerDesignateCentralImage: 10.0.13.80:8787/tripleotrain/centos-binary-designate-central:current-tripleo
  ContainerDesignateConfigImage: 10.0.13.80:8787/tripleotrain/centos-binary-designate-worker:current-tripleo
  ContainerDesignateMDNSImage: 10.0.13.80:8787/tripleotrain/centos-binary-designate-mdns:current-tripleo
  ContainerDesignateProducerImage: 10.0.13.80:8787/tripleotrain/centos-binary-designate-producer:current-tripleo
  ContainerDesignateSinkImage: 10.0.13.80:8787/tripleotrain/centos-binary-designate-sink:current-tripleo
  ContainerDesignateWorkerImage: 10.0.13.80:8787/tripleotrain/centos-binary-designate-worker:current-tripleo
  ContainerEc2ApiConfigImage: 10.0.13.80:8787/tripleotrain/centos-binary-ec2-api:current-tripleo
  ContainerEc2ApiImage: 10.0.13.80:8787/tripleotrain/centos-binary-ec2-api:current-tripleo
  ContainerEtcdConfigImage: 10.0.13.80:8787/tripleotrain/centos-binary-etcd:current-tripleo
  ContainerEtcdImage: 10.0.13.80:8787/tripleotrain/centos-binary-etcd:current-tripleo
  ContainerGlanceApiConfigImage: 10.0.13.80:8787/tripleotrain/centos-binary-glance-api:current-tripleo
  ContainerGlanceApiImage: 10.0.13.80:8787/tripleotrain/centos-binary-glance-api:current-tripleo
  ContainerGnocchiApiImage: 10.0.13.80:8787/tripleotrain/centos-binary-gnocchi-api:current-tripleo
  ContainerGnocchiConfigImage: 10.0.13.80:8787/tripleotrain/centos-binary-gnocchi-api:current-tripleo
  ContainerGnocchiMetricdImage: 10.0.13.80:8787/tripleotrain/centos-binary-gnocchi-metricd:current-tripleo
  ContainerGnocchiStatsdImage: 10.0.13.80:8787/tripleotrain/centos-binary-gnocchi-statsd:current-tripleo
  ContainerHAProxyConfigImage: 10.0.13.80:8787/tripleotrain/centos-binary-haproxy:current-tripleo
  ContainerHAProxyImage: 10.0.13.80:8787/tripleotrain/centos-binary-haproxy:current-tripleo
  ContainerHeatApiCfnConfigImage: 10.0.13.80:8787/tripleotrain/centos-binary-heat-api-cfn:current-tripleo
  ContainerHeatApiCfnImage: 10.0.13.80:8787/tripleotrain/centos-binary-heat-api-cfn:current-tripleo
  ContainerHeatApiConfigImage: 10.0.13.80:8787/tripleotrain/centos-binary-heat-api:current-tripleo
  ContainerHeatApiImage: 10.0.13.80:8787/tripleotrain/centos-binary-heat-api:current-tripleo
  ContainerHeatConfigImage: 10.0.13.80:8787/tripleotrain/centos-binary-heat-api:current-tripleo
  ContainerHeatEngineImage: 10.0.13.80:8787/tripleotrain/centos-binary-heat-engine:current-tripleo
  ContainerHorizonConfigImage: 10.0.13.80:8787/tripleotrain/centos-binary-horizon:current-tripleo
  ContainerHorizonImage: 10.0.13.80:8787/tripleotrain/centos-binary-horizon:current-tripleo
  ContainerIronicApiConfigImage: 10.0.13.80:8787/tripleotrain/centos-binary-ironic-api:current-tripleo
  ContainerIronicApiImage: 10.0.13.80:8787/tripleotrain/centos-binary-ironic-api:current-tripleo
  ContainerIronicConductorImage: 10.0.13.80:8787/tripleotrain/centos-binary-ironic-conductor:current-tripleo
  ContainerIronicConfigImage: 10.0.13.80:8787/tripleotrain/centos-binary-ironic-pxe:current-tripleo
  ContainerIronicInspectorConfigImage: 10.0.13.80:8787/tripleotrain/centos-binary-ironic-inspector:current-tripleo
  ContainerIronicInspectorImage: 10.0.13.80:8787/tripleotrain/centos-binary-ironic-inspector:current-tripleo
  ContainerIronicNeutronAgentImage: 10.0.13.80:8787/tripleotrain/centos-binary-ironic-neutron-agent:current-tripleo
  ContainerIronicPxeImage: 10.0.13.80:8787/tripleotrain/centos-binary-ironic-pxe:current-tripleo
  ContainerIscsidConfigImage: 10.0.13.80:8787/tripleotrain/centos-binary-iscsid:current-tripleo
  ContainerIscsidImage: 10.0.13.80:8787/tripleotrain/centos-binary-iscsid:current-tripleo
  ContainerKeepalivedConfigImage: 10.0.13.80:8787/tripleotrain/centos-binary-keepalived:current-tripleo
  ContainerKeepalivedImage: 10.0.13.80:8787/tripleotrain/centos-binary-keepalived:current-tripleo
  ContainerKeystoneConfigImage: 10.0.13.80:8787/tripleotrain/centos-binary-keystone:current-tripleo
  ContainerKeystoneImage: 10.0.13.80:8787/tripleotrain/centos-binary-keystone:current-tripleo
  ContainerManilaApiImage: 10.0.13.80:8787/tripleotrain/centos-binary-manila-api:current-tripleo
  ContainerManilaConfigImage: 10.0.13.80:8787/tripleotrain/centos-binary-manila-api:current-tripleo
  ContainerManilaSchedulerImage: 10.0.13.80:8787/tripleotrain/centos-binary-manila-scheduler:current-tripleo
  ContainerManilaShareImage: 10.0.13.80:8787/tripleotrain/centos-binary-manila-share:current-tripleo
  ContainerMemcachedConfigImage: 10.0.13.80:8787/tripleotrain/centos-binary-memcached:current-tripleo
  ContainerMemcachedImage: 10.0.13.80:8787/tripleotrain/centos-binary-memcached:current-tripleo
  ContainerMetricsQdrConfigImage: 10.0.13.80:8787/tripleotrain/centos-binary-qdrouterd:current-tripleo
  ContainerMetricsQdrImage: 10.0.13.80:8787/tripleotrain/centos-binary-qdrouterd:current-tripleo
  ContainerMistralApiImage: 10.0.13.80:8787/tripleotrain/centos-binary-mistral-api:current-tripleo
  ContainerMistralConfigImage: 10.0.13.80:8787/tripleotrain/centos-binary-mistral-api:current-tripleo
  ContainerMistralEngineImage: 10.0.13.80:8787/tripleotrain/centos-binary-mistral-engine:current-tripleo
  ContainerMistralEventEngineImage: 10.0.13.80:8787/tripleotrain/centos-binary-mistral-event-engine:current-tripleo
  ContainerMistralExecutorImage: 10.0.13.80:8787/tripleotrain/centos-binary-mistral-executor:current-tripleo
  ContainerMultipathdConfigImage: 10.0.13.80:8787/tripleotrain/centos-binary-multipathd:current-tripleo
  ContainerMultipathdImage: 10.0.13.80:8787/tripleotrain/centos-binary-multipathd:current-tripleo
  ContainerMysqlClientConfigImage: 10.0.13.80:8787/tripleotrain/centos-binary-mariadb:current-tripleo
  ContainerMysqlConfigImage: 10.0.13.80:8787/tripleotrain/centos-binary-mariadb:current-tripleo
  ContainerMysqlImage: 10.0.13.80:8787/tripleotrain/centos-binary-mariadb:current-tripleo
  ContainerNeutronApiImage: 10.0.13.80:8787/tripleotrain/centos-binary-neutron-server-ovn:current-tripleo
  ContainerNeutronConfigImage: 10.0.13.80:8787/tripleotrain/centos-binary-neutron-server-ovn:current-tripleo
  ContainerNeutronDHCPImage: 10.0.13.80:8787/tripleotrain/centos-binary-neutron-dhcp-agent:current-tripleo
  ContainerNeutronL3AgentImage: 10.0.13.80:8787/tripleotrain/centos-binary-neutron-l3-agent:current-tripleo
  ContainerNeutronMetadataImage: 10.0.13.80:8787/tripleotrain/centos-binary-neutron-metadata-agent:current-tripleo
  ContainerNovaApiImage: 10.0.13.80:8787/tripleotrain/centos-binary-nova-api:current-tripleo
  ContainerNovaComputeImage: 10.0.13.80:8787/tripleotrain/centos-binary-nova-compute:current-tripleo
  ContainerNovaComputeIronicImage: 10.0.13.80:8787/tripleotrain/centos-binary-nova-compute-ironic:current-tripleo
  ContainerNovaConductorImage: 10.0.13.80:8787/tripleotrain/centos-binary-nova-conductor:current-tripleo
  ContainerNovaConfigImage: 10.0.13.80:8787/tripleotrain/centos-binary-nova-api:current-tripleo
  ContainerNovaLibvirtConfigImage: 10.0.13.80:8787/tripleotrain/centos-binary-nova-compute:current-tripleo
  ContainerNovaLibvirtImage: 10.0.13.80:8787/tripleotrain/centos-binary-nova-libvirt:current-tripleo
  ContainerNovaMetadataConfigImage: 10.0.13.80:8787/tripleotrain/centos-binary-nova-api:current-tripleo
  ContainerNovaMetadataImage: 10.0.13.80:8787/tripleotrain/centos-binary-nova-api:current-tripleo
  ContainerNovaSchedulerImage: 10.0.13.80:8787/tripleotrain/centos-binary-nova-scheduler:current-tripleo
  ContainerNovaVncProxyImage: 10.0.13.80:8787/tripleotrain/centos-binary-nova-novncproxy:current-tripleo
  ContainerOctaviaApiImage: 10.0.13.80:8787/tripleotrain/centos-binary-octavia-api:current-tripleo
  ContainerOctaviaConfigImage: 10.0.13.80:8787/tripleotrain/centos-binary-octavia-api:current-tripleo
  ContainerOctaviaDriverAgentConfigImage: 10.0.13.80:8787/tripleotrain/centos-binary-octavia-api:current-tripleo
  ContainerOctaviaDriverAgentImage: 10.0.13.80:8787/tripleotrain/centos-binary-octavia-api:current-tripleo
  ContainerOctaviaHealthManagerImage: 10.0.13.80:8787/tripleotrain/centos-binary-octavia-health-manager:current-tripleo
  ContainerOctaviaHousekeepingImage: 10.0.13.80:8787/tripleotrain/centos-binary-octavia-housekeeping:current-tripleo
  ContainerOctaviaRsyslogImage: 10.0.13.80:8787/tripleotrain/centos-binary-rsyslog:current-tripleo
  ContainerOctaviaWorkerImage: 10.0.13.80:8787/tripleotrain/centos-binary-octavia-worker:current-tripleo
  ContainerOpenvswitchImage: 10.0.13.80:8787/tripleotrain/centos-binary-neutron-openvswitch-agent:current-tripleo
  ContainerOvnControllerConfigImage: 10.0.13.80:8787/tripleotrain/centos-binary-ovn-controller:current-tripleo
  ContainerOvnControllerImage: 10.0.13.80:8787/tripleotrain/centos-binary-ovn-controller:current-tripleo
  ContainerOvnDbsConfigImage: 10.0.13.80:8787/tripleotrain/centos-binary-ovn-northd:current-tripleo
  ContainerOvnDbsImage: 10.0.13.80:8787/tripleotrain/centos-binary-ovn-northd:current-tripleo
  ContainerOvnMetadataImage: 10.0.13.80:8787/tripleotrain/centos-binary-neutron-metadata-agent-ovn:current-tripleo
  ContainerOvnNbDbImage: 10.0.13.80:8787/tripleotrain/centos-binary-ovn-nb-db-server:current-tripleo
  ContainerOvnNorthdImage: 10.0.13.80:8787/tripleotrain/centos-binary-ovn-northd:current-tripleo
  ContainerOvnSbDbImage: 10.0.13.80:8787/tripleotrain/centos-binary-ovn-sb-db-server:current-tripleo
  ContainerPankoApiImage: 10.0.13.80:8787/tripleotrain/centos-binary-panko-api:current-tripleo
  ContainerPankoConfigImage: 10.0.13.80:8787/tripleotrain/centos-binary-panko-api:current-tripleo
  ContainerPlacementConfigImage: 10.0.13.80:8787/tripleotrain/centos-binary-placement-api:current-tripleo
  ContainerPlacementImage: 10.0.13.80:8787/tripleotrain/centos-binary-placement-api:current-tripleo
  ContainerQdrouterdConfigImage: 10.0.13.80:8787/tripleotrain/centos-binary-qdrouterd:current-tripleo
  ContainerQdrouterdImage: 10.0.13.80:8787/tripleotrain/centos-binary-qdrouterd:current-tripleo
  ContainerRabbitmqConfigImage: 10.0.13.80:8787/tripleotrain/centos-binary-rabbitmq:current-tripleo
  ContainerRabbitmqImage: 10.0.13.80:8787/tripleotrain/centos-binary-rabbitmq:current-tripleo
  ContainerRedisConfigImage: 10.0.13.80:8787/tripleotrain/centos-binary-redis:current-tripleo
  ContainerRedisImage: 10.0.13.80:8787/tripleotrain/centos-binary-redis:current-tripleo
  ContainerRsyslogConfigImage: 10.0.13.80:8787/tripleotrain/centos-binary-rsyslog:current-tripleo
  ContainerRsyslogImage: 10.0.13.80:8787/tripleotrain/centos-binary-rsyslog:current-tripleo
  ContainerRsyslogSidecarConfigImage: 10.0.13.80:8787/tripleotrain/centos-binary-rsyslog:current-tripleo
  ContainerRsyslogSidecarImage: 10.0.13.80:8787/tripleotrain/centos-binary-rsyslog:current-tripleo
  ContainerSaharaApiImage: 10.0.13.80:8787/tripleotrain/centos-binary-sahara-api:current-tripleo
  ContainerSaharaConfigImage: 10.0.13.80:8787/tripleotrain/centos-binary-sahara-api:current-tripleo
  ContainerSaharaEngineImage: 10.0.13.80:8787/tripleotrain/centos-binary-sahara-engine:current-tripleo
  ContainerSkydiveAgentImage: 10.0.13.80:8787/tripleotrain/centos-binary-skydive-agent:current-tripleo
  ContainerSkydiveAnalyzerImage: 10.0.13.80:8787/tripleotrain/centos-binary-skydive-analyzer:current-tripleo
  ContainerSwiftAccountImage: 10.0.13.80:8787/tripleotrain/centos-binary-swift-account:current-tripleo
  ContainerSwiftConfigImage: 10.0.13.80:8787/tripleotrain/centos-binary-swift-proxy-server:current-tripleo
  ContainerSwiftContainerImage: 10.0.13.80:8787/tripleotrain/centos-binary-swift-container:current-tripleo
  ContainerSwiftObjectImage: 10.0.13.80:8787/tripleotrain/centos-binary-swift-object:current-tripleo
  ContainerSwiftProxyImage: 10.0.13.80:8787/tripleotrain/centos-binary-swift-proxy-server:current-tripleo
  ContainerZaqarConfigImage: 10.0.13.80:8787/tripleotrain/centos-binary-zaqar-wsgi:current-tripleo
  ContainerZaqarImage: 10.0.13.80:8787/tripleotrain/centos-binary-zaqar-wsgi:current-tripleo
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
  imagename: docker.io/tripleotrain/centos-binary-aodh-api:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-aodh-evaluator:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-aodh-listener:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-aodh-notifier:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-barbican-api:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-barbican-keystone-listener:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-barbican-worker:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-ceilometer-central:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-ceilometer-compute:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-ceilometer-notification:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-cinder-api:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-cinder-backup:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-cinder-scheduler:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-cinder-volume:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-collectd:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-cron:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-designate-api:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-designate-backend-bind9:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-designate-central:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-designate-mdns:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-designate-producer:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-designate-sink:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-designate-worker:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-ec2-api:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-etcd:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-glance-api:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-gnocchi-api:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-gnocchi-metricd:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-gnocchi-statsd:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-haproxy:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-heat-api-cfn:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-heat-api:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-heat-engine:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-horizon:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-ironic-api:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-ironic-conductor:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-ironic-inspector:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-ironic-pxe:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-ironic-neutron-agent:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-iscsid:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-keepalived:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-keystone:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-manila-api:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-manila-scheduler:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-manila-share:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-mariadb:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-memcached:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-mistral-api:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-mistral-engine:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-mistral-executor:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-mistral-event-engine:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-multipathd:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-neutron-dhcp-agent:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-neutron-l3-agent:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-neutron-metadata-agent:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-neutron-openvswitch-agent:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-neutron-server-ovn:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-neutron-metadata-agent-ovn:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-nova-api:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-nova-compute-ironic:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-nova-compute:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-nova-conductor:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-nova-libvirt:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-nova-novncproxy:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-nova-scheduler:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-octavia-api:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-octavia-health-manager:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-octavia-housekeeping:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-octavia-worker:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-ovn-controller:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-ovn-nb-db-server:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-ovn-northd:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-ovn-sb-db-server:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-panko-api:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-placement-api:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-qdrouterd:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-rabbitmq:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-redis:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-sahara-api:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-sahara-engine:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-skydive-agent:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-skydive-analyzer:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-swift-account:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-swift-container:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-swift-object:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-swift-proxy-server:current-tripleo
  push_destination: 10.0.13.80:8787
- image_source: kolla
  imagename: docker.io/tripleotrain/centos-binary-zaqar-wsgi:current-tripleo
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
  imagename: docker.io/tripleotrain/centos-binary-rsyslog:current-tripleo
  push_destination: 10.0.13.80:8787
```

Push các image lên local-registry tại node UnderCloud :

```sh
sudo openstack overcloud container image upload  --config-file /home/stack/custom-templates/local_registry_images.yaml
```

### 3.5. Tạo file `05-network-environment.yaml`

```sh

resource_registry:
  OS::TripleO::Controller::Net::SoftwareConfig: /home/stack/custom-templates/custom-nics/controller.yaml
  OS::TripleO::Compute::Net::SoftwareConfig: /home/stack/custom-templates/custom-nics/compute.yaml

parameter_defaults:
  EC2MetadataIp: 10.0.13.80
  ControlPlaneDefaultRoute: 10.0.13.1

  StorageNetCidr: '10.0.15.0/24'
  StorageAllocationPools: [{'start': '10.0.15.131', 'end': '10.0.15.199'}]
  StorageNetworkVlanID: 15

  InternalApiNetCidr: '10.0.11.0/24'
  InternalApiAllocationPools: [{'start': '10.0.11.131', 'end': '10.0.11.199'}]
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

  DnsServers: 8.8.8.8
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

Tạo file cấu hình `controller.yaml`

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

  EC2MetadataIp:
    description: The IP address of the EC2 metadata server.
    type: string

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
                  -
                    ip_netmask: 169.254.169.254/32
                    next_hop: {get_param: EC2MetadataIp}
                  -
                    default: true
                    next_hop: {get_param: ControlPlaneDefaultRoute}
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

Bước 5 : Tạo file `compute.yaml`

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
    description: >
      The subnet CIDR of the control plane network. (The parameter is
      automatically resolved from the ctlplane subnet's cidr attribute.)
    type: string
  ControlPlaneDefaultRoute:
    default: ''
    description: The default route of the control plane network. (The parameter
      is automatically resolved from the ctlplane subnet's gateway_ip attribute.)
    type: string
  ControlPlaneStaticRoutes:
    default: []
    description: >
      Routes for the ctlplane network traffic.
      JSON route e.g. [{'destination':'10.0.0.0/16', 'nexthop':'10.0.0.1'}]
      Unless the default is changed, the parameter is automatically resolved
      from the subnet host_routes attribute.
    type: json
  ControlPlaneMtu:
    default: 1500
    description: The maximum transmission unit (MTU) size(in bytes) that is
      guaranteed to pass through the data path of the segments in the network.
      (The parameter is automatically resolved from the ctlplane network's mtu attribute.)
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
    description: The maximum transmission unit (MTU) size(in bytes) that is
      guaranteed to pass through the data path of the segments in the
      Storage network.
    type: number
  StorageInterfaceRoutes:
    default: []
    description: >
      Routes for the storage network traffic.
      JSON route e.g. [{'destination':'10.0.0.0/16', 'nexthop':'10.0.0.1'}]
      Unless the default is changed, the parameter is automatically resolved
      from the subnet host_routes attribute.
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
    description: The maximum transmission unit (MTU) size(in bytes) that is
      guaranteed to pass through the data path of the segments in the
      InternalApi network.
    type: number
  InternalApiInterfaceRoutes:
    default: []
    description: >
      Routes for the internal_api network traffic.
      JSON route e.g. [{'destination':'10.0.0.0/16', 'nexthop':'10.0.0.1'}]
      Unless the default is changed, the parameter is automatically resolved
      from the subnet host_routes attribute.
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
    description: The maximum transmission unit (MTU) size(in bytes) that is
      guaranteed to pass through the data path of the segments in the
      Tenant network.
    type: number
  TenantInterfaceRoutes:
    default: []
    description: >
      Routes for the tenant network traffic.
      JSON route e.g. [{'destination':'10.0.0.0/16', 'nexthop':'10.0.0.1'}]
      Unless the default is changed, the parameter is automatically resolved
      from the subnet host_routes attribute.
    type: json

  ExternalMtu:
    default: 1500
    description: The maximum transmission unit (MTU) size(in bytes) that is
      guaranteed to pass through the data path of the segments in the
      External network.
    type: number

  EC2MetadataIp: 
    description: The IP address of the EC2 metadata server.
    type: string
    
  DnsServers: # Override this via parameter_defaults
    default: []
    description: >
      DNS servers to use for the Overcloud (2 max for some implementations).
      If not set the nameservers configured in the ctlplane subnet's
      dns_nameservers attribute will be used.
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
                  - 
                    ip_netmask: 169.254.169.254/32
                    next_hop: {get_param: EC2MetadataIp}
                  -
                    default: true
                    next_hop: {get_param: ControlPlaneDefaultRoute}
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

### 3.6 Tạo file `/home/stack/custom-templates/06-extra-parameters.yaml`

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



### 3.7. Tạo file gắn cụ thể các IP cố định tới các node `/home/stack/custom-templates/07-predictive-ips.yaml`

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

### 3.8. Đặt file map hostname của các node `/home/stack/custom-templates/08-scheduler-hints-env.yaml`

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

### 3.9. Tạo file `/home/stack/custom-templates/09-no-tls-endpoint-public.yaml`

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

### 3.10. Tích hợp với Ceph-External

Mô hình của chúng ta sử dụng Storage là Ceph External dành cho các dịch vụ sau : lưu trữ image của Glance, lưu trữ volume và backup snapshot của Cinder.

Thực hiện cài hệ thống Ceph Nautilus với đúng Topo và IP-Planning ở trên theo hướng dẫn [này](https://github.com/uncelvel/tutorial-ceph/blob/master/docs/setup/ceph-ansible-nautilus.md)

Tạo pool trên ceph cho glance va cinder tren node ceph
```sh
ceph osd pool create volumes 64 64
ceph osd pool create images 32 32
ceph osd pool create backups 8 8
```

Init pool tren node ceph 

```sh
rbd pool init volumes
rbd pool init images
rbd pool init backups
```

Thực hiện tạo `client.openstack` cho Ceph Cluster với câu lệnh sau :

```sh
ceph auth add client.openstack mgr 'allow *' mon 'profile rbd' osd 'profile rbd pool=volumes,  profile rbd pool=images, profile rbd pool=backups'
```

Sau khi cài xong Ceph, ta cần thực hiện lấy các tham số sau :

 - CephClientKey

 Thực hiện lệnh `ceph auth list`, tìm kiếm key của `client.openstack` như sau :
 
![tripleo](/OpenStack/images/tripleo-esxi-10.png)

 - CephClusterFSID

Thực hiện lệnh `ceph -s` và lấy Ceph cluster ID như sau :

![tripleo](/OpenStack/images/tripleo-esxi-11.png)

 - CephExternalMonHost
Các IP dải Storage của các node Ceph.

Tạo file để tích hợp với Ceph-External `/home/stack/custom-templates/10-ceph-external.yaml` với nội dung sau :

```sh
resource_registry:
  OS::TripleO::Services::CephExternal: /usr/share/openstack-tripleo-heat-templates/deployment/ceph-ansible/ceph-external.yaml

parameter_defaults:
  NovaEnableRbdBackend: false
  CinderEnableRbdBackend: true
  CinderBackupBackend: ceph
  GlanceBackend: rbd
  CinderRbdPoolName: volumes
  CinderBackupRbdPoolName: backups
  GlanceRbdPoolName: images
  CephClientUserName: openstack

  CephClientKey: AQD6v/FeIhOZNBAAd5t16th3R6HMZWZ8PJPRoQ==
  CephClusterFSID: cf025dfc-9c52-404a-a473-6f835978d4e1
  CephExternalMonHost: 10.0.15.87, 10.0.15.88, 10.0.15.89
  CinderEnableIscsiBackend: false
```

### 3.11. Tạo file `/home/stack/custom-templates/11-container-cli.yaml`
Trên bản Train, có thể sử dụng câu lệnh quản lý container là `podman` hoặc `docker`. Ta chọn sử dụng `container_cli` là `docker`. ( Tại bản Train, podman vẫn chưa ổn định và có nhiều bug)

Nội dung file `11-container-cli.yaml` :

```sh
# DEPRECATED: Containerized deployments with Docker are deprecated. This file
# will be removed in Train release.

# Environment that enables Docker.
resource_registry:
  OS::TripleO::Services::Docker: /usr/share/openstack-tripleo-heat-templates/deployment/deprecated/docker/docker-baremetal-ansible.yaml

parameter_defaults:
  ContainerCli: docker
```

### 3.12. Tạo file `/home/stack/custom-templates/12-disable-swift.yaml`

Tạm thời không sử dụng Swift nên sẽ disable project này. Tạo file disable `12-disable-swift.yaml` với nội dung :

```sh
resource_registry:
  OS::TripleO::Services::SwiftProxy: OS::Heat::None
  OS::TripleO::Services::SwiftStorage: OS::Heat::None
  OS::TripleO::Services::SwiftRingBuilder: OS::Heat::None
  OS::TripleO::Services::SwiftDispersion: OS::Heat::None
  OS::TripleO::Services::ExternalSwiftProxy: OS::Heat::None

parameter_defaults:
 ControllerEnableSwiftStorage: false
```

### 3.13. Tạo file `/home/stack/custom-templates/13-disable-panko.yaml`

Tạm thời không sử dụng Panko nên sẽ disable project này. Tạo file disable `13-disable-panko.yaml` với nội dung :

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

### 3.14. Tạo file `/home/stack/custom-templates/14-disable-telemetry.yaml`

Tạm thời không sử dụng Panko nên sẽ disable project này. Tạo file disable `14-disable-telemetry.yaml` với nội dung :

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

### 3.15. Tạo file `/home/stack/custom-templates/15-docker-ha.yaml`

Thực hiện tạo file `15-docker-ha.yaml` để khai báo các dịch vụ HA cần cài với pacemaker và corosync. Nội dung file như sau :

```sh
# Environment file to deploy the HA services via docker
# Add it *after* -e docker.yaml:
# ...deploy..-e docker.yaml -e docker-ha.yaml
resource_registry:
  # Pacemaker runs on the host
  # FIXME(bogdando): switch it, once it is containerized
  OS::TripleO::Services::Pacemaker: /usr/share/openstack-tripleo-heat-templates/deployment/pacemaker/pacemaker-baremetal-puppet.yaml
  # FIXME(bogdando): switch it, once it is containerized
  OS::TripleO::Services::PacemakerRemote: /usr/share/openstack-tripleo-heat-templates/deployment/pacemaker/pacemaker-remote-baremetal-puppet.yaml
  OS::TripleO::Tasks::ControllerPreConfig: OS::Heat::None
  OS::TripleO::Tasks::ControllerPostConfig: OS::Heat::None

  # Services that are disabled for HA deployments with pacemaker
  OS::TripleO::Services::Keepalived: OS::Heat::None

  # HA Containers managed by pacemaker
  OS::TripleO::Services::CinderVolume: /usr/share/openstack-tripleo-heat-templates/deployment/cinder/cinder-volume-pacemaker-puppet.yaml
  OS::TripleO::Services::Clustercheck: /usr/share/openstack-tripleo-heat-templates/deployment/pacemaker/clustercheck-container-puppet.yaml
  OS::TripleO::Services::HAproxy: /usr/share/openstack-tripleo-heat-templates/deployment/haproxy/haproxy-pacemaker-puppet.yaml
  OS::TripleO::Services::MySQL: /usr/share/openstack-tripleo-heat-templates/deployment/database/mysql-pacemaker-puppet.yaml
  OS::TripleO::Services::OsloMessagingRpc: /usr/share/openstack-tripleo-heat-templates/deployment/rabbitmq/rabbitmq-messaging-rpc-pacemaker-puppet.yaml
  OS::TripleO::Services::OsloMessagingNotify: /usr/share/openstack-tripleo-heat-templates/deployment/rabbitmq/rabbitmq-messaging-notify-shared-puppet.yaml
  OS::TripleO::Services::Redis: /usr/share/openstack-tripleo-heat-templates/deployment/database/redis-pacemaker-puppet.yaml
  OS::TripleO::Services::OVNDBs: /usr/share/openstack-tripleo-heat-templates/deployment/ovn/ovn-dbs-pacemaker-puppet.yaml

parameter_defaults:
  ContainerCli: docker
  ClusterCommonTag: true

```

Câu lệnh boot : 

```sh
openstack overcloud deploy --templates \
-r /home/stack/custom-templates/01-roles-data.yaml \
-e /home/stack/custom-templates/02-node-info.yaml \
-e /home/stack/custom-templates/03-network-isolation.yaml \
-e /home/stack/custom-templates/04-overcloud_images.yaml \
-e /home/stack/custom-templates/05-network-environment.yaml \
-e /home/stack/custom-templates/06-extra-parameters.yaml \
-e /home/stack/custom-templates/07-predictive-ips.yaml \
-e /home/stack/custom-templates/08-scheduler-hints-env.yaml \
-e /home/stack/custom-templates/09-no-tls-endpoint-public.yaml \
-e /home/stack/custom-templates/10-ceph-external.yaml \
-e /home/stack/custom-templates/11-container-cli.yaml \
-e /home/stack/custom-templates/12-disable-swift.yaml \
-e /home/stack/custom-templates/13-disable-panko.yaml \
-e /home/stack/custom-templates/14-disable-telemetry.yaml \
-e /home/stack/custom-templates/15-docker-ha.yaml \
--log-file overcloud_deployment.log --timeout 360
```