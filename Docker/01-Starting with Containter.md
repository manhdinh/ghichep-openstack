## Bắt đầu với Containter

Docker cho phép chạy ứng dụng bên trong Linux Containter
	- Chạy 1 containter
	- Làm việc bên trong 1 containter
	- Kiểm tra 1 containter
	
### 1. Mô hình Docker
Docker engine là 1 tiến trình Linux dùng cho việc quản lý các container

![docker](/ManhDV/Docker/images/docker-model.png)

Docker CLI là công cụ cung cấp giao diện dòng lệnh để tương tác với Docker Engine
Kiểm tra version của Docker engine và client :

```sh
docker version
```

Lấy thêm thông tin về Docker engine : 

```sh
docker info
```

**Chú ý** : Nếu bạn cần truy cập đến Docker engine từ xa, bạn cần bật lắng nghe cho engine trên TCP socket port 2375 cho unsecure access và port 2376 cho encryted TLS access.

Các setup mặc định cung cấp kết nối trực tiếp un-encrypted và un-authenticated tới Docker daemon và nên được bảo mật với HTTP hoặc Web proxy đứng trước.

### 2. Chạy một containter

Chạy 1 ứng dụng bởi Docker client

```sh
docker run busybox echo 'Hello Docker'
Hello Docker

docker run busybox whoami
root

docker run busybox route
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
default         172.17.0.1      0.0.0.0         UG    0      0        0 eth0
172.17.0.0      *               255.255.0.0     U     0      0        0 eth0
```

Cú pháp để chạy 1 containter là `docker run <image> <command>` . Với `busybox` là image busybox và `echo, whoami, route` là các câu lệnh chạy bên trong containter.

Khi client chỉ định 1 image, Docker đầu tiên sẽ tìm kiếm image đó tại localhost. Nếu image không tồn tại tại local, thì image sẽ được pulll (tải) từ một image registry công cộng là Docker Hub.

Chạy 1 containter tên là Busybox :

```sh
docker run -it busybox
/ # echo "Hello Docker"
Hello Docker
/ # exit
#
```

Option `-i` dùng để tạo 1 session tương tác và `-t` để mở 1 terminal session.

Chạy 1 containter dưới dạng daemon mode với random port

```sh
docker run -d httpd
```

Chạy 1 containter với port chỉ định 

```sh
docker run -d -p 5000 httpd
```

Chạy 1 containter trong chế độ daemon và ánh xạ port từ host tới port containter

```sh
docker run -d -p 4000:80 httpd

netstat -natp | grep 4000
tcp6       0      0 :::4000                 :::*                    LISTEN      15140/docker-proxy-

curl localhost:4000
<html><body><h1>It works!</h1></body></html>
```

Ảnh xạ tất cả các port trong containter tới các port random của host

```sh
docker run -d -P nginx
docker ps
CONTAINER ID  IMAGE   COMMAND      CREATED     STATUS        PORTS                NAMES
9bfe5c2b1769  nginx   "nginx -g"   4 minutes   Up 4 minutes  0.0.0.0:32769->80/tcp, 0.0.0.0:32768->443/tcp
```

Chỉ định tên cho containter

```sh
docker run -d -p 4000:80 --name webserver httpd
```

Mặc định, 1 container không bị hạn chế về việc sử dụng tài nguyên và có thể dùng cpu, memory và IO của host kernel. Kiểm soát việc cung cấp tài nguyên cho 1 container như sau :

```sh
docker run -d -p 4000:80 --name webserver --memory 400m --cpus 0.5 httpd
```
Câu lệnh trên buộc containter với tên là `webserver` dùng không quá 400MB RAM và 1 nửa CPU của host.

Liệt kê các containter đang chạy :
```sh
docker ps
CONTAINER ID  IMAGE   COMMAND             CREATED         STATUS        PORTS                NAMES
f312f456b54f  httpd   "httpd-foreground"  10 seconds ago  Up 9 seconds  0.0.0.0:80->80/tcp   webserver
```

Liệt kê tất cả các containter :
```sh
docker ps -a
CONTAINER ID  IMAGE     COMMAND             CREATED         STATUS        PORTS                NAMES
f312f456b54f  httpd     "httpd-foreground"  10 seconds ago  Up 9 seconds  0.0.0.0:80->80/tcp   webserver
027758d7be6b  wordpress "/app-entrypoint.sh n"   12 days ago          Exited (1) 6 days ago  wordpress
```

Kiểm soát trạng thái của 1 container
```sh
docker stop|start|restart webserver
```

Remove 1 containter đã bị dừng :
```sh
docker rm webserver
```

Remove 1 containter đang chạy
```sh
docker rm -f webserver
```

Remove tất cả containter 
```sh
docker rm -f `docker ps -aq`
```

Start 1 container và remove nó khi container exit
```sh
docker run -rm -it busyxbox
```

### 3. Làm việc bên trong 1 container
``
Một khi bạn chạy 1 container, bạn có thể sử dụng shell command bên trong contaner. Ví dụ : 
```sh
docker run -it --name centos_bash centos
[root@8de7b5e65c98 /]# nmap 8.8.8.8
bash: nmap: command not found
[root@8de7b5e65c98 /]# yum install -y nmap
Installed:
  nmap.x86_64 2:6.40-7.el7
Complete!
[root@8de7b5e65c98 /]# nmap 8.8.8.8
Nmap scan report for google-public-dns-a.google.com (8.8.8.8)
Host is up (0.0034s latency).
...
[root@8de7b5e65c98 /]# exit
exit
#
```

Mặc dù container không còn chạy khi exit, thì container vẫn tồn tại với package đã được cài đặt sẵn : 
```sh
docker start -ai centos_bash
[root@8de7b5e65c98 /]# nmap 8.8.8.8
Nmap scan report for google-public-dns-a.google.com (8.8.8.8)
Host is up (0.0045s latency).
...
```

### 4. Kiểm tra một container.

Để kiểm tra 1 container, sử dụng `inspect` command :

```sh
docker inspect webserver
```

Kiểm tra log từ 1 container đang chạy với chế độ `tailf` :
```sh
docker logs -f webserver
AH00558: httpd: Could not reliably determine the server's fully qualified domain name
Set the 'ServerName' directive globally to suppress this message
```

Kiểm tra port được dùng bởi container chạy dưới chế độ daemon :
```sh
docker port webserver
80/tcp -> 0.0.0.0:80
```

Thực hiện login vào container và sử dụng các câu lệnh bằng cách dùng `exec` command
```sh
docker exec -it web /bin/bash
root@267ef3db09e5:/usr/local/apache2# pwd
/usr/local/apache2
root@267ef3db09e5:/usr/local/apache2# cat /etc/*release
PRETTY_NAME="Debian GNU/Linux 8 (jessie)"
...
root@267ef3db09e5:/usr/local/apache2# ps -ef
UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  0 17:07 ?        00:00:00 httpd -DFOREGROUND
daemon       6     1  0 17:07 ?        00:00:00 httpd -DFOREGROUND
daemon       7     1  0 17:07 ?        00:00:00 httpd -DFOREGROUND
daemon       8     1  0 17:07 ?        00:00:00 httpd -DFOREGROUND
root        90     0  0 17:09 ?        00:00:00 /bin/bash
root        94    90  0 17:10 ?        00:00:00 ps -ef
...
root@267ef3db09e5:/usr/local/apache2# uname -r
3.10.0-327.18.2.el7.x86_64

root@267ef3db09e5:/usr/local/apache2# free -m
             total       used       free     shared    buffers     cached
Mem:          3791        633       3158          8          2        412
-/+ buffers/cache:        218       3573
Swap:         1637          0       1637
root@267ef3db09e5:/usr/local/apache2# exit
exit
#
```
Container vẫn sử dụng Debian GNU/Linux 8 system. http command có process ID  là 1, /bin/bash có PID là 90 và ps -ef có PID là 94. Process chạy trên host process table không thể nhìn thấy bên trong container. Kernel được dùng chung là : 3.10.0-229.1.2.el7.x86_64. Câu lệnh `free -m` cho thấy số RAM còn trống trên host mặc dù container có thể bị giới hạn RAM.

### 5. Các câu lệnh của Docker

|Command|Mô tả|
|-------|-----|
|attach| Attach tới 1 container đang chạy|
|commit| Tạo 1 image mới từ 1 container |
|cp| Copy files/folders từ 1 filesystem của container tới 1 host path|
|diff| Kiểm tra thay đổi trên 1 filesystem của container|
|events| Lấy event thời gian thực từ server|
|export| Stream nội dung của 1 conainer như là 1 file nén dạng tar|
|info| Show thông tin mở rộng về hệ thống |
|inspect| Trả về thông tin cơ bản trong container|
|kill| Kill 1 container đang chạy |
|load| Load 1 image từ 1 file nén dạng tar|
|logs| Xem log của 1 container|
|port| Tìm kiếm port public được NAT tới port private|
|pause| Pause toàn bộ process bên trong 1 container|
|ps| Liệt kê container|
|restart| Restart 1 container đang chạy|
|rm| Remove 1 hoặc nhiều container|
|run| Chạy 1 command trong 1 container mới|
|start| Start 1 container đang dừng chạy|
|stop| Stop 1 container|
|top| List các process đang chạy của 1 container|
|unpause| unpause 1 container đang pause|
|version| Hiển thị thông tin phiên bản Docker|
|wait| Block tới khi 1 container dừng, sau đó in ra exit code|
