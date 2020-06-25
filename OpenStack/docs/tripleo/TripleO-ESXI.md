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

## 3 Xử lý Network

### 3.1. Xử lý cấu hình các file Template network

Xử lý file template ~/templates/network_data.yaml

```sh
- name: Storage
  vip: true
  vlan: 15
  name_lower: storage
  ip_subnet: '10.0.15.0/24'
  allocation_pools: [{'start': '10.0.15.181', 'end': '10.0.15.199'}]
  mtu: 1500
- name: InternalApi
  name_lower: internal_api
  vip: true
  vlan: 11
  ip_subnet: '10.0.11.0/24'
  allocation_pools: [{'start': '10.0.11.181', 'end': '10.0.11.199'}]
  mtu: 1500
- name: Tenant
  vip: false  # Tenant network does not use VIPs
  name_lower: tenant
  vlan: 12
  ip_subnet: '10.0.12.0/24'
  allocation_pools: [{'start': '10.0.12.181', 'end': '10.0.12.195'}]
  mtu: 1500
- name: External
  vip: true
  name_lower: external
  vlan: 17
  ip_subnet: '10.0.17.0/24'
  allocation_pools: [{'start': '10.0.17.181', 'end': '10.0.17.199'}]
  gateway_ip: '10.0.17.1'
  mtu: 1500
```

Render toàn bộ các file dạng Jinja2 sang dạng YAML 

```sh
cd /usr/share/openstack-tripleo-heat-templates
./tools/process-templates.py -o ~/openstack-tripleo-heat-templates-rendered
```

### 3.2. Customize network

Thực hiện cấu hình theo dạng multple NIC. Các file cần thực hiện cấu hình như sau : 

 - File network isolaton : `network-isolation.yaml`
 - File network default : `network-environment.yaml`
 - File network data tùy chỉnh, dùng để tạo thêm các network từ bên ngoài (Các đường PUBLIC provider network ) : `network_data`
 - File tùy chỉnh `roles_data` để gán các network mới vào role.
 - Template network để định nghĩa `NIC layout` cho mỗi node.
 - File environment để cho phép các NIC. Mặc định sẽ sử dụng các file trong thư mục `environments`.
 - Bất kỳ file cấu hình tủy chỉnh network của hệ thống.

Bước 1 : tạo thư mục các file cấu hình :

```sh
 mkdir ~/custom-templates
 ```

Tại thư mục custom template chứa 3 file cấu hình sau : 

```sh
/home/stack/templates/custom-nics/controller.yaml
/home/stack/templates/custom-nics/compute.yaml
```

Bước 2 : Thực hiện cấu hình `/home/stack/templates/network-environment.yaml` 

```sh

resource_registry:
  "OS::TripleO::Controller::Net::SoftwareConfig": /home/stack/templates/custom-nics/controller.yaml
  "OS::TripleO::Compute::Net::SoftwareConfig": /home/stack/templates/custom-nics/compute.yaml

parameter_defaults:
  StorageNetCidr: '10.0.15.0/24'
  StorageAllocationPools: [{'start': '10.0.15.181', 'end': '10.0.15.199'}]
  StorageNetworkVlanID: 15

  InternalApiNetCidr: '10.0.11.0/24'
  InternalApiAllocationPools: [{'start': '10.0.11.181', 'end': '10.0.11.199'}]
  InternalApiNetworkVlanID: 11

  TenantNetCidr: '10.0.12.0/24'
  TenantAllocationPools: [{'start': '10.0.12.181', 'end': '10.0.12.199'}]
  TenantNetworkVlanID: 12
  TenantNetPhysnetMtu: 1500

  ExternalNetCidr: '10.0.17.0/24'
  ExternalAllocationPools: [{'start': '10.0.17.181', 'end': '10.0.17.199'}]
  ExternalInterfaceDefaultRoute: '10.0.17.1'
  ExternalNetworkVlanID: 17

  ControlFixedIPs: [{'ip_address':'10.0.13.100'}]
  InternalApiVirtualFixedIPs: [{'ip_address':'10.0.11.100'}]
  PublicVirtualFixedIPs: [{'ip_address':'10.0.17.100'}]
  StorageVirtualFixedIPs: [{'ip_address':'10.0.15.100'}]
  RedisVirtualFixedIPs: [{'ip_address':'10.0.11.101'}]

  DnsServers: []
  NeutronNetworkType: 'geneve,vlan'
  NeutronNetworkVLANRanges: 'datacentre:1:1000'
```

Bước 3 : Định nghĩa cấu hình NIC của Server

Copy file config NIC vào thư mục chứa file câu hình của Controller và Compute : 
```sh
cp /usr/share/openstack-tripleo-heat-templates/network/scripts/run-os-net-config.sh /home/stack/templates/custom-nics/
```

Tại file `netwowrk-environment.yaml` có sử dụng các registry  gồm các cấu hình network của Controller và Compute. 

Tạo các file cấu hình như sau : 

Bước 4 : Tạo file cấu hình `/home/stack/templates/custom-nics/controller.yaml`

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
                name: nic3
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
                    - get_param: ControlPlaneStaticRoutes
              - type: interface
                name: nic4
                mtu:
                  get_param: StorageMtu
                use_dhcp: false
                addresses:
                - ip_netmask:
                    get_param: StorageIpSubnet
                routes:
                  list_concat_unique:
                    - get_param: StorageInterfaceRoutes
              - type: interface
                name: nic1
                mtu:
                  get_param: InternalApiMtu
                use_dhcp: false
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
                use_dhcp: false
                addresses:
                - ip_netmask:
                    get_param: TenantIpSubnet
                routes:
                  list_concat_unique:
                    - get_param: TenantInterfaceRoutes
                members:
                - type: interface
                  name: nic2
                  mtu:
                    get_param: TenantMtu
                  use_dhcp: false
                  primary: true
              - type: ovs_bridge
                name: bridge_name
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
                  name: nic5
                  mtu:
                    get_param: ExternalMtu
                  use_dhcp: false
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
    description: The maximum transmission unit (MTU) size(in bytes) that is
      guaranteed to pass through the data path of the segments in the
      Storage network.
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
    description: The maximum transmission unit (MTU) size(in bytes) that is
      guaranteed to pass through the data path of the segments in the
      External network.
    type: number

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
                name: nic3
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
                    - get_param: ControlPlaneStaticRoutes
                    - - default: true
                        next_hop:
                          get_param: ControlPlaneDefaultRoute
              - type: interface
                name: nic4
                mtu:
                  get_param: StorageMtu
                use_dhcp: false
                addresses:
                - ip_netmask:
                    get_param: StorageIpSubnet
                routes:
                  list_concat_unique:
                    - get_param: StorageInterfaceRoutes
              - type: interface
                name: nic1
                mtu:
                  get_param: InternalApiMtu
                use_dhcp: false
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
                use_dhcp: false
                addresses:
                - ip_netmask:
                    get_param: TenantIpSubnet
                routes:
                  list_concat_unique:
                    - get_param: TenantInterfaceRoutes
                members:
                - type: interface
                  name: nic2
                  mtu:
                    get_param: TenantMtu
                  use_dhcp: false
                  primary: true             
outputs:
  OS::stack_id:
    description: The OsNetConfigImpl resource.
    value:
      get_resource: OsNetConfigImpl

```

Tạo file ` /home/stack/templates/node-info.yaml`

```sh
parameter_defaults:
  OvercloudControllerFlavor: control
  OvercloudComputeFlavor: compute
  ControllerCount: 3
  ComputeCount: 3
```

Tạo file `/home/stack/templates/inject-trust-anchor-hiera.yaml` với nội dung sau của `content` là file `/etc/pki/ca-trust/source/anchors/cm-local-ca.pem` :

```sh
parameter_defaults:
  CAMap:
    undercloud-ca:
      content: |
      -----BEGIN CERTIFICATE-----
      MIIDjTCCAnWgAwIBAgIQVQokqJ1HRv2BQ0THxF2MhTANBgkqhkiG9w0BAQsFADBQ
      MSAwHgYDVQQDDBdMb2NhbCBTaWduaW5nIEF1dGhvcml0eTEsMCoGA1UEAwwjNTUw
      YTI0YTgtOWQ0NzQ2ZmQtODE0MzQ0YzctYzQ1ZDhjODUwHhcNMjAwNjE4MTE1MzA2
      WhcNMjEwNjE4MTE1MzA2WjBQMSAwHgYDVQQDDBdMb2NhbCBTaWduaW5nIEF1dGhv
      cml0eTEsMCoGA1UEAwwjNTUwYTI0YTgtOWQ0NzQ2ZmQtODE0MzQ0YzctYzQ1ZDhj
      ODUwggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQD3d5u9kzHdsw27n4jX
      SdyITIH9dCRVCntjpx62XOAR1ESwJYJu2Tpg2dNyFDbeFMIYgu+xzztYQFRh6CoI
      8Ci0QmWm/iSmft0oNkGXLxLhA6fcBpdG9C9rf5dYdSVivPSmaoJ7oS3XK5N8W/+F
      t6u30cZt3cpq7JKW9ZpdhrkVGQSWl1RFi9DHOrDqCGxIZB508y7VOXMwMJBym9ka
      IaJ83KHfqyn8QCX8Vl5SOLS57I/8KKlFUoQA5o0uRMZueN6aS8QTWgZaRV2GxYKv
      Z+xK8bt6fYNw1uzBfRaCaeYMxcxGDRacFTBm8tk4mTZjz/YzGHgmreT5cdP8mudg
      tlqtAgMBAAGjYzBhMA8GA1UdEwEB/wQFMAMBAf8wHQYDVR0OBBYEFH+fyot0ecep
      FwBz42z3RrQXAJrkMB8GA1UdIwQYMBaAFH+fyot0ecepFwBz42z3RrQXAJrkMA4G
      A1UdDwEB/wQEAwIBhjANBgkqhkiG9w0BAQsFAAOCAQEAsovi3DdzB1FxyFgDy6//
      J7sMjcLsuyjelKa7gE8ZuhiEdaatGfBT1/upi2bckhxOEv5B8CkjWYc32iExHl95
      Igh94QNFpoNOr9Mz14t4dJPgcLLkapqdteJwhO9JJBIt0flvaiu4sC12ANQd79Il
      ZXmhA8y85rf+EWG5mLa4Q9yDdflb2PRy3sGQKpHnDzWcJUUTyw9Y7qsqjBeMPjUp
      mEowxB4vi+btU63wSRs+MIM08+n5mE8/ID41P6Z+ZJYIr6tt59ibq5AnUJ2pAts+
      B6cxBGExQ2dEaoZBK5F4AQCouvaOsMGYg+HkzTVfsVCpZO9iWYiHs2RjYm+qmgSO
      Rg==
      -----END CERTIFICATE-----

```

```sh

Thực hiện gắn các node cụ thể với mô hình overcloud dự kiến

```sh
openstack baremetal node set --property capabilities='node:controller-0,boot_option:local' $controller1_ID
openstack baremetal node set --property capabilities='node:controller-1,boot_option:local' $controller2_ID
openstack baremetal node set --property capabilities='node:controller-2,boot_option:local' $controller3_ID
openstack baremetal node set --property capabilities='node:compute-0,boot_option:local' $compute1_ID
openstack baremetal node set --property capabilities='node:compute-1,boot_option:local' $compute2_ID
openstack baremetal node set --property capabilities='node:compute-2,boot_option:local' $compute3_ID
```

Đặt file map hostname của các node `/home/stack/templates/scheduler_hints_env.yaml`

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

Gắn cụ thể các IP cố định tới các node `/home/stack/templates/predictive_ips.yaml`

```sh
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

Câu lệnh boot : 
