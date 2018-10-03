## Làm việc với image

Khi client chỉ định 1 image, Docker đầu tiên sẽ tìm kiếm image đó trong localhost. Nếu image đó không có trong local, image sẽ được pull từ public image registry Docker Hub. Docker Hub quản lý thông tin về user account, image và public namespace, lưu trữ image bạn tạo, tìm kiếm image hoặc publish image.

### 1. Bắt đầu làm việc với image

Tìm kiếm 1 image từ Docker Hub
```sh
# docker search mysql
NAME                       DESCRIPTION                                     STARS     OFFICIAL   AUTOMATED
mysql                      MySQL is a widely used, open-source relati...   2353      [OK]
mysql/mysql-server         Optimized MySQL Server Docker images. Crea...   144                  [OK]
centurylink/mysql          Image containing mysql. Optimized to be li...   45                   [OK]
...
```

Pull 1 image từ Docker Hub
```sh
# docker pull mysql/mysql-server
Using default tag: latest
...
Status: Downloaded newer image for mysql/mysql-server:latest
```

Liệt kê local images
```sh
# docker images
REPOSITORY           TAG                 IMAGE ID            CREATED             SIZE
httpd                latest              bf8f39bc3b6b        47 hours ago        194.5 MB
centos               latest              8596123a638e        5 days ago          196.7 MB
ubuntu               latest              c5f1cf30c96b        2 weeks ago         120.7 MB
mysql/mysql-server   latest              de3969c3af1c        4 weeks ago         296.7 MB
busybox              latest              47bcc53f74dc        9 weeks ago         1.113 MB
```

Start một container từ local image
```sh
docker run -it centos
```

Remove local image :
```sh
# docker rmi mysql/mysql-server
Untagged: mysql/mysql-server:latest
Deleted: sha256:de3969c3af1c0d7538ce052695e64d9afc4e777569bd5ca61b0440da14fbfd4a
...
```

### 2. Tạo 1 image từ 1 container
Chúng ta sẽ thử tạo 1 image mới từ 1 image đã có sẵn đi kèm 1 số package. Với ví dụ sau là Apache Web server.
Pull base Centos image từ Docker Hub.
```sh
docker pull centos:lastest
```

Start container với chế độ tương tác và cài đặt Apache Web Server 
```sh
# docker run -it --name centos_with_httpd centos
[root@3b929be30cb2 /]# yum update -y; yum install -y httpd; yum clean all
[root@3b929be30cb2 /]# systemctl enable httpd
[root@3b929be30cb2 /]# echo ServerName apache.example.com >> /etc/httpd/conf/httpd.conf
[root@3b929be30cb2 /]# echo Hello Docker > /var/www/html/index.html
[root@3b929be30cb2 /]# exit
```

Commit image đã được thay đổi thành 1 local image mới :
```sh
# docker commit -m "CentOS base image with Apache Web Server installed" centos_with_httpd myhttpd
# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
myhttpd             latest              fb6b21fd89c3        39 seconds ago      246.8 MB
centos              latest              8596123a638e        5 days ago          196.7 MB
```

Chạy 1 container từ image mới vừa được tạo :
```sh
# docker run -d -p 80:80 --name myweb myhttpd /usr/sbin/httpd -DFOREGROUND
```

Attach bash và kiểm tra các process đang chạy :
```sh
# docker exec -it myweb /bin/bash
[root@58076c84093f /]# ps -ef
UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  0 18:14 ?        00:00:00 /usr/sbin/httpd -DFOREGROUND
apache       5     1  0 18:14 ?        00:00:00 /usr/sbin/httpd -DFOREGROUND
apache       6     1  0 18:14 ?        00:00:00 /usr/sbin/httpd -DFOREGROUND
apache       7     1  0 18:14 ?        00:00:00 /usr/sbin/httpd -DFOREGROUND
apache       8     1  0 18:14 ?        00:00:00 /usr/sbin/httpd -DFOREGROUND
apache       9     1  0 18:14 ?        00:00:00 /usr/sbin/httpd -DFOREGROUND
root        10     0  0 18:19 ?        00:00:00 /bin/bash
root        21    10  0 18:19 ?        00:00:00 ps -ef
```

Bạn có thể thấy rằng image mới đã hoạt động, đảy chúng tới public domain bằng việc push image tới Docker Hub (giả sử bạn đã có 1 tài khoản trên Docker Hub).

Tag 1 image mới với namespace liên quan đến account của bạn :
```sh
# docker tag myhttpd kalise/myhttpd:latest
# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
kalise/myhttpd      latest              b48380ba8a9b        26 minutes ago      246.8 MB
myhttpd             latest              b48380ba8a9b        26 minutes ago      246.8 MB
centos              latest              8596123a638e        5 days ago          196.7 MB
```

Login vào Docker Hub và push image
```sh
# docker login --username=kalise --password=********
Login Succeeded
# docker push kalise/myhttpd:latest
The push refers to a repository [docker.io/kalise/myhttpd]
138fc78301b9: Pushed
5f70bf18a086: Mounted from kalise/httpd
```

### 3. Tạo 1 file image từ Docker file
Việc dựng 1 container image từ Dockerfile file là một phương pháp tốt hơn để tạo các container với định dạng tốt hơn, so với việc chỉnh sửa các container đang chạy và tiến hành commit chúng thành image. Ví dụ sau đây thực hiện việc tạo 1 Dockerfile file để tạo image Apache Web Server riêng từ image Centos base. Các bước thực hiện :

 - 1. Chọn 1 base image
 - 2. Cài đặt các packge cần thiết cho Apache Web server
 - 3. Map server port tới port chỉ định trên host
 - 4. Chạy Web server
	
Tạo 1 thư mục project và edit text file gọi là Dockerfile :
```sh
# mkdir -p httpd-project
# cd httpd-project
# touch Dockerfile
```

Dockerfile sẽ có dạng như sau : 
```sh
# My HTTP Docker image
# Version 1

# Pull the CentOS 7.2 image from the Docker Hub registry
FROM centos:7

MAINTAINER Tom Cat
USER root

# Update packages list and install some useful tools
RUN yum update -y

# Add the httpd package
RUN yum install -y httpd

# Clean the yum cache
RUN yum clean all

# Set the Web Server name
RUN echo ServerName apache.example.com >> /etc/httpd/conf/httpd.conf

# Create an index.html file
RUN echo Your Web server is successful > /var/www/html/index.html
```

 - `FROM` statement đặt base image cho các chỉ dẫn tiếp sau. Một Dockerfile hợp lệ phải có `FROM` như là chỉ dẫn ban đầu. Image có thể là bất kỳ image hợp lệ nào từ public hoặc private repo. Trong trường hợp này chúng ta bắt đầu từ Centos image như là base image cho Web Server.

 - `MANITAINER` statement đại diện cho trường `Author` của image được tạo.
 - `LABEL` statement ghi nhãn cho image với một chuỗi text do người dùng đặt ra.
 - `USSER` statement đặt user name hoặc UUID để dùng khi chạy 1 image.
 - `RUN` statement xử lý bất kỳ câu lệnh nào trong layer mới trên top của image và commit kết quả. Kết quả sẽ được dùng tại bước tiếp theo của Dockerfile. Để tránh 1 layer mới được thêm vào mỗi RUN statement, multiple RUN statement có thể được tổ hợp trong 1 single instruction. Để khiến Dockerfile dễ hiểu và tiện lợi hơn, có thể đặt các câu lệnh sau dấu \ . Việc tóm tắt Dockerfile có dạng như sau : 
 
```sh
# My HTTP Docker image: version 1.1

# Pull the CentOS 7.2 image from the Docker Hub registry
FROM centos:7

MAINTAINER Tom Cat tom.cat@warnerbros.com
LABEL Version 1.1
USER root

# Update packages list and install the httpd package
RUN yum update -y && yum install -y httpd && yum clean all

# Set the Web Server name and create an index.html file
RUN \
echo ServerName apache.example.com >> /etc/httpd/conf/httpd.conf && \
echo Your Web server is successful > /var/www/html/index.html
```

Để start các container dưới chế độ daemon, Docker sử dụng command `EXPOSE`. Statement này sẽ đặt container lắng nghe trên 1 port network của host. `CMD` statement được dùng để chạy ứng dụng được chứa bởi image, cùng với bất kỳ tham số cần tìm.  Dạng mới của Dockerfile có dạng như sau : 
```sh
# My HTTP Docker image: version 1.2

# Pull the CentOS 7.2 image from the Docker Hub registry
FROM centos:7

MAINTAINER Tom Cat tom.cat@warnerbros.com
LABEL Version 1.2
USER root
EXPOSE 80

# Update packages list and install the httpd package
RUN yum update -y && yum install -y httpd && yum clean all

# Set the Web Server name and create an index.html file
RUN \
echo ServerName apache.example.com >> /etc/httpd/conf/httpd.conf && \
echo Your Web server is successful > /var/www/html/index.html

# Start the Apache web server application at runtime
CMD ["/usr/sbin/apachectl", "-DFOREGROUND"]
```

Docker dựng 1 image từ 1 Dockerfile với ngữ cảnh nhất định. Ngữ cảnh xây dựng gồm có file đặt tại PATH hoặc URL. `PATH` là 1 thư mục filesystem trên host. `URL` là 1 location của 1 GIT repo. `COPY` statement copy các file mới hoặc thư mục từ source PATH location và thêm chúng vào filesystem của container đích. Các file resource có thể được chỉ định nhưng chúng phải liên quan tới thư mục gốc.
Ví dụ : bạn muốn đặt 1 file mặc định là index.html page của Webserver tới 1 page index.html tùy chỉnh. Đặt page tùy chỉnh như sau :
```sh
# cd /root/httpd-centos
# vi index.html
<!DOCTYPE html>
<html>
<body><h1>Hello Docker!</h1></body>
</html>
```

Thay đổi Dockerfile như sau :
```sh
# My HTTP Docker image: version 1.3

# Pull the CentOS 7.2 image from the Docker Hub registry
FROM centos:7

MAINTAINER Tom Cat tom.cat@warnerbros.com
LABEL Version 1.3
USER root
EXPOSE 80

# Update packages list and install the httpd package
RUN yum update -y && yum install -y httpd && yum clean all

# Set the Web Server name
RUN echo ServerName apache.example.com >> /etc/httpd/conf/httpd.conf

# Overwrite the default index.html page
COPY index.html /var/www/html/index.html

# Start the Apache web server application at runtime
CMD ["/usr/sbin/apachectl", "-DFOREGROUND"]
```

Thay đổi build context và build image
```sh
# cd /root/httpd-centos
# ls -l
total 8
-rw-r--r-- 1 root root 582 May 24 15:42 Dockerfile
-rw-r--r-- 1 root root  67 May 24 16:09 index.html
# docker build -t myapache:centos .
# docker run -d -p 80:80 --name web myapache:centos
# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
myapache            latest              b5cfdf80a528        8 minutes ago       246.8 MB
centos              7                   8596123a638e        6 days ago          196.7 MB
```

Start container từ image đó và kiểm tra hoạt động :
```sh
# docker run -d -p 80:80 --name web myapache
# curl localhost
<!DOCTYPE html>
<html>
<body><h1>Hello Docker!</h1></body>
</html>
```
**Chú ý**  :
	- Tránh cài đặt các package không cần thiết
	- Chỉ chạy 1 process mỗi container
	- Giảm thiếu tối đa số lượng layer
	
Để xây dựng 1 image từ Dockerfile file, dùng các `build option` và chỉ ra đường dẫn của Dockerfile file. Trong trường hợp này, Dockerfile nằm tại current directory
```sh
# docker build -t myapache:centos .
```

Kiểm tra image được tạo với tên là `myapache` và tag là `centos`.
```sh
# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
myapache            centos              63d1feab2644        46 seconds ago      224.1 MB
centos              7                   8596123a638e        6 days ago          196.7 MB
```

Start 1 container từ image đó và kiểm tra hoạt động
```sh
# docker run -d -p 80:80 --name myweb myapache:centos /usr/sbin/httpd -DFOREGROUND
# curl localhost:80
Your Web server is successful
```

Tag image 
```sh
# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
myapache            centos              f297cdf1a74a        53 minutes ago      314.2 MB

# docker tag myapache:centos kalise/myapache:latest

# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
kalise/myapache     latest              f297cdf1a74a        55 minutes ago      314.2 MB
myapache            centos              f297cdf1a74a        55 minutes ago      314.2 MB
```

Bằng việc tag image, bạn có thể thêm tên để có thể hiểu rõ hơn về image đó.
Image vừa được tạo có thể push trên Docker Hub hoặc có thể save thành 1 tarball
```sh
# docker save -o myapache.tar kalise/myapache:latest
```

### 5. Xây dựng 1 application server
Trong phần này ta sẽ tạo 1 application server image based trên Tomcat. Tạo thư mcuj cho app :
```sh
mkdir taas
cd taas
```

Tạo Dockerfile như sau : 
```sh
# Create the image from the latest centos image
FROM centos:latest

LABEL Version 1.0
MAINTAINER kalise <https://github.com/kalise/>

ENV TOMCAT='tomcat-7' \
    TOMCAT_VERSION='7.0.75' \
    JAVA_VERSION='1.7.0' \
    USER_NAME='user' \
    INSTANCE_NAME='instance'

# Install dependencies
RUN yum update -y && yum install -y wget gzip tar

# Install JDK
RUN yum install -y java-${JAVA_VERSION}-openjdk-devel && \
yum clean all

# Install Tomcat
RUN wget --no-cookies http://archive.apache.org/dist/tomcat/${TOMCAT}/v${TOMCAT_VERSION}/bin/apache-tomcat-${TOMCAT_VERSION}.tar.gz -O /tmp/tomcat.tgz && \
tar xzvf /tmp/tomcat.tgz -C /opt && \
ln -s  /opt/apache-tomcat-${TOMCAT_VERSION} /opt/tomcat && \
rm /tmp/tomcat.tgz

# Add the tomcat manager users file
ADD ./tomcat-users.xml /opt/tomcat/conf/

# Expose HTTP and AJP ports
EXPOSE 8080 8009

# Mount external volumes for logs and webapps
VOLUME ["/opt/tomcat/webapps", "/opt/tomcat/logs"]

ENTRYPOINT ["/opt/tomcat/bin/catalina.sh", "run"]
```

Tại cùng thư mục, tạo Tomcat user `tomcat-users.xml` file như sau : 
```sh
<?xml version='1.0' encoding='utf-8'?>
<tomcat-users>
  <role rolename="manager-gui"/>
  <user username="tomcat" password="tomcat" roles="tomcat, manager-gui"/>
</tomcat-users>
```

Compile image :
```sh
docker build -t taas:1.0 .
```

Chạy container :
```sh
docker run -d -p 8080:8080 taas:1.0
```

Chỏ tới browser với port 8080 và login và Tomcat app server.
