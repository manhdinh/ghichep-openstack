## Container Storage

Docker phân loại các yêu cầu lưu trữ thành 3 loại chính : 
	- Layered Filesystem cho container chạy storage
	- Persistent Volumes cho việc lưu trữ dữ liệu lâu dài
	- Registry cho việc lưu trữ image

Phần này sẽ đi sâu vào chi tiết của 3 lớp lưu trữ riêng biệt : *graph driver storage*, *volume storage* và *registry storage*.

### 1. Layered Filesystem
Lưu trữ được dùng cho việc đọc các tầng image filesystem từ một trạng thái container đang chạy với đặc trưng là yêu cầu IOPS hoặc các hoạt động read/write có cường độ cao, dẫn tới hiệu năng trở thành `key storage metric`. Docker chọn 1 kiến trúc `layered storage` cho các image và container. Một layered file system được tạo bởi nhiều các layer riêng rẽ cho phép các image được cấu trúc và tái cấu trúc theo nhu cầu thay vì việc tạo một image lớn và nguyên khối.

Các layered filesystem được hỗ trợ là : 

	- device mapper
	- btrfs
	- aufs
	- overlay
	
Overlay file system trở thành lựa chọn mặc định cho tất cả các cả phiên bản Linux. Tuy nhiên Centos/RHEL vẫn được khuyến cáo để dùng device mapper.
Kiểm tra storage layered file system, kiểm tra docker engine :
```sh
docker system info | grep -i "Storage Driver"
Storage Driver : overlay
```

Khi 1 docker image được pull từ registry, engine dowload tất cả các layer phụ thuộc tới host. Khi container được tạo từ 1 image gồm nhiều layer, docker dùng khả năng *Copy-on-Write* của layered file system để thêm 1 thư mục read write trên lên lớp trên của các layer read only đã có sẵn.

![Docker](/ManhDV/Docker/images/container-layers.jpg)

Pull 1 image ubuntu từ Docker Hub
```sh
[root@clastix00 ~]# docker pull ubuntu:15.04
15.04: Pulling from library/ubuntu
9502adfba7f1: Pull complete
4332ffb06e4b: Pull complete
2f937cc07b5f: Pull complete
a3ed95caeb02: Pull complete
Digest: sha256:2fb27e433b3ecccea2a14e794875b086711f5d49953ef173d8a03e8707f1510f
Status: Downloaded newer image for ubuntu:15.04

[root@host ~]# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
ubuntu              15.04               d1b55fd07600        14 months ago       131 MB
```

Image layer được đặt trong thư mục tại localhost :
```sh
[root@host ~]# ll /var/lib/docker/overlay
total 0
drwx------ 3 root root 17 Apr 11 13:25 863324a4a64a561da3d5f1623040dd292d079a810fd8767296ae6d6f7561b902
drwx------ 3 root root 17 Apr 11 13:25 8c56e1d6822091e11edfb1b14b586ba29788de5bcdaa00234a0b5dfa1432ff7b
drwx------ 3 root root 17 Apr 11 13:34 c154bf1e4f992ce599b497da5fb554e52d32127eb4ecc3a141cbf48331788dba
drwx------ 3 root root 17 Apr 11 13:25 e8ef46122ade93666dd7c1218580063d0f4e0869f940707b2a3cdbf7ea83e9cb
```

Khi 1 container start, layer read write ban đầu sẽ rỗng tới khi có sự thay đổi bằng việc chạy container process. Khi một process cố gắng write 1 file mới hoặc update 1 file có sẵn, filesystem tạo 1 bản copy của file mới ở tầng trên của writeable layer.
```sh
[root@host ~]# docker run --name ubuntu -it ubuntu:15.04
root@3b927950034f:/# useradd adriano
root@3b927950034f:/# passwd adriano
Enter new UNIX password:
Retype new UNIX password:
passwd: password updated successfully
...
```

Không may là, khi container biến mất thì upper writeable layer cũng sẽ bị remove và tất cả nội dung sẽ bị mất trừ khi một image mới được tạo trong khi container còn sống. Khi 1 image mới được tạo từ 1 container đang chạy, chỉ các thay đổi được tạo ra cho writeable layer, sẽ được thêm vào layer mới.

Để tạo 1 layer mới từ 1 container đang chạy : 
```sh
[root@host ~]# docker commit -m "added a new user" ubuntu ubuntu:latest
sha256:e46141824bfc8118d6f960b9be1d70bb917e2b159734ee2d2d26e2336521528a

[root@host ~]# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
ubuntu              latest              e46141824bfc        2 minutes ago       132 MB
ubuntu              15.04               d1b55fd07600        14 months ago       131 MB


[root@host ~]# docker history ubuntu:latest
IMAGE               CREATED             CREATED BY                                      SIZE      COMMENT
e46141824bfc        40 seconds ago      /bin/bash                                       330 kB    added a new user
d1b55fd07600        14 months ago       /bin/sh -c #(nop) CMD ["/bin/bash"]             0 B
<missing>           14 months ago       /bin/sh -c sed -i 's/^#\s*\(deb.*universe\...   1.88 kB
<missing>           14 months ago       /bin/sh -c echo '#!/bin/sh' > /usr/sbin/po...   701 B
<missing>           14 months ago       /bin/sh -c #(nop) ADD file:3f4708cf445dc1b...   131 MB
```

Chú ý rằng image ubuntu đã thay đổi không có các bản copy của chỉnh nó của mỗi layer. Image mới sẽ chia sẻ các layer bên dưới với những image trước đã như ví dụ sau : 
![Docker](/ManhDV/Docker/images/saving-space.jpg)

Tất cả các container khởi động với image mơí nhất sẽ share các layer với tất cả container từ image đầu tiên. Điều này sẽ tối ưu cả không gian sử dụng image và hiệu năng hệ thống.

### 2. Persisten volume

Các container thường yêu cầu dữ liệu phải được lưu trữ lâu dài và liên tục, vượt lên trên vòng đời của container, ưu cầu phải dùng tới việc lưu trữ bằng persistent volume. Thực tế cho thấy, dữ liệu nên được cô lập với container. VD như việc quản lý dữ liệu nên được tách rời từ vòng đời của container.

Việc lưu trữ dữ liệu lâu dài (persistent volume) là 1 use case quan trọng, đặc biệt là với database, image, file và folder chia sẻ giữa các container. Để làm được việc này thông thường có 2 cách tiếp cận :

 - 1. host based volumes
 - 2. shared volumes

Trong trường hợp đầu, persistent volume tồn tại trên cùng host nơi container đang chạy. Trong trường hợp sau, volumes tồn tại trên 1 shared filesystem như NFS, GlusterFS... Trong TH đầu tiên, dữ liệu được lưu trữ lâu dài trên host, có nghĩa rằng nếu container bị di chuyển tới host khác, nội dung của volume không còn ở trên container mới. Trong TH2, miễn là mount point được truy cập từ mọi node thì volume có thể được mount tới bất kỳ container nào. TH2 cung cấp dữ liệu thông qua một cluster của các host.

Persistent volume được map từ volume được chỉ định bên từ Dockerfile tới filesystem trên host đang chạy container. 
VD Dockerfile định nghĩa một volume `var/log` tới web app lưu trữ access log :
```sh
# Create the image from the latest nodejs
# The image is stored on Docker Hub at docker.io/kalise/nodejs-web-app:latest

FROM node:latest

MAINTAINER kalise <https://github.com/kalise/>

# Create app directory
RUN mkdir -p /usr/src/app
WORKDIR /usr/src/app

# Install app dependencies
COPY package.json /usr/src/app/
RUN npm install

# Bundle app source
COPY . /usr/src/app

# Declare Env variables
ENV MESSAGE "Hello World!"

# Define the log mount point
VOLUME /var/log

# Expose the port server listen to
EXPOSE 8080
CMD [ "npm", "start" ]
```

Start một container từ nodejs image : 
```sh
[root@centos ~]# docker run --name=nodejs \
   -p 80:8080 -d \
   -e MESSAGE="Hello" \
docker.io/kalise/nodejs-web-app:latest
```

Tất cả các persistent volume được tạo ở thư mục  `var/lib/docker/volumes/` . Kiểm tra container để check volume :

```sh
[root@centos ~]# docker inspect nodejs
...
"Mounts": [
      {
          "Name": "84894a09fe25f503cd0f2d3a30eaa00a08d72190a92e2568d395cea5a277c456",
          "Source": "/var/lib/docker/volumes/84894a09fe25f503cd0f2d3a30eaa00a08d72190a92e2568d395cea5a277c456/_data",
          "Destination": "/var/log",
          "Driver": "local",
          "Mode": "",
          "RW": true,
          "Propagation": ""
      } ]
...
```

Volume vẫn còn kể cả container bị xóa :
```sh
[root@centos ~]# docker rm -f nodejs

[root@centos ~]# ls -l /var/lib/docker/volumes/84894a09fe25f503cd0f2d3a30eaa00a08d72190a92e2568d395cea5a277c456/
total 4
drwxr-xr-x 4 root root 4096 Apr 11 16:32 _data
```

Để xóa bằng tay 1 volume, tìm volume và remove nó :
```sh
[root@centos ~]# docker volume list
DRIVER              VOLUME NAME
local               84894a09fe25f503cd0f2d3a30eaa00a08d72190a92e2568d395cea5a277c456
    
[root@centos ~]# docker volume remove 84894a09fe25f503cd0f2d3a30eaa00a08d72190a92e2568d395cea5a277c456
```

Persistent volume có thể được tạo trước container và sau đó attach tới container :
```sh
[root@centos ~]# docker volume create --name myvolume
[root@centos ~]# docker volume inspect myvolume
```

```sh
    [
        {
            "Driver": "local",
            "Labels": {},
            "Mountpoint": "/var/lib/docker/volumes/myvolume/_data",
            "Name": "myvolume",
            "Options": {},
            "Scope": "local"
        }
    ]
```

```sh
[root@centos ~]# docker run --name=nodejs \
       -p 80:8080 -d \
       -e MESSAGE="Hello" \
       -v myvolume:/var/log \
    docker.io/kalise/nodejs-web-app:latest

[root@centos ~]# docker inspect nodejs
```

```sh
    ...
    "Mounts": [
            {
                "Type": "volume",
                "Name": "myvolume",
                "Source": "/var/lib/docker/volumes/myvolume/_data",
                "Destination": "/var/log",
                "Driver": "local",
                "Mode": "z",
                "RW": true,
                "Propagation": ""
            }
        ]
    ...
```

**Chú ý** : Persistent volume được khởi tại khi 1 container được tạo. Nếu image parent của container chứa dữ liệu tại một mount point nào đó, data đã tồn tại được copy tới volume mới trong lúc khởi tạo volume.

Các persisten volume có thể được mount trên bất kỳ thư mục nào của filesystem trên host. Điều này giúp chia sẽ dữ liệu giữa host và container. VD, bạn có thể mount volume trên thư mục `/logs` của host đang chạy container :
```sh
[root@centos ~]# docker run --name=nodejs \
   -p 80:8080 -d \
   -e MESSAGE="Hello" \
   -v /logs:/var/log \
docker.io/kalise/nodejs-web-app:latest
```

Lúc này volume data sẽ được đặt tại thư mục `/logs` của host.

Volume có thể được mount bởi các container khác. Tuy nhiên việc nhiều container cùng viết xuống 1 single shared volume có thể khiến dữ liệu bị sai lệch.

**Chú ý** : Khi mounting một persistent volume trên một thư mục host, nếu image parent của container chứa dữ liệu tại một mount point nào đó, data đã tồn tại được copy tới volume mới trong lúc khởi tạo volume.

### 3. Registry
#### 3.1 Docker registry service

Docker registry service là 1 hệ thống lưu trữ và phân phối nội dung bao gồm các image đã được tag. Dịch vụ registry chính là Docker Hub nhưng người dùng cũng có thể có registry riêng. Người dùng tương tác với 1 registry bằng việc dùng các câu lệnh như `push` và `pull`
```sh
[root@centos ~]# docker pull ubuntu:latest
```

Command trên hướng dẫn docker engine pull bản image ubutnu mới nhất từ Docker Hub. Đó là bản rút gọn của :
```sh
[root@centos ~]# docker pull docker.io/library/ubuntu:latest
```

Để pull các image từ local registry, dùng :
```sh
[root@centos ~]# docker pull <myregistrydomain>:<port>/kalise/ubuntu:latest
```

Command trên hướng dẫn Docker engine contact tới registry đặt tại `<myregistrydomain>:<port>` để tìm image `kalise/ubuntu:lastest`.

### 3.2. Triển khai local Registry service
Để triển khai 1 local registry service trên `myregistry:5000`, cần cài đặt Docker trên server đó và tạo 1 Registry container dựa vào registry image được cung cấp bởi Docker Hub :
```sh
[root@centos ~]# docker pull registry:latest
[root@centos ~]# docker run -d -p 5000:5000 --restart=always --name docker-registry registry:latest
```

Lấy 1 image từ public Docker Hub, tag nó tới local registry service.
```sh
[root@centos ~]# docker pull docker.io/kalise/httpd
[root@centos ~]# docker tag docker.io/kalise/httpd myregistry:5000/kalise/httpd
```

Registry phía trên được coi là không an toàn bởi Docker Engine. Để registry này có thể dùng được, mỗi Docker engine host cần được hướng dẫn để trust registry service chạy trên host `myregistry:5000`.

Thay đổi file cấu hình docker engine `/etc/docker/daemon.json` :
```sh
{
  "storage-driver": "devicemapper",
  "hosts": ["tcp://0.0.0.0:2375","unix:///var/run/docker.sock"],
  "insecure-registries": ["myregistry:5000"]
}
```

và sau đó khởi động lại Docker engine
```sh
systemctl restart docker.service
```

Sau khi local registry được trust thì ta có thể push image vào đó :
```sh
docker push myregistry:5000/kalise/httpd
```

Image có thể được pull từ local registry :
```sh
docker pull myregistry:5000/kalise/httpd
```

Để secure registry với self-signed certificate. Đầu tiên cần tạo ceritficate 
```sh
mkdir /etc/certs
cd /etc/certs
openssl req \
-newkey rsa:4096 -nodes -sha256 -keyout domain.key \
-x509 -days 365 -out domain.crt
```

Đảm bảo rằng tham số Common Name đúng với IP của host registry. Docker engine cần được hướng dẫn trust certificate này :
```sh
mkdir -p /etc/docker/certs.d/myregistry:5000
cp /etc/certs/domain.crt /etc/docker/certs.d/myregistry:5000/ca.crt
```

Remove insecure registry ở bước trước bằng việc thay đổi file cấu hình `/etc/docker/daemon.json` :
```sh
{
  "storage-driver": "devicemapper",
  "hosts": ["tcp://0.0.0.0:2375","unix:///var/run/docker.sock"],
}
```
sau đó restart Docker engine.

Start registry ở chế độ secure mode với certificate như là 1 local volume và đjăt các biến môi trường liên quan tới container : 
```sh
docker run -d -p 5000:5000 --restart=always --name docker-registry \
  -v /etc/certs:/certs \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
  -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
  registry:latest
```

Bây giờ bạn có thể push/pull image to/from local registry
```sh
docker push myregistry:5000/kalise/httpd
docker pull myregistry:5000/kalise/httpd
```

### 3.3. Storage backend cho registry
Mặc định thì data trong container là dạng ephemeral (không bền vững), điều đó có nghĩa nó sẽ biến mất khi container registry mất đi. Để giữ registry container giữ data image lâu dài, sử dụng docker volume trên filesystem của host.
```sh
mkdir /data
docker run -d -p 443:5000 --restart=always --name docker-registry \
-v /data:/var/lib/registry \
-v /etc/certs:/certs \
-e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
-e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
registry:latest
```

Bằng việc dùng thư mục `/data` như là backend cho registry container, các image được push vào registry đó sẽ tồn tại kể cả khi registry biến mất. Tuy nhiên, việc dùng 1 persistent backend không bảo vệ dữ liệu mất đi khi nơi lưu trữ gặp vấn đề. Trong môi trường production, 1 lựa chọn an toàn hơn được dùng là sử dụng 1 hệ thống shared storage như là NFS share.

### 3.4. Đặt Docker registry mặc dịnh
Cấu hình 1 registry mirror để thay đổi registry mặc định từ Docker Hub thành local registry. Khi request đầu tiên của docker cho 1 image từ local registry mirror, nó sẽ pull image từ public Docker Hub và lưu trữ nó tại local. Tại các request tiếp theo, local registry mirror sẵn sàng cho việc sử dụng image từ local.

Chỉnh sửa file cấu hình `/etc/docker/daemon.json` :
```sh
{
  "storage-driver": "devicemapper",
  "hosts": ["tcp://0.0.0.0:2375","unix:///var/run/docker.sock"],
  "registry-mirrors": ["https://myregistry:5000"]
}
```

Restart lại Docker engine


