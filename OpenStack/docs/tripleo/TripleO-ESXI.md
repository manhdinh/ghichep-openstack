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
inspection_iprange = 10.0.13.122,10.0.13.48
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
openstack baremetal node set --property capabilities='profile:control,boot_option:local' $controller1_ID
openstack baremetal node set --property capabilities='profile:control,boot_option:local' $controller2_ID
openstack baremetal node set --property capabilities='profile:control,boot_option:local' $controller3_ID
openstack baremetal node set --property capabilities='profile:compute,boot_option:local' $compute1_ID
openstack baremetal node set --property capabilities='profile:compute,boot_option:local' $compute2_ID
openstack baremetal node set --property capabilities='profile:compute,boot_option:local' $compute3_ID
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
  vlan: 14
  name_lower: storage
  ip_subnet: '10.0.14.0/24'
  allocation_pools: [{'start': '10.0.14.81', 'end': '10.0.14.95'}]
  mtu: 1500
- name: StorageMgmt
  name_lower: storage_mgmt
  vip: true
  vlan: 15
  ip_subnet: '10.0.15.0/24'
  allocation_pools: [{'start': '10.0.15.81', 'end': '10.0.15.95'}]
  mtu: 1500
- name: InternalApi
  name_lower: internal_api
  vip: true
  vlan: 11
  ip_subnet: '10.0.11.0/24'
  allocation_pools: [{'start': '10.0.11.81', 'end': '10.0.11.95'}]
  mtu: 1500
- name: Tenant
  vip: false  # Tenant network does not use VIPs
  name_lower: tenant
  vlan: 12
  ip_subnet: '10.0.12.0/24'
  allocation_pools: [{'start': '10.0.12.81', 'end': '10.0.12.95'}]
  mtu: 1500
- name: External
  vip: true
  name_lower: external
  vlan: 101
  ip_subnet: '10.0.0.0/24'
  allocation_pools: [{'start': '10.0.0.4', 'end': '10.0.0.250'}]
  gateway_ip: '10.0.0.1'
  mtu: 1500
- name: Management
  # Management network is enabled by default for backwards-compatibility, but
  # is not included in any roles by default. Add to role definitions to use.
  enabled: true
  vip: false  # Management network does not use VIPs
  name_lower: management
  vlan: 10
  ip_subnet: '10.0.10.0/24'
  allocation_pools: [{'start': '10.0.10.81', 'end': '10.0.10.95'}]
  gateway_ip: '10.0.10.1'
```

Render toàn bộ các file dạng Jinja2 sang dạng YAML 

```sh
cd /usr/share/openstack-tripleo-heat-templates
./tools/process-templates.py -o ~/openstack-tripleo-heat-templates-rendered
```

### 3.2. Customize network

Thực hiện cấu hình theo dạng multple NIC. Các file cần thực hiện cấu hình như sau : 

 - File network isolaton : `network-isolation.yaml`
 - File network defailt : `network-environment.yaml`
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
~/custom-templates/network.yaml
~/custom-templates/nic-configs/controller-nics.yaml
~/custom-templates/nic-configs/compute-nics.yaml
```

Bước 2 : Thực hiện cấu hình `network.yaml` 

```sh
resource_registry:
  OS::TripleO::Controller::Net::SoftwareConfig: /home/stack/custom-templates/nicconfigs/controller-nics.yaml
  OS::TripleO::Compute::Net::SoftwareConfig: /home/stack/custom-templates/nicconfigs/compute-nics.yaml

parameter_defaults:
  NeutronBridgeMappings: 'datacentre:br-ex,tenant:br-tenant'
  NeutronNetworkType: 'vxlan'
  NeutronTunnelType: 'vxlan'
  NeutronExternalNetworkBridge: "''"

# Internal API used for private OpenStack Traffic
  InternalApiNetCidr: 10.0.11.0/24
  InternalApiAllocationPools: [{'start': '10.0.11.81', 'end': '10.0.11.99'}]
  InternalApiNetworkVlanID: 11

# Tenant Network Traffic - will be used for VXLAN over VLAN
  TenantNetCidr: 10.0.12.0/24
  TenantAllocationPools: [{'start': '10.0.12.81', 'end': '10.0.12.99'}]
  TenantNetworkVlanID: 12

# Public Storage Access - e.g. Nova/Glance <--> Ceph
  StorageNetCidr: 10.0.15.0/24
  StorageAllocationPools: [{'start': '10.0.15.81', 'end': '10.0.15.99'}]
  StorageNetworkVlanID: 14

# Private Storage Access - i.e. Ceph background cluster/replication
  StorageMgmtNetCidr: 10.0.15.0/24
  StorageMgmtAllocationPools: [{'start': '10.0.16.81', 'end': '10.0.16.99'}]
  StorageMgmtNetworkVlanID: 15

# External Networking Access - Public API Access
  ExternalNetCidr: 10.0.17.0/24

# Leave room for floating IPs in the External allocation pool (if required)
ExternalAllocationPools: [{'start': '10.0.17.81', 'end': '10.0.17.100'}]
# Set to the router gateway on the external network
  ExternalInterfaceDefaultRoute: 10.0.17.1

# Gateway router for the provisioning network (or Undercloud IP)
  ControlPlaneDefaultRoute: 10.0.13.80
# The IP address of the EC2 metadata server. Generally the IP of the Undercloud
  EC2MetadataIp: 10.0.13.80
# Define the DNS servers (maximum 2) for the Overcloud nodes
  DnsServers: ["8.8.8.8","8.8.4.4"]
```

Bước 3 : Định nghĩa cấu hình NIC của Server

Tại file `netwowrk.yaml` có sử dụng các registry ` ~/custom-templates/nic-configs/` gồm các cấu hình network của Controller và Compute. 

Tạo các file cấu hình như sau : 

```sh
 mkdir ~/custom-templates/nic-configs
```

Bước 4 : Tạo file cấu hình `~/custom-templates/nic-configs/controller.yaml`

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
      The subnet CIDR of the control plane network.
    type: string
  ControlPlaneDefaultRoute:
    default: ''
    description: The default route of the control plane network.
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
  StorageMgmtIpSubnet:
    default: ''
    description: IP address/subnet on the storage_mgmt network
    type: string
  StorageMgmtNetworkVlanID:
    default: 16
    description: Vlan ID for the storage_mgmt network traffic.
    type: number
  StorageMgmtMtu:
    default: 1500
    description: The maximum transmission unit (MTU) size(in bytes) that is
      guaranteed to pass through the data path of the segments in the
      StorageMgmt network.
    type: number
  StorageMgmtInterfaceRoutes:
    default: []
    description: >
      Routes for the storage_mgmt network traffic.
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
    description: The maximum transmission unit (MTU) size(in bytes) that is
      guaranteed to pass through the data path of the segments in the
      External network.
    type: number
  ExternalInterfaceDefaultRoute:
    default: ''
    description: default route for the external network
    type: string
  ExternalInterfaceRoutes:
    default: []
    description: >
      Routes for the external network traffic.
      JSON route e.g. [{'destination':'10.0.0.0/16', 'nexthop':'10.0.0.1'}]
      Unless the default is changed, the parameter is automatically resolved
      from the subnet host_routes attribute.
    type: json

  DnsServers: # Override this via parameter_defaults
    default: []
    description: >
      DNS servers to use for the Overcloud 
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
            get_file: ../../scripts/run-os-net-config.sh
          params:
            $network_config:
              network_config:
              - type: interface
                name: eth3
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
                name: eth4
                mtu:
                  get_param: StorageMtu
                use_dhcp: false
              - type: vlan
                device: eth4
                mtu:
                  get_param: StorageMtu
                vlan_id:
                  get_param: StorageNetworkVlanID
                addresses:
                - ip_netmask:
                    get_param: StorageIpSubnet
                routes:
                  list_concat_unique:
                    - get_param: StorageInterfaceRoutes
              - type: interface
                name: eth5
                mtu:
                  get_param: StorageMgmtMtu
                use_dhcp: false
              - type: vlan
                device: eth5
                mtu:
                  get_param: StorageMgmtMtu
                vlan_id:
                  get_param: StorageMgmtNetworkVlanID
                addresses:
                - ip_netmask:
                    get_param: StorageMgmtIpSubnet
                routes:
                  list_concat_unique:
                    - get_param: StorageMgmtInterfaceRoutes
              - type: interface
                name: eth1
                mtu:
                  get_param: InternalApiMtu
                use_dhcp: false
              - type: vlan
                device: eth1
                mtu:
                  get_param: InternalApiMtu
                vlan_id:
                  get_param: InternalApiNetworkVlanID
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
                members:
                - type: interface
                  name: eth2
                  mtu:
                    get_param: TenantMtu
                  use_dhcp: false
                  primary: true
                - type: vlan
                  mtu:
                    get_param: TenantMtu
                  vlan_id:
                    get_param: TenantNetworkVlanID
                  addresses:
                  - ip_netmask:
                      get_param: TenantIpSubnet
                  routes:
                    list_concat_unique:
                      - get_param: TenantInterfaceRoutes
              - type: ovs_bridge
                name: bridge_name
                mtu:
                  get_param: ExternalMtu
                dns_servers:
                  get_param: DnsServers
                use_dhcp: false
                members:
                - type: interface
                  name: eth0
                  mtu:
                    get_param: ExternalMtu
                  use_dhcp: false
                  primary: true
                - type: vlan
                  mtu:
                    get_param: ExternalMtu
                  vlan_id:
                    get_param: ExternalNetworkVlanID
                  addresses:
                  - ip_netmask:
                      get_param: ExternalIpSubnet
                  routes:
                    list_concat_unique:
                      - get_param: ExternalInterfaceRoutes
                      - - default: true
                          next_hop:
                            get_param: ExternalInterfaceDefaultRoute
outputs:
  OS::stack_id:
    description: The OsNetConfigImpl resource.
    value:
      get_resource: OsNetConfigImpl
```

Bước 5 : Tạo file `~/custom-templates/nic-configs/compute.yaml`

```sh
heat_template_version: rocky
description: >
  Software Config to drive os-net-config to configure multiple interfaces for the Compute role.
parameters:
  ControlPlaneIp:
    default: ''
    description: IP address/subnet on the ctlplane network
    type: string
  ControlPlaneSubnetCidr:
    default: ''
    description: >
      The subnet CIDR of the control plane network.
    type: string
  ControlPlaneDefaultRoute:
    default: ''
    description: The default route of the control plane network.
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

  DnsServers: # Override this via parameter_defaults
    default: []
    description: >
      DNS servers to use for the Overcloud 
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
            get_file: ../../scripts/run-os-net-config.sh
          params:
            $network_config:
              network_config:
              - type: interface
                name: eth3
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
                name: eth4
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
                name: eth1
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
                  name: eth2
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
                members:
                - type: interface
                  name: nic6
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

File cấu hình `~/custom-templates/nic-configs/ceph.yaml`

```sh
heat_template_version: rocky
description: >
  Software Config to drive os-net-config to configure multiple interfaces for the CephStorage role.
parameters:
  ControlPlaneIp:
    default: ''
    description: IP address/subnet on the ctlplane network
    type: string
  ControlPlaneSubnetCidr:
    default: ''
    description: >
    The subnet CIDR of the control plane network. 
    type: string
  ControlPlaneDefaultRoute:
    default: ''
    description: The default route of the control plane network. 
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
  StorageMgmtIpSubnet:
    default: ''
    description: IP address/subnet on the storage_mgmt network
    type: string
  StorageMgmtNetworkVlanID:
    default: 16
    description: Vlan ID for the storage_mgmt network traffic.
    type: number
  StorageMgmtMtu:
    default: 1500
    description: The maximum transmission unit (MTU) size(in bytes) that is
      guaranteed to pass through the data path of the segments in the
      StorageMgmt network.
    type: number
  StorageMgmtInterfaceRoutes:
    default: []
    description: >
      Routes for the storage_mgmt network traffic.
      JSON route e.g. [{'destination':'10.0.0.0/16', 'nexthop':'10.0.0.1'}]
      Unless the default is changed, the parameter is automatically resolved
      from the subnet host_routes attribute.
    type: json

  DnsServers: # Override this via parameter_defaults
    default: []
    description: >
      DNS servers to use for the Overcloud 
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
            get_file: ../../scripts/run-os-net-config.sh
          params:
            $network_config:
              network_config:
              - type: interface
                name: eth0
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
                name: eth1
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
                name: eth2
                mtu:
                  get_param: StorageMgmtMtu
                use_dhcp: false
                addresses:
                - ip_netmask:
                    get_param: StorageMgmtIpSubnet
                routes:
                  list_concat_unique:
                    - get_param: StorageMgmtInterfaceRoutes
outputs:
  OS::stack_id:
    description: The OsNetConfigImpl resource.
    value:
      get_resource: OsNetConfigImpl
```

Tạo file ` ~/custom-templates/layout.yaml`

```sh
resource_registry:
  OS::TripleO::Controller::Ports::InternalApiPort: /usr/share/openstack-tripleo-heat/templates/network/ports/internal_api_from_pool.yaml
  OS::TripleO::Controller::Ports::TenantPort: /usr/share/openstack-tripleo-heat/templates/network/ports/tenant_from_pool.yaml
  OS::TripleO::Controller::Ports::StoragePort: /usr/share/openstack-tripleo-heat/templates/network/ports/storage_from_pool.yaml
  OS::TripleO::Controller::Ports::StorageMgmtPort: /usr/share/openstack-tripleo-heat/templates/network/ports/storage_mgmt_from_pool.yaml

  OS::TripleO::Compute::Ports::InternalApiPort: /usr/share/openstack-tripleo-heat/templates/network/ports/internal_api_from_pool.yaml
  OS::TripleO::Compute::Ports::TenantPort: /usr/share/openstack-tripleo-heat/templates/network/ports/tenant_from_pool.yaml
  OS::TripleO::Compute::Ports::StoragePort: /usr/share/openstack-tripleo-heat/templates/network/ports/storage_from_pool.yaml
  OS::TripleO::Compute::Ports::StorageMgmtPort: /usr/share/openstack-tripleo-heat/templates/network/ports/storage_mgmt_from_pool.yaml

parameter_defaults:
  NtpServer: 10.0.13.1
  ControllerCount: 3
  ComputeCount: 3
  CephStorageCount: 0

  ControllerSchedulerHints:
  'capabilities:node': 'controller-%index%'
  NovaComputeSchedulerHints:
  'capabilities:node': 'compute-%index%'

  ControllerIPs:
  internal_api:
  - 10.0.11.81
  - 10.0.11.82
  - 10.0.11.83
  tenant:
  - 10.0.12.81
  - 10.0.12.82
  - 10.0.12.83
  storage:
  - 10.0.15.81
  - 10.0.15.82
  - 10.0.15.83
  storage_mgmt:
  - 10.0.16.81
  - 10.0.16.82
  - 10.0.16.83
  ComputeIPs:
  internal_api:
  - 10.0.11.84
  - 10.0.11.85
  - 10.0.11.86
  tenant:
  - 10.0.12.84
  - 10.0.12.85
  - 10.0.12.86
  storage:
  - 10.0.15.84
  - 10.0.15.85
  - 10.0.15.86
  storage_mgmt:
  - 10.0.16.84
  - 10.0.16.85
  - 10.0.16.86


```