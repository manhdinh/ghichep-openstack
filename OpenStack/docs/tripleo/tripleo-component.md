# Các thành phần của OpenStack TripleO

Các thành phần của TripleO : 
- Các Shared Library
- Installer
- Node Management
- Deployment & Orchestration
- User interface
- TripleO-validation

## 1. Shared Library

### 1.1. diskimage-builder
diskimage-builder là một tool đóng image. Nó được dùng bởi `openstack overcloud image build`

### 1.2. dib-utils
dib-utils chứa các tool được dùng bởi diskimage-builder

### 1.3. os-*-config
Các project os-*-config là bộ công cụ để cấu hình các máy ảo qua TripleO. Bao gồm :
- os-collect-config
- os-refresh-config
- os-apply-config
- os-net-config

## 2. Installer

### 2.1. Instack

Instack là một script dùng để cài đặt và cấu hình việc triển khai cloud. Các công việc setup của instack bao gồm : 
- Cài đặt các phần mềm yêu cầu để chạy việc triển khai cloud và cấu hình dịch vụ.
- Khai báo Tuskar DB với role mặc định (Tuskar project đã không cần thiết với mô hình TripleO)
- Thiết lập chính sách quản lý flavor trong quá trình cài đặt
- Download hoặc build OS image.
- Upload các image đó lên Glance

### 2.2. instack-undercloud
instack-undercloud alf một dạng install TripleO undercloud dựa vào instack.

## 3. Node Management

### 3.1. Ironic
Ironic project chịu trách nhiệm cung cấp và quản lý các máy baremetal.

Trong môi trường lab, Ironic có thể cung cấp và quản lý máy ảo như là các node baremetal thông qua driver `pxe_ssh`.

### 3.2. ironic inspector (trước kia là ironic-discovered)

ironic inspector là project phụ trách việc thu thập thông tin về các đặc tính của phần cứng cho các node mới tham gia.

### 3.3. VirualBMC
Là một câu lệnh hỗ trợ chuyển đổi các IPMI call tới libvirt call. Được dùng cho mục tiêu test việc cung cấp baremetal trên môi trường ảo hóa.

## 4. Deployment và Orchestration

### 4.1. Heat
Heat là một OpenStack orchestration tool. Nó sẽ đọc các file YAML mô tả các resource để triển khai OpenStack (máy, cấu hình...) và đưa các resource đó vào trạng thái mong muốn, thường alf bằng cách giao tiếp với các thành phần khác (vd như Nova)

### 4.2. heat-templates
heat-templates repository chứa các element image thêm vào các disk image, để chúng có thể sẵn sàng cho việc cấu hình bởi Puppet qua Heat.

### 4.3. tripleo-heat-templates
tripleo-heat-templates mô tả việc triển khai OpenStack trên các file Heat Orchestration Template YAML và Puppet manifest. Các template được triển khai qua Heat.

### 4.4. puppet-*
Module OpenStack Puppet được dùng để cấu hình việc triển khai OpenStack (viết cấu hình, start dịch vụ...). Chúng được dùng thông qua các tripleo-heat-template.

### 4.5. tripleo-puppet-element
tripleo-puppet-element mô tả nội dung của các disk image mà TripleO sử dụng để triển khai OpenStack. Nó giống như các tripleo-puppet

## 5. User interface

### 5.1. python-openstackclient

python-openstackclient là một bộ công cụ các câu lệnh để quản lý cách dịch vụ OpenStack.

### 5.2. python-tripleoclient

python-tripleoclient là bộ công cụ câu lệnh được nhúng vào trong python-openstackclient. Nó cung cấp các function liên quan tới việc cài đặt instack và khởi tạo cấu hình như `node introspection`, build hoặc upload overcloud image...

### 5.3. tripleo-ui
TripleO UI là giao diện Web của Triple O

### 5.4. tripleo-validations

Pre-deployment validation và post-deployment validation cho luồng triển khai của tripleo.


