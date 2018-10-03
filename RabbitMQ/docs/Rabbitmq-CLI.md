## Note các câu lệnh trong RabbitMQ

## 1. User

 - Tạo user

```sh
rabbitmqctl add_user user1 password
```

 - List các user

```sh 
rabbitmqctl list_users
```

 - Change password

```sh 
rabbitmqctl change_password user1 newpassword
```

 - Gán quyền admin cho một user
 
```sh 
rabbitmqctl set_user_tags user1 administrator 
```

 - Xóa user
 
```sh
rabbitmqctl delete_user user1
```

## 2. Vhost (Virtual Host)

Khái niệm : Tương tự với khái niệm vhost trong apache. Trong RabbitMQ các tài nguyên sẽ được chia thành các logical group riêng rẽ với nhau. Mỗi Vhost thường dùng cho 1 ứng dụng riêng biệt,

 - Tạo virtual host :
 
```sh
rabbitmqctl add_vhost /vhost1
```
 
 - List vhost
 
```sh
rabbitmqctl list_vhosts
```

 - Xóa vhost

```sh 
rabbitmqctl delete_vhosts /vhost1
```

 - Gán quyền 1 user vào 1 vhost
 
	 - Cú pháp :
	
```sh
rabbitmqctl set_permissions [-p vhost] [user] [permission ⇒ (modify) (write) (read)]
```

	 - Ví dụ : 
	 
```sh
rabbitmqctl set_permissions -p /vhost1 user1 ".*" ".*" ".*" 
```

 - Show quyền hạn của vhost
 
```sh
rabbitmqctl list_permissions -p /vhost1
```

 - Show quyền hạn của một user

```sh
rabbitmqctl list_user_permissions user1
```

 - Xóa quyền của một user   

```sh
rabbitmqctl clear_permissions -p /vhost1 user1 
```

## 3. Bật các plugin : web-gui, rabbitmaqadmin

 - Bật plugin 
 
```
rabbitmq-plugins enable rabbitmq_management
systemctl restart rabbitmq-server
```

 - Sau khi bật plugin, tải công cụ rabbitmqadmin về và sử dụng
 
```sh
wget http://localhost:15672/cli/rabbitmqadmin
chmod a+x rabbitmqadmin
mv rabbitmqadmin /usr/sbin/
```

## 4. Port firewall trong RabbitMQ

RabbitMQ sử dụng port 5672 và port 15672 cho dịch vụ WEB GUI

 - Mở các port sau :
 
```sh
firewall-cmd --add-port=5672/tcp --permanent
firewall-cmd --add-port=15672/tcp --permanent
firewall-cmd --reload
```
















