## Tài liệu hướng dẫn cấu hình HAProxy, Keepalive, Nginx, MySQL 5.5 và PHP 5.6 trên Centos 6.9

Mô hình hệ thống : 

 - VM 1 : 
	- OS : Centos 6.9
	- IP : 192.168.10.10
	
 - VM 2 : 
	- OS : Centos 6.9
	- IP : 192.168.10.11	
	
 - VIP : 192.168.10.15

### 1. Install MySQL 5.5 Replication

#### 1.1. Thực hiện trên cả 2 vm
 
 - Off iptables 

```sh
service iptables stop
chkconfig iptables off
``
 
 - Tải release và tải gói mysql
```sh
rpm -Uvh https://dl.fedoraproject.org/pub/epel/epel-release-latest-6.noarch.rpm
rpm -Uvh https://mirror.webtatic.com/yum/el6/latest.rpm
yum install mysql55w mysql55w-server -y 
```

#### 1.2. Trên node database master vm1

- Backup file config
```sh
cp -p /etc/my.cnf /etc/my.cnf.orig.`date +%F`
```

 - Chỉnh sửa file config. Thông số **binlog-do-db=mysql** là database admin muốn được replicate.
 
cp /etc/my.cnf /etc/my.cnf.orig
rm -rf /etc/my.cnf
cat << EOF > /etc/my.cnf
[mysqld]
server-id = 1
binlog-do-db=mysql
expire-logs-days=7
relay-log = /var/lib/mysql/mysql-relay-bin
relay-log-index = /var/lib/mysql/mysql-relay-bin.index
log-error = /var/lib/mysql/mysql.err
master-info-file = /var/lib/mysql/mysql-master.info
relay-log-info-file = /var/lib/mysql/mysql-relay-log.info
log-bin = mysql-bin
skip-host-cache
skip-name-resolve
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
user=mysql
symbolic-links=0
[mysqld_safe]
log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid
EOF 



 - Start mysql cùng hệ thống
```sh
service mysqld start
chkconfig mysqld on
```

 - Secure các cấu hình mysql. Đặt pass cho user root mysql
```sh
mysql_secure_installation
```
![stack](/Service/HAProxy-LAMP-Stack/images/stack-00.png)

 - Tạo user/pass cho replication service
```sh
mysql -u root -pCloud2018#
CREATE USER replication@192.168.10.11;
GRANT REPLICATION SLAVE ON *.* TO replication@192.168.10.11 IDENTIFIED BY 'Cloud2018#';
flush privileges;
exit;
```

 - Dump file sql
```sh
mysqldump --all-databases -u root -pCloud2018# --master-data > masterdatabase.sql
```

 - Unlock các table trong database 
```sh
mysql -u root -pCloud2018#
FLUSH TABLES WITH READ LOCK;
show master status;
unlock tables;
```

 - Khi thực hiện câu lệnh : **show master status;** . Cần chú ý lấy thông tin `File` và `Position` để thực hiện về sau.
![stack](/Service/HAProxy-LAMP-Stack/images/stack-01.png)
 
 - Scp file sql sang node Slave
```sh
scp masterdatabase.sql  root@192.168.10.11:/root/
```

#### 1.3. Thực hiện trên node MySQL Slave

 - Backup file cấu hình
```sh
cp -p /etc/my.cnf /etc/my.cnf.orig.`date +%F`
```

cat << EOF > /etc/my.cnf
[mysqld]
server-id = 2     
replicate-do-db=mysql
relay-log = /var/lib/mysql/mysql-relay-bin
relay-log-index = /var/lib/mysql/mysql-relay-bin.index
log-error = /var/lib/mysql/mysql.err
master-info-file = /var/lib/mysql/mysql-master.info
relay-log-info-file = /var/lib/mysql/mysql-relay-log.info
log-bin = mysql-bin
skip-host-cache
skip-name-resolve
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
user=mysql
symbolic-links=0
[mysqld_safe]
log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid
EOF 

 - Start mysql cùng hệ thống
```sh 
service mysqld start
chkconfig mysqld on
```
 - Thực hiện bước secure mysql tương tự node master
```sh
mysql_secure_installation
```

 - Restore lại database MySQL từ node master

```sh
mysql -u root -pCloud2018# < /root/masterdatabase.sql
exit;
```

 - Thực hiện xác định node master. Chú ý thay đổi tham số `MASTER_LOG_FILE` và `MASTER_LOG_POS` đúng với thông tin lấy được trên node master. 
```sh
mysql -u root -pCloud2018#
STOP SLAVE;
CHANGE MASTER TO MASTER_HOST='MASTER_IP',MASTER_USER='replication',MASTER_PASSWORD='Cloud2018#', MASTER_LOG_FILE='mysql-bin.000009', MASTER_LOG_POS=107;
START SLAVE;
SHOW SLAVE STATUS\G;
```

#### 1.4. Kiểm tra MySQL Replication

 - Trên node master. Insert 1 giá trị vào database được replicate
 
```sh
mysql -u root -pCloud2018#
use mysql;
create table sample (c int);
insert into sample (c) values (1);
select * from sample;
```

![stack](/Service/HAProxy-LAMP-Stack/images/stack-01.png)

 - Trên node slave. Lấy thông tin trên database được replicate.
 
```sh
mysql -u root -pCloud2018#
use mysql;
select * from sample;
```

Giá trị lấy ra giống với trên node master chứng tỏ database đã được replicate thành công.


### 2. Cài đặt Nginx và PHP 5.6 ( Thực hiện trên cả 2 máy )

#### 2.1. Cài đặt php version 5.6
 
```sh
yum install epel-release -y 
rpm -Uvh https://mirror.webtatic.com/yum/el6/latest.rpm
yum install php56w php56w-opcache php56w-fpm -y
```

#### 2.2. Cài đặt Nginx

 - Cài đặt service nginx và start service
 
```sh
yum install nginx
service nginx start
chkconfig nginx on
```

- Thay đổi worker process thành 4 trong file  /etc/nginx/nginx.conf
```sh
sed -i 's|worker_processes auto|worker_processes 4|'g /etc/nginx/nginx.conf
```

 - Thay đổi file config default nginx. Chú ý đổi tham số `SERVER_IP` là IP của chính server đó.
 
```sh
cp /etc/nginx/conf.d/default.conf /etc/nginx/conf.d/default.conf.orig
rm -rf /etc/nginx/conf.d/default.conf
cat << EOF > /etc/nginx/conf.d/default.conf
#
# The default server
#
server {
    listen       SERVER_IP:80;
    server_name example.com;

   
    location / {
        root   /usr/share/nginx/html;
        index index.php  index.html index.htm;
    }

    error_page  404              /404.html;
    location = /404.html {
        root   /usr/share/nginx/html;
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }

    # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
    #
    location ~ \.php$ {
        root           /usr/share/nginx/html;
        fastcgi_pass   127.0.0.1:9000;
        fastcgi_index  index.php;
        fastcgi_param  SCRIPT_FILENAME   $document_root$fastcgi_script_name;
        include        fastcgi_params;
    }
}
EOF
```

 - Thay đổi user nginx trong file /etc/php-fpm.d/www.conf
```sh
sed -i 's|user = apache|user = nginx|'g /etc/php-fpm.d/www.conf
sed -i 's|group = apache|group = nginx|'g /etc/php-fpm.d/www.conf
service php-fpm restart
```

 - Thực hiện login vào và kiểm tra service Web nginx của từng server : `http://SERVER_IP`
 
### 3. Cài đặt HAProxy và Keepalived

#### 3.1. Thực hiện trên node MASTER VM1

 - Update sysctl
```sh
echo "net.ipv4.ip_nonlocal_bind=1" >> /etc/sysctl.conf
sysctl -p
```

 - Install keepalived
```sh
yum install keepalived -y
```

 - Cấu hình keepalive. Chú ý thay đổi tham số `VIP` cho phù hợp.
```sh
cp /etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf.orig 
rm -rf /etc/keepalived/keepalived.conf
cat << EOF > /etc/keepalived/keepalived.conf
vrrp_script chk_haproxy {           
        script "killall -0 haproxy"     
        interval 2                      
        weight 2                        
}

vrrp_instance VI_1 {
        interface eth0
        state MASTER
        virtual_router_id 51
        priority 101                    
        virtual_ipaddress {
            192.168.10.15       
        }
        track_script {
            chk_haproxy
        }
}
EOF
```
 - Start service keepalived cùng hệ thống
```sh
chkconfig keepalived on && service keepalived start 
```

 - Install haproxy
```sh
yum install haproxy -y
```
 - Config haproxy
```sh
cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.orig
rm -rf /etc/haproxy/haproxy.cfg

cat << EOF > /etc/haproxy/haproxy.cfg
global
        daemon
        maxconn 256

defaults
        mode http
        timeout connect 5000ms
        timeout client 50000ms
        timeout server 50000ms
        stats enable
        stats uri /stats

listen httpd
    bind 192.168.10.15:80
    balance  roundrobin
    mode  http
    server nginx1 192.168.10.11:80 check
    server nginx2 192.168.10.12:80 check
EOF
```
	
 - Start service haproxy cùng hệ thống
```sh
chkconfig haproxy on && service haproxy restart 
```

### Với các máy ảo tạo trên OPS 

 - Thực hiện tạo VIP với câu lệnh : 
```sh
openstack port create --network $network_id vip
```

 VD IP VIP được được là 192.168.10.15
 
 - Update port-pair, cho phép port của máy ảo OPS nhận được VIP
```sh
neutron port-update $port_vm1_id --allowed_address_pairs list=true type=dict ip_address=192.168.10.15
neutron port-update $port_vm2_id --allowed_address_pairs list=true type=dict ip_address=192.168.10.15

