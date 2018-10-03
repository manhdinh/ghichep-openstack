# Mục lục 

 *	[1 Thành phần Cinder](#1)
 *	[2 Workflow của Cinder khi tạo mới volume](#2)
 *	[3 Workflow của Cinder khi Attach Volume](#3)
 *	[4 Workflow của Cinder khi Backup Volume](#4)
 *	[5 Workflow của Cinder khi Restore Volume](#5)

# 1 Thành phần Cinder <a name="1"> </a>

- Chúng ta có 4 quy trình tạo nên Cinder Service :

|Process|mô tả|
|---------|-----|
|Cinder-api|là một ứng dụng WSGI chấp nhận và xác nhận các yêu cầu REST (JSON hoặc XML) từ client và chuyển chúng tới các quy trình CInder khác nếu thích hợp với AMQP|
|Cinder-scheduler|Chương trình lập lịch các định back-end nào sẽ là điểm đến cho một yêu cầu tạo ra volume hoặc chuyển yêu cầu đó. Nó duy trì trạng thái không liên tục cho các back-end (Ví dụ : khả năng sẵn có , khả năng và các thông số kỹ thuật được hỗ trợ) có thể được tận dụng khi đưa ra các quyết định về vị trí . Thuật toán được sử dụng bởi chương trình lên lịch có thể được thay đổi thông qua cấu hình Cinder|
|Cinder-volume|Cinder-volume chấp nhận các yêu cầu từ các quy trình CInder khác đóng vai trò là thùng chứa hoạt động cho các trình điều khiển Cinder. Quá trình này là đa luồng và thường có một luồng thực hiện trên mỗi Cinder back-end giống như định nghĩa trong tập tin cấu hình Cinder.|
|Cinder-backup|Xử lý tương tác với các mục tiêu sao có khả năng sao lưu (Ví dụ như OpenStack Object Storage Service - Swift). Khi một máy client yêu cầu sao lưu volume được tạo ra hoặc quản lý.|


# 2 Workflow của Cinder khi tạo mới volume <a name="2"> </a>

![cinder](/ManhDV/OpenStack/images/cinder-process-diagram.png)

Hình bên trên mô tả quy trình tạo Volume , tiếp theo chúng ta cùng đến với quy trình tạo ra volume mới của Cinder :

![cinder](/ManhDV/OpenStack/images/create-new-volume-diagram.png)

1. Client yêu cầu tạo ra Volume thông qua việc gọi REST API (Client cũng có thể sử dụng tiện ích CLI của python-client)
2. Cinder-api : Quá trình xác nhận hợp lệ yêu cầu thông tin người dùng , một khi được xác nhận một message được gửi lên hàng chờ AMQP để xử lý.
3. Cinder-volume thực hiện quá trình đưa message ra khỏi hàng đợi , gửi thông báo tới cinder-scheduler để báo cáo xác định backend cung cấp volume.
4. Cinder-scheduler thực hiện quá trình báo cáo sẽ đưa thông báo ra khỏi hàng đợi , tạo danh sách các ứng viên dựa trên trạng thái hiện tại và yêu cầu tạo volume theo tiêu chí (kích thước, vùng sẵn có, loại volume (bao gồm cả thông số kỹ thuật bổ sung)).
5. Cinder-volume thực hiện quá trình đọc message phản hồi từ cinder-scheduler từ hàng đợi. Lặp lại qua các danh sách ứng viên bằng các gọi backend driver cho đến khi thành công.
6. NetApp Cinder tạo ra volume được yêu cầu thông qua tương tác với hệ thống lưu trữ con (phụ thuộc vào cấu hình và giao thức).
7. Cinder-volume thực hiện quá trình thu thập dữ liệu và metadata volume và thông tin kết nối để trả lại thông báo đến AMQP.
8. Cinder-api thực hiện quá trình đọc message phản hồi từ hàng đợi và đáp ứng tới client.
9. Client nhận được thông tin bao gồm trạng thái của yêu cầu tạo, Volume UUID, ....


## 3 Workflow của Cinder khi Attach Volume <a name="3"> </a>

![cinder](/ManhDV/OpenStack/images/cinder-process-diagram.png)

1. Client yêu cầu attach volume thông qua Nova REST API (Client có thể sử dụng tiện ích CLI của python-novaclient)
2. Nova-api thực hiện quá trình xác nhận yêu cầu và thông tin người dùng. Một khi đã được xác thực, gọi API Cinder để có được thông tin kết nối cho volume được xác định.
3. Cinder-api thực hiện quá trình xác nhận yêu cầu hợp lệ và thông tin người dùng hợp lệ . Một khi được xác nhận , một message sẽ được gửi đến người quản lý volume thông qua AMQP.
4. Cinder-volume tiến hành đọc message từ hàng đợi , gọi Cinder driver tương ứng với volume được gắn vào.
5. NetApp Cinder driver chuẩn bị Cinder Volume chuẩn bị cho việc attach (các bước cụ thể phụ thuộc vào giao thức lưu trữ được sử dụng).
6. Cinder-volume thưc hiện gửi thông tin phản hồi đến cinder-api thông qua hàng đợi AMQP.
7. Cinder-api thực hiện quá trình đọc message phản hồi từ cinder-volume từ hàng đợi; Truyền thông tin kết nối đến RESTful phản hồi gọi tới NOVA.
8. Nova tạo ra kết nối với bộ lưu trữ thông tin được trả về Cinder.
9. Nova truyền volume device/file tới hypervisor , sau đó gắn volume device/file vào máy ảo client như một block device thực thế hoặc ảo hóa (phụ thuộc vào giao thức lưu trữ).

## 4 Workflow của Cinder khi Backup Volume <a name="4"> </a>

![cinder](/ManhDV/OpenStack/images/cinder_backup_process.png)

1. Client gửi request backup một Client volume với REST API (clint có thể dùng `python-cinderclient` CLI )
2. `cinder-api` process xác thực yêu cầu, thông tin người dùng, và khi thông tin được xác thực, message sẽ được đẩy tới backup manager thông qua AMQP.
3. `cinder-backup` đọc message từ queue, tạo 1 database record cho việc backup và chuyển thông tin từ database cho volume cần bckup
4. `cinder-backup` gọi phương thức `backup-volume` của Cinder volume driver tương ứng với volume cần được backup, đưa bản tin backup và tạo kết nối cho backup service để dùng (NFS, Switf, Ceph...)
5. Cinder voume driver phù hợp được gắn với Cinder volume
6. Volume driver gọi phương thức `backup` cho service backup đã được cấu hình, bàn giao volume attachment
7. Backup service chuyển data và metadata của Cinder volume và meta tới backup repository
8. Backup service update database với một record hoàn chỉnh cho backup này và thông báo thông tin phản hồi tới `cinder-api` process thông qua AQMP
9. `cinder-api` process đọc message phản hồi từ queue và cho qua các kết quả trên RESTful response tới client

## 5 Workflow của Cinder khi Restore Volume <a name="5"> </a>

![cinder](/ManhDV/OpenStack/images/cinder_backup_process_2.png)

1. Client gửi request restore một Client volume với REST API (clint có thể dùng `python-cinderclient` CLI )
2. `cinder-api` process xác thực yêu cầu, thông tin người dùng, và khi thông tin được xác thực, message sẽ được đẩy tới backup manager thông qua AMQP.
3. `cinder-backup` đọc message từ queue, tìm trong database record cho việc backup và một record database volume mới hoặc đã có sẵn trước đó, tùy thuộc và volume có trước đó có được yêu cầu hay không. 
4. `cinder-backup` gọi phương thức `backup-resore` của Cinder volume driver tương ứng với volume cần được backup, đưa bản tin backup và tạo kết nối cho backup service để dùng (NFS, Switf, Ceph...)
5. Cinder voume driver phù hợp được gắn với Cinder volume
6. Volume driver gọi phương thức `restore` cho service backup đã được cấu hình, bàn giao volume attachment
7. Backup service xác định metadata và data backup cho Cinder volume trong backup repo và sử dụng chúng để restore Cinder volume đích tới trạng thái của volume gốc.
8. Backup service thông báo thông tin phản hồi tới `cinder-api` process thông qua AQMP
9. `cinder-api` proces đọc message phải hồi từ `cinder-backup` từ queue và thông báo kết quả trên RESTful response tới client.
