# I. Kiến trúc của TripleO

Kiến trúc của TripleO gồm 2 thành phần chính : `UnderCloud` và `OverCloud`.

`UnderCloud` hay `Director` là một hệ thống OpenStack đơn, có nhiệm vụ như một hệ thống quản lý, dùng để tạo các hệ thống Cloud với level production - hay còn gọi là các hệ thống `OverCloud`

![tripleo](/OpenStack/images/tripleo-21.png)

# II. Các thành phần chính

## 1. Ironic

### 1.1. Giới thiệu

Ironic cung cấp các host dạng baremetal cho người dùng cuối thông qua việc API của Compute service.

Ironic sẽ quản lý vòng đời (lifecycle) của các phần cứng baremetal rên OverCloud và có native API riêng để phát hiện các node beremetal. 

Người quản trị phải register các node với Ironic thông qua các driver đặc thù như PXE, IPMI hoặc IP ILO, DELL IDRAC

 Ironic sử dụng thông tin thu thập được trong quá trình `introspection` để match node với các role OpenStack, như Controller, Compute hoặc Storage node. 

 Ví dụ, khi phát hiện ra 1 node với 10 ổ cứng, Ironic sẽ xác định nó là node storage.

 ### Bare Metal được dùng trong trường hợp nào?

 Một số use-case cho việc dùng bare metal (server vật lý) trên cloud : 
  - Cung cấp cụm tính toán hiệu năng cao
  - Một số task tính toán cần dùng thiết bị vật lý
  - Các máy chạy database.
  - Project đơn, phần cứng riêng biệt với các yêu cầu đặc biệt về hiệu năng, bảo mật...

### Một số Key-Technologies cho Bare metal

#### PXE (Preboot Execution Enviroment)

PXE khởi động BIOS của system và NIC lên chương trình khởi động (bootstrap) cho máy thông thông qua network. Quá trình này sẽ load OS tới RAM của máy ảo và sau đó thực hiện bởi CPU.

 #### DHCP (Dynamic Host Configuration Protocol)

 DHCP là giao thức network cung cấp các thông tin như IP cho interface và dịch vụ. 

 Sử dụng PXE, BIOS sử dụng DHCP để lấy địa chỉ IP cho network interface và để xác định vị trí máy chủ đang lưu trữ network bootstrap program (NBP).

 #### NBP (Network Bootstrap Program)

NBP tương đương với GRUB (GRand Unifield Bootloader) hoặc LILO (LInux LOader) - là một chương trình boot trên môi trường hard-drive. NBP sẽ load OS kernel tới RAM để OS có thể được bootstrap thông qua network.

#### TFTP (Trivial File Transfer Protocol)

TFPT là giao thức truyền file, được dùng để truyền tự động các cấu hình hoặc boot file giữa máy ảo trên môi trường local. TFTP được dùng để download NBP qua network bằng thông tin lấy từ DHCP server.

#### IPMI (Intelligent Platform Management Interface)

IPIM là một hệ thống giao diện tiêu chuẩn được sử dụng để quản lý hệ thống máy tính và giám sát hoạt động của chúng qua kết nối mạng.

### 1.2. Kiến trúc và các thành phần

![tripleo](/OpenStack/images/tripleo-22.png)


Ironic bao gồm các thành phần sau : 

 - ironic-api : API RESTful xử lý các yêu cầu ứng dụng bằng cách gửi chúng đến 
 RPC. Có thể được chạy qua WSGI hoặc như một chương trình riêng biệt.
 - ironic-conductor : Add/edit/delete node,  thực hiện power on / off với IPMI hoặc các giao thức khác, và thực hiện provision/deploy/clean các bare metal node. Ironic conductor sử dụng driver để thực hiện các hành động đó
 - ironic-python-agent  : chương trình python chạy tạm thời trên RAM để cung cấp dịch vụ ironic-conductor và ironic-inspector.

 Ngoài ra , Ironic còn cần 1 số dịch vụ như ; 
- Database : Lưu trữ thông tin và trạng thái. Các đơn giản và thông dụng nhất là sử dụng database backend như dịch vụ Compute.
- Messaging (như RabbitMQ) : Được dùng để thực hiện RPC giữa ironic-api và ironic-conductor

### 1.3. Workflow thực hiện việc triển khai Bare Metal

#### Trước khi Deploy

Trước khi hiện việc triển khai instance với baremetal, cần chuẩn bị trước một số bước sau : 
 - Cài đặt và cấu hình các gói liên quan như : tftp-server, ipmi, syslinux... tại nơi ironic-conductor chạy.
 - Nova phải thực hiện cấu hình để sử dụng endpoint của bare metal service và các node Nova compute phải được cấu hình để sử dụng Ironic driver.
 - Tạo các flavor cho phần cứng. Nova cần phải biết boot từ flavor nào.
 - Các image phải được có sẵn trên Glance. Các image thông thường được dùng bao gồm : 
   - bm-deploy-kernel
   - bm-deploy-ramdisk
   - user-image
   - user-image-vmlinuz
   - user-image-initrd

- Phần cứng đã được enroll qua RESTful API service.

#### Quá trình Deploy

![tripleo](/OpenStack/images/tripleo-23.png)

- Bước 1 : Nova API yêu cầu boot instance tới Nova-Scheduler thông qua message queue.
- Bước 2 : Nova scheduler thực hiện filter và tìm kiếm hypervisor thích hợp. Nova scheduler sử dụng các `extra_spec` như `cpu_arch` để match với node vật lý thích hợp.
- Bước 3 : Nova compute manager yêu cầu tài nguyên của hypervisor được chọn.
- Bước 4 : Nova compute manager tạo ra các VIFs (Visual interface) trên Neutron tới request về network interface trong quá trình thực hiện request nova boot. MAC của card (hoặc bond) sẽ được tạo random khi VIF được gắn vào node.
- Bước 5 : Một `spawn task` sẽ được tạo bởi nova compute bao gồm các thông tin như image được boot từ đâu... Quá trình này sẽ gọi tới `driver.spawn` từ virt layer của Nova compute. Trong quá trình này, virt driver sẽ thực hiện các việc sau : 
   - Update tới các ironic node các thông tin như : image deploy, instance UUID, flavor properties.
    - Xử duyệt power của node và các deploy interface bằng cách gọi Ironic API.
    - Thực hiện gắn VIF được tạo ở trên tới node. VIF indentifier được lưu tới ironic port hoặc port-group, đồng thời update VIF MAC match với MAC của port hoặc port-group (port-group có độ ưu tiên cao hơn so với port). 
    - Tạo ra các config drive (nếu được yêu cầu).

 - Bước 6 : Ironic virt driver của Nova đưa ra yêu cầu qua API Ironic tới Ironic Conductor về việc xử lý bare metal node.
 - Bước 7 :   Virtual interface được gắn và Neutron API update DHCP port để thực hiện các option PXE/TFTP. Trong trường hợp sử dụng `neutron` network interface, ironic sẽ tạo các port provisioning riêng biệt trên Neutron. Nếu sử dụng `FLAT` network, các port được tạo bởi Nova cho cả việc provisioning và triển khai network cho instance.

- Bước 8 : Interface boot của node Ironic chuẩn bị cấu hình (i)PXE và cache để deploy kernel và ramdisk.

- Bước 9 : Interface management của Ironic node thực hiện các command để enable nework boot của một node.

- Bước 10 : Interface deploy của Ironic node sẽ lưu tạm thời instance image (nếu dùng iscsi interface), kernel và ramdisk (nếu dùng netboot).

- Bước 11: Interface power của Ironic node đưa ra chỉ thị node cần power-on.

- Bước 12: Boot-node thực hiện triển khai ramdisk.

- Bước 13 : Tùy thuộc vào driver sử dụng, Ironic conductor copy image tới node vật lý (qua ISCSI hoặc URL tạm thời). 

- Bước 14 : Interface của Boot-node đối chiếu cấu hình pxe tới instance image (trong trường hợp local boot, hoặc set boot device tới disk), và yêu cầu ramdisk để soft-power-off node. Nếu soft-power-off bởi ramdisk agent không thành công, bare metal node sẽ power off qua IPMI/BMC call.

- Bước 15 : Interface deploy yêu cầu remove provisioning port nếu nó được tạo và gắn tenant port tới node nếu nó chưa được gắn sẵn. Sau đó node sẽ được power-on.

- Bước 16 : Trạng thái provisioning của bare metal node chuyển sang `active`.

## 2. Heat (Orchestration )

### 2.1. Giới thiệu về Heat

Heat hoạt động như một công cụ điều phối tổ hợp ứng dụng (`application stack`), có nhiệm vụ thực hiện chỉ định các element được dùng cho một ứng dụng  trước khi triển khai lên trên Cloud. Heat sử dụng các template stack, có dạng file `.yaml` và bao gồm các thông tin về resource hạ tầng cho việc triển khai. Các resource là các object trong OpenStack như : compute resource, network configuration, security group, scaling rule và các resource tùy chỉnh.

Heat tạo các resource dựa vào các chuỗi phụ thuộc (`dependency chain`), giám sát chúng để đảm bảo sẵn sàng sử dụng và mở rộng chúng nếu cần thiết. Các template giúp `application stack` trở nên linh động và vẫn đạt được kết quả như cũ  kể cả khi việc triển khai được lặp lại.

![tripleo](/OpenStack/images/tripleo-24.png)

OpenStack Heat API được sử dụng để provision và manaage các resource liên quan đến việc triển khai OverCloud như : 
 - Chỉ định số node để provision với mỗi node role.
 - Các phần mềm để cấu hình cho mỗi node 
 - Troubleshoot trong khi triển khai
 - Tạo thay đổi khi post-deployment.

Ví dụ : Heat template chỉ định các tham số cấu hình của Controller node : 

```sh
NeutronExternalNetworkBridge:
    description: Name of bridge used for external network traffic.
    type: string
    default: 'br-ex'
NeutronBridgeMappings:
    description: >
      The OVS logical->physical bridge mappings to use. See the Neutron
      documentation for details. Defaults to mapping br-ex - the external
      bridge on hosts - to a physical name 'datacentre' which can be used
      to create provider networks (and we use this for the default floating
      network) - if changing this either use different post-install network
      scripts or be sure to keep 'datacentre' as a mapping network name.
    type: string
    default: "datacentre:br-ex"
```

Heat sử dụng các template trên Director (UnderCloud) để tạo OverCloud, bao gồm cả Ironic để thực hiện quản lý nguồn cho các node. Các resource và trạng thái của quá trình triển khai OverCloud với `Heat tool`.

### 2.2. Kiến trúc và các thành phần

![tripleo](/OpenStack/images/tripleo-25.png)

Heat có 4 thành phần chính bao gồm : 
 - *Heat* : CLI để giao tiếp với Heat API 
 - *Heat-API* : cung cấp OpenStack REST API để thực hiện các request và gửi tới Heat-engine.
 - *Heat-API-CFN* : cung cấp AWS-style Query API để tích hợp với AWS CloudFormation để thực hiện các request và gửi tới Heat-engine.
 - *Heat-Engine* : thực hiện các công việc điều phối chạy các template và cung cấp các event ngược trở lại cho API xử lý.

Trước khi tìm hiểu các Heat hoạt động, chúng ta sẽ tổng hợp một số khái niệm cần nắm được trong Heat như sau : 
  - *Resource* : Các object trong OpenStack như : compute resource, network configuration, security group, scaling rule và các resource tùy chỉnh.
  - *Stack* : Tổ hợp các resource
  - *Parameters* : Cho phép người dùng cung cấp `input` tới template trong quá trình triển khai. Ví dụ : nếu bạn muốn nhập tên cho máy ảo, thì tên đó được coi như một parameter và có thể thay đổi trong quá trình triển khai.
- *Template* : File dạng YAML chứa các thông tin về việc triển khai. 
- *Output* : Cung cấp thông tin đầu ra cho user.
- *Nested Stack* : Một template được tham chiếu bởi URL bên trong một template khác. 

### 2.3. Workflow của Heat

![tripleo](/OpenStack/images/tripleo-26.png)

 - Bước 1 : Quá trình điều phối (orchestration) được định nghĩa dưới dạng các template (định dạng file YAML) miêu tả các resource.
 - Bước 2 : Người dùng tạo các `stack` bằng cách chỏ heat-cli tool tới file template và parameter.
- Bước 3 : Heat-CLI giao tiếp với Heat-API
- Bước 4 : Heat-API gửi request tới Heat-Engine thông qua Message Queue.
- Bước 5 : Heat-Engine tiếp tục thực hiện request bằng cách giao tiếp với các OpenStack API và cung cấp output ngược trở lại cho người dùng.

## 3. Heat Template

### 3.1. Cấu trúc của Heat Template

Cấu trúc của Heat Template gồm 3 phần chính : 

 - `Parameter` : Các cài đặt được truyền tới Heat, dùng để tùy chỉnh stack. Các Perameter có thể là các input được user truyền vào hoặc là các giá trị cấu hình mặc định. Các parameter được định nghĩa tại section `paramter` trong template.
 - `Resource` : Chỉ rõ các object trong OpenStack được tạo và cấu hình, như một phần của stack. Các resource được định nghĩa tại section `resources` của template.
- `Output` : Giá trị trả về từ Heat sau khi tạo stack. Người dùng có thể truy cập tới các giá trị này thông qua Heat-API hoặc client tool. Các output được định nghĩa tại section `output` của template.

Ví dụ về một Heat template :

```sh
heat_template_version: 2013-05-23

description: > A very basic Heat template.

parameters:
  key_name:
    type: string
    default: lars
    description: Name of an existing key pair to use for the instance
  flavor:
    type: string
    description: Instance type for the instance to be created
    default: m1.small
  image:
    type: string
    default: cirros
    description: ID or name of the image to use for the instance

resources:
  my_instance:
    type: OS::Nova::Server
    properties:
      name: My Cirros Instance
      image: { get_param: image }
      flavor: { get_param: flavor }
      key_name: { get_param: key_name }

output:
  instance_name:
    description: Get the instance's name
    value: { get_attr: [ my_instance, name ] }
```

Template sử dụng resource type `type: OS::Nova::Server` để tạo máy ảo gọi là `my_instance` với flavor, image và key chỉ định. Stack sẽ trả về giá trị của `instance_name` , gọi là `My Cirros Instance`. 

Heat temple sẽ yêu cầu tham số `heat_template_version`, dùng để định nghĩa version các cú pháp (syntax) và các function. Tìm hiểu thêm thông tin về heat template version tại [đây](https://docs.openstack.org/heat/latest/template_guide/hot_spec.html#heat-template-version).


### 3.2. Environment File

Environment file là một dạng template đặc biệt cung cấp việc tùy chỉnh cho Heat template . Chúng bao gồm 3 key part chính : 
 - `Resource Registry` : định nghĩa các `resource name` tùy chỉnh, link tới các Heat template. Resource registry cung cấp phương pháp tạo ra các resource tùy chỉnh (những resource không tồn tại trong bộ resource core). Các resource registry được định nghĩa tại section `resource_registry` của template.

 - `Parameters` : đây là các cài đặt chung được áp dụng ở top-level của template parameter. Ví dụ, nếu bạn có 1 template để triển khai `nested stack` là `resource registry mapping`, `parameter` sẽ chỉ áp dụng tới top-level của template và không áp dụng tới các `nested resource`. Các parameter được định nghĩa tại section `paramter` trong environment file.

- `Parameter Defaults` : parameter chỉnh sửa giá trị mặc định cho các parameter trên mọi template. Cũng với ví dụ trên, nếu bạn có 1 template để triển khai `nested stack` là `resource registry mapping`, `default parameter` sẽ áp dụng tới toàn bộ tempate. Các default parameter được định nghĩa tại section `default_paramter` trong environment file.

Ví dụ về một environment file cơ bản : 

```sh
resource_registry:
  OS::Nova::Server::MyServer: myserver.yaml

parameter_defaults:
  NetworkName: my_network

parameters:
  MyIP: 192.168.0.1
```

Ví dụ, environment file này (my_env.yaml) có thể được bao gồm khi tạo 1 stack từ 1 Heat template là my_template.yaml. File `my_env.yaml` sẽ tạo một resource type mới gọi là `OS::Nova::Server::MyServer`. File `myserver.yaml` sẽ cung cấp triển khai cho tài liệu này và sẽ ghi đè lên bất cứ loại resource type nào được tích hợp.

Parameter `My IP` sẽ chỉ áp dụng cho Heat template chính, được triển khai cùng với environment file. Tróng ví dụ trên, nó sẽ chỉ áp dụng cho các parameter trong `my_template.yaml`. 

`NetworkName` áp dụng cho cả Heat template chính (my_template.yaml) và các template liên kết với các resource bao hàm trong template chính, như là `OS::Nova::Server::MyServer` resource và `myserver.yaml` template. 

### 3.3. Default Director Template 

UnderCloud sử dụng các Heat template nâng cao để tạo `OverCloud`. Tổ hợp các template này có ở trên [tripleo-heat-templates](https://github.com/openstack/tripleo-heat-templates) repo trên Github. Để clone tổ hợp template này, thực hiện câu lệnh sau : 

```sh
git clone https://github.com/openstack/tripleo-heat-templates.git
```

Có rất nhiều Heat template và environment file có trong tổ hợp này. Tuy nhiên, các file chính và quan trọng gồm : 

#### Các file chính : 

 - `overcloud.j2.yaml` 

File template chính được dùng để tạo môi trường cho OverCloud. File sử dụng cú pháp `Jinja2` để lặp một số section nhất định trong template để tạo các role tùy chỉnh. Định dạng Jinja2 sẽ được hoàn trả lại thành YAML trong quá trình triển khai OverCloud.

- `overcloud-resource-registry-puppet.j2.yaml` 

File environment chính để tạo môi trường cho OverCloud. File cung cấp tổ hợp các cấu hình cho Puppet module được lưu trữ OverCloud image. Sau khi UnderCloud ghi OverCloud image xuống mỗi node, Heat khởi động cấu hình Puppet cho mỗi node bằng các resource đã được đăng ký trong file environment này. File cũng sử dụng định dang Jinja2 và sẽ được hoàn trả lại thành YAML trong quá trình triển khai OverCloud.


- `roles_data.yaml`

File định nghĩa role trên OverCloud và map các service tới mỗi role.

- `network_data.yaml`

File định nghĩa network trong OverCloud như : subnet, allocation pool, và trạng thái VIP. Thông thường  `network_data` file sẽ chứa các network mặc định sau : External, Internal API, Storage, Storage Management, Tenant và Management. Bạn có thể tạo file `network_data` tùy chỉnh và thêm vào câu lệnh `openstack overcloud deploy` với option `-n`.

- `plan-environment.yaml`

File định nghĩa các metadata cho OverCloud plan. File bao gồm ` plan name` , `main template` và các `file environment` để áp dụng vào OverCloud.

- `capabilities-map.yaml` 

Ánh xạ của các `environment file` cho OverCloud plan. File được dùng để mô tả và cho phép các `environment file` thông qua WEB Gui của UnderCloud. Các environment file tùy chỉnh không được định nghĩa trong file `capabilities-map.yaml`  nhưng được phát hiện trong `environment` directory trên OverCloud plan sẽ được liệt kê tại subtab `Other` của `2 Specify Deployment Configuration > Overall Settings` trên Web GUI.

#### Các thư mục chính : 

- `environments`

Chứa các file environment bổ sung mà bạn có thể sử dụng khi tạo OverCloud. Các file environment này cho phép các function bổ sung cho môi trường OpenStack.

Ví dụ, thư mục chứa một file environment để cho phép Cinder NetAPP Backend storage (cinder-netapp-config.yaml). Bất kỳ file môi trường nào được phát hiện trong thư mục này mà không được định nghĩa trong file `capabilities-map.yaml` sẽ được liệt kê tại subtab `Other` của `2 Specify Deployment Configuration > Overall Settings` trên Web GUI.

- `network`

Tổ hợp các Heat template hỗ trợ tạo các network và port riêng biệt.

- `puppet`

Các template chủ yếu được điều khiển bởi cấu hình với Puppet. Environment file `overcloud-resource-registry-puppet.j2.yaml` sử dụng file trong thư mục này để thực hiện cấu hình Puppet cho ứng dụng trên mỗi jode.

- `puppet/services` 

Thư mục chứa các Heat template cho mỗi dịch vụ trong kiến trúc dịch vụ tổng hợp.

- `extraconfig`

Các template dùng để cho phép các function bổ sung hoạt động.

- `firstboot`

Cung cấp các script `first_boot` mà UnderCloud sử dụng khi tạo các node lúc ban đầu.

### 3.4. Tùy chỉnh cấu hình First Boot.

