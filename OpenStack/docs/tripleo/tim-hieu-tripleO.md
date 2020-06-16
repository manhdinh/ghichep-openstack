# Tìm hiểu OpenStack TripleO

## 1. TripleO là gì và dùng để làm gì?

TripleO (OpenStack On OpenStack) là một chương trình tập trung vào việc triển khai, nâng cấp và thực thi OpenStack cloud bằng cách sử dụng các cơ sử cloud của OpenStack làm nền tảng. Xây dựng trên nova, neutron và geat để tự động hóa việc quản lý ở quy mô trung tâm dữ liệu.

## 2. Kiến trúc của TripleO

### 2.1. Mô hình UnderCloud và OverCloud
![tripleo](/OpenStack/images/tripleo-00.png)

Với TripleO, bạn có thể tạo một `undercloud` (một môi trường triển khai cloud) chứa các thành phần cần thiết của OpenStack để triển khai và quản lý `overcloud`. Overcloud là một giải pháp được triển khai và có thể sử dụng với bất kỳ mục tiêu nào (production, staging, test...)

![tripleo](/OpenStack/images/tripleo-01.png)

TripleO sử dụng một số thành phần core của OpenStack gồm : Nova, Ironic, Neutron, Heat, Glance và Ceilometer để triển khai OpenStack trên phần cứng baremetal. 

Nova và Ironic được dùng để quản lý máy ảo baremetel bao gồm hạ tầng trên `overcloud`.

Neutron đươc dùng để cung cấp môi trường mạng để triển khai overcloud.

Image của máy ảo được lưu tại Glance

Các dữ liệu của Overcloud được thu thập bởi Ceilometer.

Mô hình vật lý sau mô tả cách triển khai `undercloud` trên server vật lý và `overcloud` được triển khai phân tán trên nhiều server.

![tripleo](/OpenStack/images/tripleo-02.png)

### 2.2. Cảm hứng lấy từ mô hình SpinalStack

Một vài mặt trong workflow của SpinalStack đã được mang sang TripleO, cung cấp các tùy chọn để thực hiện đánh giá phần cứng (`hardware introspection`) trước khi triển khai OpenStack.

Tính năng `hardware introspection` cho phép bạn thu thập dữ liệu về thuộc tính của phần cứng trước khi triển khai, ví dụ như một số cấu hình phàn cứng chuyên dùng để triển khai các node Compute hoặc Storage. Tính năng này cũng cho phép đánh giá hiệu năng. Ví dụ như các phần cứng không đạt yêu cầu về hiệu năng sẽ bị loại bỏ khi triển khai.


## 3. Các lợi ích khi sử dụng TripleO

Sử dụng tổ hợp TripleO của các thành phần OpenStack và API của chúng, cũng như phần hạ tầng để triển khai và thao tác OpenStack sẽ mang lại nhiều lợi ích như : 

 - API của TripleO cũng là API của OpenStack. Việc này giúp việc maintrain dễ dàng hơn. Và người dùng đã quen thuộc với OpenStack cũng sẽ dễ dàng hiểu được TripleO.

  - Sử dụng các thành phần OpenStack cho phép phát triển các tính năng của TripleO nhanh hơn. TripleO tự động kế thừa tất cả các tính năng mới được thêm vào các project core như Glance, Heat...

   - Các update về security và bug fix của OpenStack với các thành phần chính sẽ được thừa kế bởi TripleO.

 - Người dùng có thể tích hợp các script và tiện ích riêng của họ với API của TripleO. Những API đó được cộng đồng OpenStack duy trì và phát triển, không bị đình trệ và kiểm soát bởi một nhà cung cấp duy nhất.

 - Đối với các nhà phát triển, việc tích hợp chặt chẽ với API OpenStack cung cấp một kiến trúc vững chắc, đã trải qua quá trình đánh giá của cộng đồng.

## 4. Overview cách triển khai TripleO

 Các bước triển khai mô hình TripleO

Bước 1. Chuẩn bị môi trường 
- Chuẩn bị môi trường là máy ảo hoặc baremetal
- Cài đặt UnderCloud

Bước 2. Chuẩn bị data UnderCloud
 - Tạo các image để thiết lập OverCloud
 - Đăng ký phần cứng cho các node với undercloud
 - Đánh giá phần cứng
 - Tạo các flavor (node profile)

Bước 3. Kế hoạch triển khai
 - Cấu hình các role cho overcloud
    - Chỉ định flavor (profile cho node tương ứng với các thông số kỹ thuật phần cứng mong muốn)
     Chỉ định image (cung cấp image)
    - Chỉ định số lượng máy ảo triển khai (size role)
- Cấu hình tham số dịch vụ
- Tạo template Heat diễn tả overcloud (tự động tạo từ những cái trên)

Bước 4. Triển khai
- Sử dụng Heat để triển khai template
- Heat sẽ sử dụng nova để xác định và dữ trữ các node thích hợp.
- Nova sử dụng Ironic để khởi động các node và cài đặt đúng image.

Bước 5 : Setup từng node
- Khi mỗi node trên overcloud start, nó sẽ lấy metadata từ các file cấu hình trên Heat Template 
- Heat file được pahan tán trên toàn các node và Heat áp dụng puppet manifest để cấu hình các dịch vụ trên các node.
- Puppet chạy trên nhiều bước, vì vậy sau mỗi bước chúng sẽ được test trigger để kiểm tra quá trình triển khai và cho phép sửa lỗi dễ dàng.

Bước 6. Khởi tạo Overcloud
- Dịch vụ trên mỗi node của overcloud được đăng ký với KeyStone

## 5. Workflow triển khai chi tiết

### 5.1. Chuẩn bị môi trường.

Đầu tiên, bạn cần kiểm tra môi trường triển khai đã sẵn sàng. TripleO có thể triển khai OpenStack trên baremetal cũng như trên môi trường ảo hóa. Bạn cần chắc chắn rằng môi trường cần đáp ứng các yêu cầu tối thiểu cũng như việc setup network đã chính xác.

Bước tiếp theo là cài đặt undercloud. Cho việc phát triển hoặc POC. Bạn thể tham khảo phần [quickstart](https://docs.openstack.org/tripleo-quickstart/latest/index.html).

### 5.2. Chuẩn bị Undercloud Data

#### Images
Trước khi triển khai overcloud, bạn có thể download hoặc build image sẽ được cài đặt trên mỗi node trên over cloud. TripleO sử dụng [diskimage-builder](https://github.com/openstack/diskimage-builder) để tạo ra các `Golden Image`. Tool diskiamge-builder tạo các image cơ bản như Centos 7 và các lớp phần mềm bổ sung thông qua các file script cấu hình (được gọi là các element). Kết quả là file image dạng qcow2 với các phần mềm sẽ được cài đặt nhưng chưa được cấu hình.

Trong khi diskimage-builder repository cung cấp các element đặc thù của OS, các element đặc thù khác của OpenStack, như nova-api, được tìm thấy trong [tripleo-image-elements](https://github.com/openstack/tripleo-image-elements). Bạn có thể thêm các element khác cho image để cung cấp các ứng dụng và dịch vụ đặc thù khác. 

Khi tất cả các image yêu cầu để triển khai trên overcloud đã được xây dựng xong, chúng sẽ được lưu tại Glance trên undercloud.

#### Node

Việc triển khai overcloud yêu cầu phần cứng phù hợp. Nhiệm vụ đầu tiên là đăng ký các phần cứng sẵn sàng với Ironic, tương tự như một hypvervisor của OpenStack cho việc quản lý baremetal server. 

Các user có thể định nghĩa các yếu tố phần cứng (như số CPU, RAM, Disk) thủ công hoặc người dùng có thể để trống các trường và xử lý việc `hareward instropection` của mỗi node sau đó.

Workflow sẽ như hình sau : 

![tripleo](/OpenStack/images/tripleo-03.png)

 - Bước 1 : User thực hiện qua tool comanndline, hoặc qua API, đăng ký thông tin quản lý cho một node với Ironic.
 - Bước 2 : User có thể chỉ thị cho Ironic reboot node.
 - Bước 3 : Với các node mới, chưa được đăng ký, do vậy chưa có hướng dẫn khởi động PXE cụ thể. Trong trường hợp này, việc khởi động quá trình kiểm tra thuộc tính phần cứng được đẩy vào ramdisk.
 - Bước 4 : Quá trình kiểm tra thuộc tính phần cứng được ramdisk thực hiện trên node và thu thập các thông tin thực tế bao gồm : số lượng CPU core, local disk size và số lượng RAM.
 - Bước 5 : Ramdisk cung cấp số liệu thực hiện tới Ironic-inspector API.
 - Bước 6 : Tất cả các số liệu thực tế được thông qua và lưu trữ tại Ironic database.
 - Bước 7 : Theo thực tế của quá trình kiểm tra phần cứng (hardware introspection), có thể bổ sung các role nâng cao tới Ironic thông qua công cụ `ahc-match`.

 #### Flavor

 Khi user tạo các máy ảo trên hệ thông Cloud OpenStack, flavor cần được tạo. Các flavor định rõ số lượng CPU, số lượng RAM, ổ cứng của máy ảo.

 Trên `Undercloud`, máy ảo thường là vật lý hơn là ảo hóa, do đó flavor cũng có 1 chút khác biệt. 
 
 Trong quá trình kiểm tra phần cứng, chỉ các node tương ứng với một flavor nhất định mới có thể đảm nhiệm  role. Điều này đảm bảo rằng các máy ảo lớn với nhiều CPU và RAM sẽ được dùng để chạy Nova trên overcloud, và các máy ảo nhỏ hơn sẽ dùng để chạy các dịch vụ nhẹ hơn, như Keystone chẳng hạn.

 TripleO xử lý các flavor theo 2 mode : 
  - POC mode : Chỉ có 1 flavor duy nhất và bất kỳ hardware nào cũng có thể tương thích với nó. 
  - Scale mode : thích hợp với môi trường triển khai Overcloud lớn hơn và cần mở rộng. Một node chỉ có thể được gán vào 1 role nếu role đó liên kết với flavor mà tương thích với phần cứng của node. Các node không tương thích với flavor nào sẽ không thể sử dụng. 
  
  Mode Scale đảm bảo các loại phần cứng sẽ đi đúng với role. Việc matching có thể sử dụng các tag node một cách thủ công hoặc sử dụng các introspection rule để thực hiện việc tag. Xem thêm tại [Profile Matching](https://docs.openstack.org/project-deploy-guide/tripleo-docs/latest/provisioning/profile_matching.html)

  ### 5.3. Thiết lập kế hoạch triển khai

  Toàn bộ việc triển khai sẽ được xây dựng trên khái niệm `overcloud  role`. Một `role` sẽ mang đến :
  - Image : phần mềm sẽ được cài đặt trên một node.
  - Flavor : Kích thước của node phù hợp với role.
  - Size : Số lượng máy ảo được triển khai.
  - Tổ hợp các Heat template : Cách thức để cấu hình node.

Trong trường hợp với `Compute` role " 
- Image phải chứa toàn bộ các phần mềm để khởi động OS và chạy KVM hypersior và các dịch vụ Nova compute.
- Flavor cần chỉ định máy có đủ CPU và RAM để chạy một số máy ảo đồng thời.
- Heat template đảm nhiệm Nova service sẽ được cấu hình đúng node khi các node khởi động lần đầu tiên.

Những thứ có thể tùy chỉnh trong quá trình lập kế hoạch triển khai là:
- Số lượng node với mỗi role
- Các tham số cấu hình dịch vụ
- Cấu hình network (các option cho NIC)
- Các cấu hình choCeph RBD backend.
- Cách pas các cấu hình thêm (ví dụ các tùy chỉnh trong site)