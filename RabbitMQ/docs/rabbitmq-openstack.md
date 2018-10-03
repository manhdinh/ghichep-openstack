# Rabbitmq cluster trong OpenStack

## I. Cài đặt Rabbitmq Cluster trong OpenStack

 - Các node RabbitMQ chạy trên các lớp application và infrastructure. Tầng application được kiểm soát bởi các cấu hình `oslo.messaging` cho các host AMQP. Nếu node AMQP fail, ứng dụng sẽ reconnect tới node tiếp theo được cấu hình với khoảng thời gian reconnect được chỉ định (reconnect interval). Reconnect interval tự thành lập SLA.
 
 - Trên tầng infrastructure, SLA là thời gian mà RabbitMQ cluster tái thiết lập. **Mnesia keeper** node trong RabbitMQ tương đương với node master trong Pacemaker. Khi nó fail, kết quả là toàn bộ AMQP Cluster có khoảng downtime. Thông thường, SLA sẽ không nhiều hơn vài phút. Việc các node khác fail sẽ không gây nên downtime cho cluster.
 
### 1. Cài đặt RabbitMQ

 - Với Debian và Ubuntu : Tham khảo [link](https://www.rabbitmq.com/install-debian.html)
 - Với RPM based : Tham khảo [link](https://www.rabbitmq.com/install-rpm.html)
 
### 2. Cấu hình RabbitMQ cho HA queue

 Các service sau cần cấu hình để có thể làm việc với HA queue
 
	 - Openstack Compute
	 - Openstack Block Storage
	 - Openstack Networking
	 - Telemetry
	 
 - Chú ý rằng trong khi các exchange và binding vẫn tồn tại khi các node đơn bị mất, nhưng các queue và message thì không bởi một queue và nội dung của nó được đặt tại node. Nếu node bị mất thì sẽ bị mất queue tương ứng đặt tại node này,
 
 - Các queue được mirror sang các node khác nhằm cải thiện khả năng hồi phục khi có sự cố.
 
 - Các bước tạo cluster :
 
	- Stop dịch vụ RabbitMQ và copy cookie từ node đầu tiên sang các node khác 
	
	```sh
	scp /var/lib/rabbitmq/.erlang.cookie root@NODE:/var/lib/rabbitmq/.erlang.cookie
	```
	
	- Trên từng node, kiểm tra owner, group và quyền của file `erlang.cookie`
	
	```sh
	chown rabbitmq:rabbitmq /var/lib/rabbitmq/.erlang.cookie
	chmod 400 /var/lib/rabbitmq/.erlang.cookie
	```
	
	- Khởi động RabbitMQ và cho phép boot cùng hệ thống :
	
	```sh
	systemctl enable rabbitmq-server.service
	systemctl start rabbitmq-server.service
	```
	
	- Kiểm tra lại trạng thái cluster 
	
	```sh
	# rabbitmqctl cluster_status
	Cluster status of node rabbit@NODE...
	[{nodes,[{disc,[rabbit@NODE]}]},
	 {running_nodes,[rabbit@NODE]},
	 {partitions,[]}]
	...done.
	```
	
	- Chạy các lệnh sau trên node :
	
	```sh
	# rabbitmqctl stop_app
	Stopping node rabbit@NODE...
	...done.
	# rabbitmqctl join_cluster --ram rabbit@rabbit1
	# rabbitmqctl start_app
	Starting node rabbit@NODE ...
	...done.
	```
	- Để đảm bảo tất cả các queue được mirror sang các node khác, đặt `ha-mode` policy key tới các node bằng việc chạy cmd sau trên một trong các node :
	
	```sh
	rabbitmqctl set_policy ha-all '^(?!amq\.).*' '{"ha-mode": "all"}'
	```
	
### 3. Cấu hình OpenStack để sử dụng RabbitMQ HA queue.

 - Cấu hình `host:port` : 
 
 `rabbit_hosts=rabbit1:5672,rabbit2:5672,rabbit3:5672`
 
 - Retry connecting với RabbitMQ : 
 
 `rabbit_retry_interval=1`
 
 - Sau bao lâu thì kết thúc việc retry connecting tới RabbitMQ :
 
 `rabbit_retry_backoff=2`
 
 - Số lần tối đa retry connecting (mặc định là không giới hạn)
 
 `rabbit_max_retries=0`
 
 - Sử dụng duable queue
 
 `rabbit_durable_queues=true`
 
 - Sử dụng HA queue trong RabbitMQ (x-ha-policy: all)
 
 `rabbit_ha_queues=true`
 
