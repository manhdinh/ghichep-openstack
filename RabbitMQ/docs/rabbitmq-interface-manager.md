# RabbitMQ Management Interface

## I. Giới thiệu

RabbitMQ Management là một giao diện giúp việc giám sát và sử dụng RabbitMQ server từ web-browser dễ dàng hơn. Bạn có thể giám sát tốc độ message và gửi/nhận message bằng tay. Bài viết này cung cấp các góc nhìn khác nhau để sử dụng RabbitMQ Management.

RabbitMQ Managment là một plugin của RabbitMQ, sử dụng trang tĩnh HTML đơn để tạo các queries background tới HTTP API cho RabbitMQ. Thông tin từ giao diện quản lý có thể hữu dụng khi bạn thực hiện debug hoặc bạn muốn có một cái nhìn tổng quan với cả hệ thống. Nếu bạn nhìn thấy số lượng của unacked message trở nên cao, có nghĩa rằng consumer đang trở nên chậm. Nếu bạn cần kiểm tra exchange có đang hoạt động hay không, bạn có thể gửi một test message.

## II. Các định nghĩa 

 - User : User có thể được thêm từ management interface và mọi user có thể được gán quyền như là read, write, cấu hình đặc quyền. Người dùng có thể được gán quyền với các virtual host cụ thể.
 - Vhost, virtual host : Virtual host cung cấp phương thức để cô lập ứng dụng sử dụng cùng một RabbitMQ instance. Người dùng khác nhau có thể có các quyền truye cập tới các vhost khác nhau, queue và exchange có thể được tạo để chúng chỉ tồn tại trong vhost.
 - Cluster : Một cluster chứa tổ hợp các computer được kết nối làm việc cùng nhau. Nếu RabbitMQ instace chứa nhiều hơn một node- nó được gọi là RabbitMQ Cluster. Một cluster là một group các node RabbitMQ.
 - Node : Một node là một computer đơn trong RabbitMQ Cluster
 
## III. Các mục trong giao diện quản lý của RabbitMQ

### 1. Overview

Overview show 2 chart, một cho queue message và một cho message rate. Bạn có thể thay đổi quãng thời gian hiện chart bằng cách ấn vào text `chart: last minute`. Thông tin về tất cả trạng thái của các message khi ấn vào `?`

  ![RABBIT](/ManhDV/RabbitMQ/images/rabbit7.png) 
  
### 2. Queue message

 - Biểu đồ hiển thị tổng số lượng các message đã vào hàng đợi với tất các queue. **Ready** hiển thị số lượng của các message sẵn sàng được phân phối. **Unacked** hiển thị số lượng message mà server đang đợi để được công nhận và xếp vào queue.
 
### 3. Message rate

 - Biểu đồ hiển thị tốc độ kiểm soát các message. **Publish** hiển thị tốc độ message được nhập vào server, **Confirm** hiển thị tốc độ mà server được confirm.
 
### 4. Global Count

 - Tổng số lượng các connection, channel, exchange, queue và consumer cho tất cả các virtual host user truy cập gần đây.
 
### 5. Node

 - Node hiển thị thông tin về các node khác nhau trong RabbitMQ Cluster hoặc thông tin về một node đơn nếu chỉ có 1 node được dùng. Các thông tin hiển thị về server memory, số lượng các tiến trình erlang mỗi node... 
 
  ![RABBIT](/ManhDV/RabbitMQ/images/rabbit8.png)  
  
### 6. Port và context

 - Các port listening cho các protocol khác nhau có thể được tìm thấy ở đây.
 
  ![RABBIT](/ManhDV/RabbitMQ/images/rabbit9.png)  
  
### 7. Import và export

 - Bạn có thể import và export các definition về cấu hình. Khi bạn dowload các definition, bạn nhận được một định dạng JSON cho broker (các setting RabbitMQ). File định dạng này có thể dùng để khôi phục các exchange, queue, virtual host, policy và user. Tính năng này có thể được dùng như 1 dạng backup. Mỗi khi bạn thao đổi cấu hình, cần giữ các cấu hình cũ để đề phòng.
 
  ![RABBIT](/ManhDV/RabbitMQ/images/rabbit10.png)  

### 8. Connection và channel

#### 8.1. Connection

 - Bảng **connection** hiện các kết nối đã được thành lập tới RabbitMQ server. **vhost** hiện thông tin vhos có connection đang hoạt động, và **user** liên kết với connection. **Channel** cung cấp thông tin về số lượng các channel sử dụng connection. **SSL/TLS** chỉ ra kết nối được bảo mật với TLS 
 
  ![RABBIT](/ManhDV/RabbitMQ/images/rabbit11.png)  

  - Nếu bạn click vào một trong các connection, bạn sẽ có overview của một connection cụ thể. Bạn có thể view các channel trong connection và data rate. Bạn có thể thấy client properties và có thể đóng connection.
  
  ![RABBIT](/ManhDV/RabbitMQ/images/rabbit12.png)    
  
  Để biết thêm các thông tin về các thuộc tính liên kết với một connection tham khảo tại [đây](https://www.rabbitmq.com/man/rabbitmqctl.1.man.html#list_connections)
  
#### 8.2 Channel

 - Bảng channel hiện các thông tin về tất cả các channel gaafnd dây. **vhost** hiện thông tin vhost có channel đang hoạt động và usernmae của user đang liên kết với channel. **mode** thông tin code đang dùng của channel, có thể là `confirm` hoặc `transactional`. Khi một channel trong `confirm mode`, cả broker và client tính các message. Broker sau đó sẽ confirm các message như là một cách xử lý chúng. Confirm mode được active khi method **confirm.select** được dùng trong một channel.

  ![RABBIT](/ManhDV/RabbitMQ/images/rabbit13.png)    
  
 - Khi bạn click vào một trong các channel, bạn nhận được thông tin chi tiết của một channel cụ thể. Từ đây có thể thấy được message rate trong số lượng của các logical consumer nhận các message thông qua channel.
 
  ![RABBIT](/ManhDV/RabbitMQ/images/rabbit14.png)    
 
### 9. Exchange

 - Một exchange nhận các message từ producer và đẩy chúng tới các queue. Exchange phải biết chính xác những gì cần phải làm với một message mà nó nhận. Tất cả các exchange có thể được list từ exchange tab. **Virtual host** hiện vhost cho exchange, **type** là exchange-type như `direct`, `topic`, `header`, `fanout`. **Feature** hiện các thông số cho exchnage (vd D-stand cho durable và AD cho auto-delete). Feature và type có thể được chỉ định khi tạo exchange. Trong danh sách có 1 số `amq.*exchange` và exchange mặc định (unamed).

  ![RABBIT](/ManhDV/RabbitMQ/images/rabbit15.png) 
  
Khi chỏ đến exchange name, một trang chi tiết về exchange sẽ hiện ra. Bạn có thể nhìn và tạo liên kết với exchange. Bạn cũng có thể đẩy một message tới exchange hoặc xóa exchange

### 10. Queue

 - Bảng queue hiện các queue cho tất cả hoặc vhost được chọn
 
  ![RABBIT](/ManhDV/RabbitMQ/images/rabbit16.png) 
  
 - Queue có các tham số và các biến phụ thuộc vào việc tạo ra như thế nào. Cột `feature` hiện các tham số thuộc về . Nó có thể là các feature như `Durable queue` (đảm bảo rằng RabbitMQ không bao giờ mất queue), `Message TTL` (hiện thời gian sống của một message được đẩy tới một queue trước khi chúng bị loại bỏ), `Auto expire` (cho biết thời gián sống của một queue khi không dùng trước khi nó tự xóa một cách tự động), `Max length` (cho biết queue có thể chứa được tối đa bao nhiêu message) và `Max length byte` (cho biết tổng tổng số body size của message mà queue có thể chứa được tối đa)

  - Nếu bạn ấn vào bất kỳ một queue từ danh sách các queue, tất cả thông tin về queue được hiển thị như ảnh dưới :
  
  ![RABBIT](/ManhDV/RabbitMQ/images/rabbit17.png)  
  
### 11. Consumer

 - Consumer hiện consumer/channel đã kết nối tới queue
 
  ![RABBIT](/ManhDV/RabbitMQ/images/rabbit18.png) 
  
### 12. Binding
 
 - Một binding có thể được tạo giữa một exchange và một queue. Tất cả các active binding tới queue đều show dưới binding. Bạn có thể tạo một binding mới tới một queue từ đây hoặc unbind một queue từ một exchange.
 
  ![RABBIT](/ManhDV/RabbitMQ/images/rabbit19.png)  
  
### 13. Publish message

 - Bạn có thể đẩy thủ công một message tới queue thông qua `publish message`. Message được publish tới default exchange với queue name như là routing key đã nhận, có nghĩa là message sẽ được gửi tới queue. Có thể đẩy một message từ một exchange từ exchange-view.
 
  ![RABBIT](/ManhDV/RabbitMQ/images/rabbit20.png) 
  
### 14. Get message

 - Có thể kiểm duyệt message một cách thủ công trong queue. `Get message` sẽ lấy message cho bạn và nếu bạn đánh dấu nó như là `requeue`, RabbitMQ đẩy nó trở lại queue theo cùng một thứ tự
 
  ![RABBIT](/ManhDV/RabbitMQ/images/rabbit21.png) 
  
### 15. Delete hoặc Purge queue

 - Một queue có thể bị delete hoặc purge bởi nút `delete`, bạn có thể làm trống queue bằng ấn purge.
 
  ![RABBIT](/ManhDV/RabbitMQ/images/rabbit22.png) 
 
### 16. Admin

 - Từ Admin view có thể thêm user và thay đổi quyền hạn user. Bạn có thể setup vhost, policy, federation và shovels. Thông tin về `shovels` có thể tìm thấy tại [link 1](https://www.rabbitmq.com/shovel.html) và [link 2](https://www.cloudamqp.com/docs/shovel.html). Thông tin về federation có thể tìm thấy tại [link](https://www.cloudamqp.com/blog/2015-03-24-rabbitmq-federation.html)
 
 
  ![RABBIT](/ManhDV/RabbitMQ/images/rabbit23.png)  
  
  ![RABBIT](/ManhDV/RabbitMQ/images/rabbit24.png) 
  
### 17. Ví dụ 

 Ví dụ sau miêu tả cách để tạo một queue `example-queue` và một exchange được gọi là ` example.exchange`
 
 - Thêm 1 queue mới :
 
  ![RABBIT](/ManhDV/RabbitMQ/images/rabbit25.png) 
  
 - Thêm 1 exchange mới :
 
  ![RABBIT](/ManhDV/RabbitMQ/images/rabbit26.png) 

 - Exchange và queue được kết nối bởi 1 binding gọi là `pdfprocess`. Message được đẩy tới exchange với routing `pdfprocess` sẽ kết thúc trong queue 
 
 - Ấn vào một exchange hoặc queue và vào mục `Add binding from this exchange` hoặc `Add binding to this queue`
 
  ![RABBIT](/ManhDV/RabbitMQ/images/rabbit27.png) 
  
 - Đẩy một message tới exchange với routing key `pdfprocess`
 
   ![RABBIT](/ManhDV/RabbitMQ/images/rabbit28.png)
   
 - Queue overview cho ví dụ example-queue khi một message được publish
 
   ![RABBIT](/ManhDV/RabbitMQ/images/rabbit29.png)