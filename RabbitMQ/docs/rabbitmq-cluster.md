# Ghi chép các lý thuyết về RabbitMQ-Cluster

## I. RabbitMQ Cluster là gì

 RabbitMQ broker là một nhóm logical gồm một hoặc một vài Erlang node, chạy ứng dụng RabbitMQ và chia sẻ các thông tin về user, virtual host, queue, exchange, binding và run-time parameter. Tập hợp các node này được gọi là một **cluster**.
 
## II. Cơ chế HA trong RabbitMQ Cluster.

 - Việc cài đặt và thực hành RabbitMQ Cluster các bạn có thể tham khảo tại link [1](https://github.com/congto/openstack-HA/blob/master/caidat-OPS-HA/ghiche-rabbitmq.md). 
 - Các câu lệnh trong RabbitMQ, các bạn tham khảo ở link [2](https://github.com/meditechopen/mdt-technical/blob/master/ManhDV/RabbitMQ/docs/Rabbitmq-CLI.md)
 - Việc sử dụng giao diện Web quản lý RabbitMQ, các bạn tham khảo link [3](https://github.com/meditechopen/mdt-technical/blob/master/ManhDV/RabbitMQ/docs/rabbitmq-interface-manager.md)
 
 Trong bài viết này tôi sẽ chỉ đề cập đến các lý thuyết của RabbitMQ Cluster.
 
### 1. Những gì được tái tạo trong Cluster

  - Mặc định, tất cả data/trạng thái cho các hoạt động của RabbitMQ broker được tái tạo tới tất cả các node, ngoài trừ các message queue sẽ được lưu tại chỉ một node, mặc dù chúng ta có thể thấy và tương tác với chúng trên tất cả các node. Để các node khác có bản sao tái tạo của các queue này, cần phải cấu hình HA (Mirrored) queue.
 
  - Các queue được tái tạo trên các node khác được gọi là các **mirrored queue**. Node đầu tiên chứa queue được gọi là node **master**, và các node chứa bản sao tái tạo các mirrored queue được gọi là node **mirror**, trong một số tài liệu các node này còn được gọi là node *slave*.
  
  - Mỗi mirrored queue đều có một master và một hoặc nhiều mirror, để phòng trường hợp nếu như node master biến mất vì bất cứ lý do gì, thì node mirror già nhất, sẽ được đề cử để trở thành node master mới.
  
  - Các message được gởi tới queue sẽ được tái tạo tới tất cả node mirror. Consumer (người muốn lấy message) đã được kết nối tới node master sẽ không 