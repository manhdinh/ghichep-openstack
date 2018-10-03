## 1. Giới thiệu

 -  Message Queueing là gì?

Message Queueing là một cách để trao đổi data giữa các tiến trình, ứng dụng hoặc server

 - RabbitqMQ là gì?
 
RabbitMQ là một phần mềm message-queueing được gọi là một message broker hoặc queue manager. Nói một cách đơn giản : Nó là một phần mềm mà các queue được định rõ, các ứng dụng có thể kết nối tới các queue và vận chuyển message tới đó.

![RABBIT](/ManhDV/RabbitMQ/images/rabbit1.png)

RabbitMQ lưu trữ các message đến khi có ứng dụng kết nối tới và lấy message ra khỏi queue. 

## 2. Kiến trúc RabbitMQ

 - Một giải pháp message broker thường đóng vai trò đứng giữa trong rất nhiều dịch vụ (như trong dịch vụ web bên dưới). Có thể dùng nó để giảm thời gian tải và phân phối bởi các web server khi giao phó hoàn toàn nhiệm vụ vận chuyển message cho bên thứ 3 thực hiện.
 
 - Khi một user nhập thông tin người dùng vào một web-interface, ứng dụng web sẽ put một `PDF processing` - task và toàn bộ thông tin tới một message và message này sẽ được đặt tại một queue được định rõ trong RabbitMQ
 
 ![RABBIT](/ManhDV/RabbitMQ/images/rabbit2.png)
 
 - Kiến trúc cơ bản của một message queue rất đơn giản. Các ứng dụng client được gọi là các "producer" sẽ tạo các message và phân phối chúng tới broker (message queue). Các ứng dụng khác, gọi là "consumer", kết nối tới queue và subcribe các message. Một phần mềm có thể là producer, hoặc consumer hoặc là cả 2. Các message sẽ được lưu trữ tại queue đến khi có consumer nhận chúng.

## 3. Cách thức hoạt động trong RabbitMQ

### 3.1. RabbitMQ Workflow

 - Message queueing cho phép các web-server phản hồi các request một cách nhanh chóng thay vì phải thực hiện các công đoạn tốn tài nguyên. Message queueing thích hợp khi bạn muốn phân tán một message tới nhiều người nhận hoặc cho việc cân bằng tải giữa các worker.
 
 - Một consumer có thể nhận một message của queue và khởi động tiến trình PDF cùng 1 thời điểm producer đẩy một message mới lên queue. Consumer có thể ở một server khác với publisher hoặc chúng có thể nằm trên cùng một server. Request có thể được tạo bởi một ngôn ngữ lập trình và kiểm soát bởi một ngôn ngữ lập trình khác - 2 ứng dụng sẽ giao tiếp với nhau thông qua các message mà chúng gửi tới nhau. Nhờ vậy, 2 ứng dụng sẽ có liên kết chặt chẽ giữa người gửi và người nhận. 
 
  ![RABBIT](/ManhDV/RabbitMQ/images/rabbit3.png)
  
 - Các bước thực hiện :
 
	- Bước 1 : Người dùng gửi ưu cầu tạo một PDF tới ứng dụng web.
	- Bước 2 : Ứng dụng Web (Producer) gửi một message tới RabbitMQ bao gồm data từ request, ví dụ như tên và email
	- Bước 3 : Một "Exchange" chấp nhận các message từ producer và định tuyến chúng tới hàng message queue phù hợp cho việc tạo PDF.
	- Bước 4 : PDF processing worker (consumer) nhận message, thực hiện task tạo PDF. 
	
### 3.2. Exchange

 - Message không được gửi trực tiếp tới queue, thay vào đó, producer gửi các message tới một exchange. Một exchange chịu trách nhiệm cho việc định tuyến bản tin tới các queue khác nhau. Exchange chấp nhận các message từ producer và định tuyến chúng tới các message queue với sự trợ giúp của "binding" và "routing key". Binding là một liên kết giữa một queue và một exchange.
 
 - Message Flow trong RabbitMQ
 
	![RABBIT](/ManhDV/RabbitMQ/images/rabbit4.png) 
  
  
	- Bước 1 : Producer gửi một message tới một exchange. Khi tạo một exchange, bạn phải định rõ ra "type" của chúng.
	- Bước 2 : Exchange nhận được các message và chịu trách nhiệm cho việc định tuyến chúng. Exchange lấy các thuộc tính message khác nhau chẳng hạn như routing key, dựa vào exchange type. 
	- Bước 3 : Các binding được tạo từ exchange tới các queue. Trong trường hợp thấy 2 binding tới 2 queue khác nhau từ một exchange thì việc định tuyến message tới queue dựa vào các thuộc tính của message.
	- Bước 4 : Các message ở trong queue tới khi chúng được kiểm soát bởi một consumer,
	- Bước 5 : Consumer kiểm soát message.
	
### 3.3. Type của Exchange 

Ở phần này sẽ chỉ đề cập đến dạng "direct" exchange. Các thông tin chi tiết về các type khác của exchange, binding key, routing key và cách sử dụng chúng sẽ được đề cập ở phần sau.

  ![RABBIT](/ManhDV/RabbitMQ/images/rabbit6.png) 
  
 - Direct : Một direct exchange sẽ phân phối các message tới các queue dựa vào message routing key. Trong direct exchange, message được định tuyến tới các queue có binding key match với routing key của message. Nếu một queue liên kết với exchange  binding key là `pdfprocess`, một message được gửi tới exchange với routing key là `pdfprocess` sẽ được định tuyến tới queue này.
 - Fanout : Fanout exchane sẽ định tuyến các message tới tất cả các queue mà liên kết tới chúng.
 - Topic : Topic exchange thực hiện wildcard match giữa routing key và routing pattern được chỉ định trong binding.
 - Headers : Headers exchange sử dụng message header attribute cho việc routing 
 
### 3.4 Các định nghĩa trong RabbitMQ

 - Producer : Ứng dụng gửi message
 - Consumer : Ứng dụng nhận message
 - Queue : Bộ đệm lưu trữ message
 - Message : Thông tin được gửi từ producer tới consumer thông qua RabbitMQ
 - Connection : Một kết nối TCP giữa ứng dụng và RabbitMQ Broker
 - Channel : Là một kết nối ảo bên trong một connection. Khi bạn đẩy hoặc nhận message từ một queue, tất cả được thực hiện qua một channel.
 - Exchange : Nhận message từ producer và đẩy chúng tới các queue dựa vào các rule chỉ định bởi exchange type. Để nhận được message, một queue cần liên kết với ít nhất một exchange.
 - Routing key : Giống như một địa chỉ cho message. Exchange sẽ dựa vào routing key để quyế định cách thức định tuyến message tới queue.
 - AMQP : Advanced Message Queuing Protocol, là giao thức được sử dụng bởi RabbitMQ cho việc truyền tin.
 - User : Để kết nối tới RabbitMQ cần username và password. Mọi người dùng đều có thể được gán quyền như read, write và cấu hình đặc quyền bên trong instance. Người dùng có thể được gán quyền vào một virtual host cụ thể.
 - Vhost, virtual host : Một virtual host cung cấp phương thức để tách riêng các ứng dụng sử dụng cùng một RabbitMQ instance. Người dùng khác nhau có thể có quyền với vhost khác nhau.
 
## 4. Giao diện quản lý và giám sát

RabbitMQ cung cấp web UI cho việc quản lý và giám sát RabbitMQ server.

Từ giao diện quản lý có thể sử dụng, tạo, xóa và list các queue. Ngoài ra có thể giám sát độ dài queue, kiểm tra tốc độ message, thay đổi và thêm quyền hạn user.

  ![RABBIT](/ManhDV/RabbitMQ/images/rabbit7.png) 

## 5. Publish và Subcribe messages

RabbitMQ mặc định sử dụng protocol AMQP. Để liên kết với RabbitMQ bạn cần một thư viện hiểu được giao thức mà RabbitMQ sử dụng. Bạn cần dowload client-library cho ngôn ngữ lập trình mà bạn muốn sử dụng cho ứng dụng của mình. Một client-library là một API dùng việc viết các ứng dụng client. Một client-library có một vài phương thức có thể sử dụng, trong trường hợp muốn giao tiếp với RabbitMQ. Các phương thức nên được dùng khi bạn muốn kết nối tới RabbitMQ broker (sử dụng cho các parameter, hostname, port-number đã quy định) hoặc khi bạn muốn khai báo một queue hoặc một exchange. Đây là một lựa chọn thư viện cho hầu hết các ngôn ngữ lập trình.

 - Các bước cần tuân theo khi setup một connection và đẩy / nhận một message :

	-  Đầu tiên, cần setup/create một đối tượng connection. Các thông tin sau cần chỉ rõ : username, password, connection URL, port... Một TCP connection sẽ được setup giữa ứng dụng và RabbitMQ khi phương thức `start` được gọi.
	-  Tiếp đó một channel cần được mở. Một channel cần được tạo bên trong TCP connection. Một connection interface có thể được dùng để mở một channel và khi channel được mở thì nó có thể được dùng để nhận và gửi message.
	-  Khai báo và tạo một queue. Nếu một queue không tồn tại thì trong quá trình khai báo queue sẽ được tạo trước. Tất cả các queue cần được khai báo trước khai có thể sử dụng.
	-  Trong subcriber/consumer : Setup các exchange và tạo liên kết từ queue tới exchange. Tất cả exchange cần được khai báo trước khi được dùng. Một exchange chấp nhận các message từ một producer và định tuyến chúng tới message queue. Để message được định tuyến có thể tới được queue, queue cần được liên kết với exchange.
	-  Trong publisher : Đẩy một message tới một exchange
	-  Đóng channel và connection.