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

TFPT là giao thông truyền file, được dùng để truyền tự động các cấu hình hoặc boot file giữa máy ảo trên môi rường local. TFTP được dùng để download NBP qua network bằng thông tin lấy từ DHCP server.

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