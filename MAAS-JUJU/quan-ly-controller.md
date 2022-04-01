# Ghi chép về MAAS và JUJU

## I. Ghi chép về MAAS

### 1.2 Quản lý Rack

Một Rack controller có thể connect tới nhiều VLAN, tới mỗi network interface khác nhau. Một rack controller có thể chỉ connect tới 1 MAAS instance tại bất kỳ thời điểm nào (phải cùng MAAS major và minor version). Cấu hình này sẽ giúp ích khi quy mô của hệ thống network bắt đầu mở rộng.


### 1.3 Cài đặt rack controller

Để cài đặt và đăng ký 1 rack controller với MAAS :

```sh
sudo snap install maas
sudo maas init rack --maas-url $MAAS_URL --secret $SECRET
```

*$SECRET* được lưu trữ trong `/var/lib/maas/secret` trên API server.

Trên UI, bạn có thể thêm 1 rack controller tại tab `Controller`. Chọn nút `Add rack controller` và chọn hướng dẫn để build model (snap hoặc package). Các câu lệnh sẽ bao gồm sẵn MAAS URL chuẩn và secret để bạn có thể `cut` và `paste` vào câu lệnh.

### 1.4 List rack controller

Bạn cũng có thể list và confirm đăng ký rack controller thông qua CLI. Chọn link trên đầu page để xem hướng dẫn cách xử lý. 

*Chú ý* : Nếu bạn cần multiple rack controller, bạn cần cấu hình High Availability cho chúng

### 1.5 Unregister rack controller

Thông thường, bạn chỉ unregister các rack controller không cần thiết. Trong trường hợp này, bạn cần xóa nó khổi API region server, không có lệnh `unregister` command

Để thực hiện, tại trang `Controllers` của Web UI. Click vào machine mà bạn muốn xóa và chọn `Delete` từ menu. Nếu Controller được dùng cho DHCP HA, thì DHCP HA cần phải disable.


### 1.6. Các mối nguy hiểm tiềm tàng của việc di chuyển rack controller

Việc moving rack controller có thể gây ra 1 số lỗi, khiến bạn rơi vào tình trạng `non-working`, hoặc có thể khiến mất dữ liệu. Các nguy hiểm này được tạo ra bởi 1 cảnh báo và 2 sai lầm tiềm ẩn : 

 - Sử dụng cùng 1 hệ thống làm rack controller và 1 vm host: Việc này có thể gây ra sự tranh chấp tài nguyên và khiến hiệu suất trở nên kém đi. Nếu tài nguyên của hệ thống không đáp ứng đủ cả 2 tác vụ, hệ thống sẽ bị chậm, thậm chí bị đóng băng.
 
 - Moving rack controller từ 1 version MAAS sang version khác : Nếu bạn delete 1 rack controller đang chạy trên version MAAS 2.6, và register rack controller đó sang MAAS 2.9, khả năng mất dữ liệu có thể xảy ra. Nếu chạy cả VM host và rack controller thì khi dịch chuyển sang MAAS khác version, các VM host có thể fail hoặc biến mất.
 
 - Kết nối 1 instance của Rack controller tới 2 instance của MAAS (bất kể phiên bản) : Việc connect 1 single rack controller tới 2 MAAS instance khác nhau có thể dẫn tới các thảm họa không thể báo trước.
 
 ### 1.7 Move rack controller từ MAAS instance
 
 Trên thực tế, không có hành động nào như `move rack controller`, mặc dù bạn có thể delete 1 rack controller từ 1 MAAS và khôi phục lại cùng 1 controller (binary-wise) trên 1 MAAS instance khác. 
 
 Đầu tiên, delete rack controller trên `Controllers` tab, chọn rack controller => `Take action` => `Delete` => `Confirm việc xóa`
 
 Tiếp theo, bạn hãy register 1 rack controller mới (luôn thực hiện từ command).
 
 Với hành động này, giả sử bạn đang sử dụng rack controller đã được cài đặt trước đó, đang chạy từ MAAS instance. Tất cả những gì cần làm là register 1 rack controller mới với `to MAAS` như sau :
 
 ```sh
 sudo maas init rack --maas-url $MAAS_URL_OF_NEW_MAAS --secret $SECRET_FOR_NEW_MAAS

 ```
 
 Trong đó `$SECRET` được lưu tại `/var/snap/maas/common/maas/secret`.
 
 Trên UI, nếu bạn tới tab `Controller` và ấn nút `Add rack controller`, MAAS sẽ đưa bạn 1 câu lệnh hoàn chỉnh, bao gồm URL và secret. Hãy cut và paste string đó để thực hiện lệnh move controller, chú ý nế bạn đang sử dụng mô hình snap hoặc build từ package.
 
 ## II. Quản lý Region

Region controller quản lý việc liên kết với người dùng thông qua WEB UI/API, cũng như việc quản lý Rack controller. MAAS postgres database cũng được quản lý bởi region controller. Thông thường, ở mức region-level sẽ chịu trách nhiệm request controller boot machine, và cung cấp Ubuntu image cần thiết để commission hoặc enlist một machine.

### 1. Setup PostgreSQL for region

Bất kỳ số lượng API server (region controller) nào cũng có thể có mặt miễn là mỗi máy chủ kết nối với cùng 1 PostgreSQL database và cho phép số lượng kết nối cần thiết.

Trên primary database host, sửa file `/etc/postgresql/9.5/main/pg_hba.conf` để cho phép API

```sh
host maasdb maas $SECONDARY_API_SERVER_IP/32 md5
```

Primary database và API server thông thường sẽ nằm tại cùng 1 host.

Áp dụng thay đổi bằng cách restart database :

```sh
sudo systemctl restart postgresql
```

### 2. Thêm 1 region host mới

Trên secondary host, thêm 1 region controller mới bằng cách cài đặt `maas-region-api` : 

```sh
sudo apt install maas-region-api
```

Bạn sẽ cần `/etc/maas/regiond.conf` file từ primary API server. Chúng ta giả sử là có hể scp từ `ubuntu` trên thư mục home bằng cách dùng password authen. Lệnh `local_config_set` sẽ edit file bằng cách trỏ đến host chứa primary PostgreSQL database. MAAS sẽ hợp lý hóa các option cấu hình DNS (bind9) để chúng khớp với những tùy chọn được dùng trong MAAS: 

```sh
sudo systemctl stop maas-regiond
sudo scp ubuntu@$PRIMARY_API_SERVER:regiond.conf /etc/maas/regiond.conf
sudo chown root:maas /etc/maas/regiond.conf
sudo chmod 640 /etc/maas/regiond.conf
sudo maas-region local_config_set --database-host $PRIMARY_PG_SERVER
sudo systemctl restart bind9
sudo systemctl start maas-regiond
```

Kiểm tra các log sau nếu có lỗi xảy ra :

```sh
/var/log/maas/regiond.log
/var/log/maas/maas.log
/var/log/syslog
```

Cải thiện hiệu năng của `region controller`

MAAS Region controller là 1 tổ hợp tiến trình gồm 4 worker chịu trách nhiệm xử lý tất cả các tác vụ bên trong của MAAS. Các worker sẽ xử lý UI, API và giao tiếp nội bộ giữa Region và Rack controller.

Tăng số lượng worker sẽ tăng số lượng kết nối database yêu cầu mỗi 11 worker cộng thêm. Điều này có thể yêu cầu PostgreSQL để tăng số lượng connection được cho phép (tham khảo phần cấu hình HA)

Để tăng số lượng worker, edit file `/etc/maaas/regiond.conf` và `set_worker`. VD :

```sh
[...]
num_workers: 8
```

Nên nhớ rằng thêm quá nhiều worker có thể khiến giảm hiệu năng. Chúng tôi khuyến cáo răng 1 worker/ 1 cpu, tối đa tổng 8 worker