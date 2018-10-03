## Cấu hình HAProxy như là một Load-Balancer Layer 4 cho ứng dụng Web Wordpress trên Ubuntu 14.04

## I. Chuẩn bị

 - Mô hình

![HA](/ManhDV/HAProxy/images/1-topo.png)

 - IP Planning 

![HA](/ManhDV/HAProxy/images/2-ip-plan.png)

## II. Cài đặt

### 1. Cài đặt MySQL Server

 - Thực hiện trên máy MySQL server 192.168.100.5
 
 - Cài đặt MySQL server 

```sh
apt-get update
apt-get install mysql-server
```

 - Khởi tạo DB và nhập các thông tin xác thực :
 
```sh
mysql_install_db
mysql_secure_installation
```

 - Cấu hình cho phép truy cập từ client tại file `/etc/mysql/my.cnf`
 
```sh
[mysqld]
bind-address        = 192.168.100.5
```

 - Khởi động lại service mysql
 
```sh
service mysql restart
```

 - Cấu hình tạo DB cho các máy Wordpress
 
```sh
mysql -u root -p
CREATE DATABASE wordpress;
CREATE USER 'wordpressuser'@'localhost' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON wordpress.* TO 'wordpressuser'@'localhost';
CREATE USER 'wordpressuser'@'192.168.100.6' IDENTIFIED BY 'password';
CREATE USER 'wordpressuser'@'192.168.100.7' IDENTIFIED BY 'password';
GRANT SELECT,DELETE,INSERT,UPDATE ON wordpress.* TO 'wordpressuser'@'192.168.100.6';
GRANT SELECT,DELETE,INSERT,UPDATE ON wordpress.* TO 'wordpressuser'@'192.168.100.7';
GRANT ALL PRIVILEGES ON wordpress.* TO 'wordpressuser'@'192.168.100.6';
GRANT ALL PRIVILEGES ON wordpress.* TO 'wordpressuser'@'192.168.100.7';
FLUSH PRIVILEGES;
exit
```

 - Test việc truy cập vào mysql server 
  
  - Tại MySQL server :
  
  ```sh
  mysql -u wordpressuser -p
  show databases;
  ```
  
  - Tại 2 máy wordpress 
  
  ```sh
  apt-get update
  apt-get install mysql-client
  mysql -u wordpressuser -h 192.168.100.5 -p
  ```
  
  Kết quả là truy cập được vào database nghĩa là thành công.
  
### 2. Cài đặt Wordpress1 192.168.100.6

#### 2.1 Setup Web server dùng Nginx

 - Tải Nginx
 
```sh
apt-get install nginx php5-fpm php5-mysql
```

 - Chỉnh sửa PHP

```sh
vi /etc/php5/fpm/php.ini

cgi.fix_pathinfo=0
```

```sh
vi /etc/php5/fpm/pool.d/www.conf
listen = /var/run/php5-fpm.sock
```

 - Restart service
 
```sh
 service php5-fpm restart
```

 - Cấu hình Nginx

```sh
cp /etc/nginx/sites-available/default /etc/nginx/sites-available/example.com
```

```sh
vi /etc/nginx/sites-available/example.com

server {
    listen 80;
    root /var/www/example.com;
    index index.php index.hmtl index.htm;
    server_name example.com;
    location / {
        try_files $uri $uri/ /index.php?q=$uri&$args;
    }
    error_page 404 /404.html;
    error_page 500 502 503 504 /50x.html;
    location = /50x.html {
        root /usr/share/nginx/www;
    }
    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_pass unix:/var/run/php5-fpm.sock;
        fastcgi_index index.php;
        include fastcgi_params;
    }
}
```
 
 - Enable thư mục và remove link từ file mặc định, restart dịch vụ
 
```sh
rm /etc/nginx/sites-enabled/default
ln -s /etc/nginx/sites-available/example.com /etc/nginx/sites-enabled/
service nginx restart
```

#### 2.2. Setup Wordpress

 - Tải file wordpress và cấu hình
 
```sh
cd ~
wget http://wordpress.org/latest.tar.gz
tar xzvf latest.tar.gz
cp ~/wordpress/wp-config-sample.php ~/wordpress/wp-config.php
```

 - Sửa cấu hình trong file `/wordpress/wp-config.php`

```sh
/** The name of the database for WordPress */
define('DB_NAME', 'wordpress');

/** MySQL database username */
define('DB_USER', 'wordpressuser');

/** MySQL database password */
define('DB_PASSWORD', 'password');

/** MySQL hostname */
define('DB_HOST', '192.168.100.5');
```

 - Phân quyền 
 
```sh
mkdir -p /var/www/example.com
cp -r ~/wordpress/* /var/www/example.com
cd /var/www/example.com
chown -R www-data:www-data *
usermod -a -G www-data meditech
chmod -R g+rw /var/www/example.com
```

 - Thực hiện các bước cấu hình trên giao diện web
 
 - Truy cập browser với địa chỉ : http://192.168.100.6
 
 ![HA](/ManhDV/HAProxy/images/3-admin_setup.png)
 
 ![HA](/ManhDV/HAProxy/images/4-admin_login.png)
 
 - Truy cập trang để test : http://192.168.100.6
 
### 3. Cài đặt Wordpress2 192.168.100.7

 - Tạo máy Wordpress2 từ bản Snapshot của máy Wordpress1
 
 - Chỉnh sửa lại IP của máy Wordpress2 trong file : /etc/network/interfaces
 
### 4. Đồng bộ 2 máy Wordpress

**Tại 2 máy Wordpress1 và 2** : 
 
 - Chỉnh sửa file `/etc/hosts` :
 
	```sh
	vi /etc/hosts
	
	192.168.100.6  wordpress-1
	192.168.100.7  wordpress-2
	```
	 
 - Cài đặt GlusterFS và cấu hình Replicated volume
	
**Trên máy Wordpress1**
 
 ```sh
 apt-get install glusterfs-server
 gluster peer probe wordpress-2
 ```
 
**Trên máy Wordpress2**
 
 ```sh
 apt-get install glusterfs-server
 gluster peer probe wordpress-1
 ```
 
**Trên máy Wordpress1 và 2**

 - Tạo thư mục glusterFS để lưu trữ file quản lý :
 
 ```sh
 mkdir /gluster-storage
 ```
 
**Trên máy Wordpress1**

 - Tạo replicating glusterFS volume gọi là volume1, nơi lưu trữ data trong `/gluster-storage` trên cả 2 máy server 
 
 ```sh
 gluster volume create volume1 replica 2 transport tcp wordpress-1:/gluster-storage wordpress-2:/gluster-storage force
 gluster volume start volume1
 ```
 
 - Kết quả hiện ra như sau là OK :
 
 ```sh
 volume start: volume1: success
 ```
 
 - Kiểm tra thông tin glusterFS volume

 ```sh
 gluster volume info
 ```
 
 Sẽ có 2 glusterFS "brick" với mỗi Wordpress server tương ứng
 
 - Sửa file `fstab` để thư mục shared file system sẽ được mount khi boot
		
	```sh
	vi /etc/fstab

	wordpress-1:/volume1   /storage-pool   glusterfs defaults,_netdev 0 0
	```
	
 - Mount GlusterFS volume tới filesystem `/storage_pool`
 
 ```sh
 mkdir /storage-pool
 mount /storage-pool
 ```
 
**Trên máy Wordpress2**

  - Sửa file `fstab` để thư mục shared file system sẽ được mount khi boot
		
	```sh
	vi /etc/fstab

	wordpress-1:/volume1   /storage-pool   glusterfs defaults,_netdev 0 0
	```
	
 - Mount GlusterFS volume tới filesystem `/storage_pool`
 
 ```sh
 mkdir /storage-pool
 mount /storage-pool
 ```
 
 - Chuyển wordpress file sang shared storage
 
**Trên Wordpress1**

```sh
mv /var/www/example.com /storage-pool/
chown www-data:www-data /storage-pool/example.com
```

 - Tạo symbolic link để point Wordpress file trên shared filesystem
 
```sh
ln -s /storage-pool/example.com /var/www/example.com
```

**Trên Wordpress2** 

```sh
rm /var/www/example.com
ln -s /storage-pool/example.com /var/www/example.com
```

## 5. Cài đặt HAProxy

 - Cài đặt HAProxy
 
```sh
apt-get update
apt-get install haproxy
```

 - Cho phép HAProxy khởi động cùng hệ thống :
 ```sh
 vi /etc/default/haproxy
 
 ENABLED=1
 ```
 
 - Cấu hình HAProxy 
 
 ```sh
 cd /etc/haproxy; sudo cp haproxy.cfg haproxy.cfg.orig
 vi haproxy.cfg
 ```
 
```sh
 global
        log /dev/log    local0
        log /dev/log    local1 notice
        chroot /var/lib/haproxy
        user haproxy
        group haproxy
        daemon

defaults
        log     global
        mode    tcp
        option  tcplog
        option  dontlognull
        contimeout 5000
        clitimeout 50000
        srvtimeout 50000
        errorfile 400 /etc/haproxy/errors/400.http
        errorfile 403 /etc/haproxy/errors/403.http
        errorfile 408 /etc/haproxy/errors/408.http
        errorfile 500 /etc/haproxy/errors/500.http
        errorfile 502 /etc/haproxy/errors/502.http
        errorfile 503 /etc/haproxy/errors/503.http
        errorfile 504 /etc/haproxy/errors/504.http
frontend www
   bind 192.168.100.8:80
   default_backend wordpress-backend

 backend wordpress-backend
   balance roundrobin
   mode tcp
   server wordpress-1 192.168.100.6:80 check
   server wordpress-2 192.168.100.7:80 check
```

 - Chỉnh sửa file syslog để ghi log của HAProxy
 
```sh
vi /etc/rsyslog.conf

$ModLoad imudp
$UDPServerRun 514
```

```sh
service rsyslog restart
```

 - Khởi động HAProxy và Nginx
 
```sh
service haproxy restart
```

**Trên Wordpress2**

 - Khởi động lại Nginx
 
```sh
service php5-fpm restart
service nginx restart
```

 - Sửa DB của Wordpress
 
```sh
cd /var/www/example.com
vi wp-config.php

define('WP_SITEURL', 'http://192.168.100.8');
define('WP_HOME', 'http://192.168.100.8');

```

 - Truy cập Wordpress với IP của HAProxy : http://192.168.100.8
