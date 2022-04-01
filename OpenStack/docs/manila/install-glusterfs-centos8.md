172.20.0.51   glusterfs-01     glusterfs-01.example.com
172.20.0.52   glusterfs-02     glusterfs-02.example.com
172.20.0.53   glusterfs-03     glusterfs-03.example.com

### Gluster1
fdisk /dev/vdb
pvcreate /dev/vdb1
vgcreate vg-brick1  /dev/vdb1
lvcreate -l 100%FREE -n brick1 vg-brick1

mkfs.xfs /dev/mapper/vg--brick1-brick1
mkdir -p /bricks/brick1
mount /dev/mapper/vg--brick1-brick1 /bricks/brick1/
echo "/dev/mapper/vg--brick1-brick1  /bricks/brick1   xfs     defaults        0 0" >> /etc/fstab

### Gluster2
fdisk /dev/vdb
pvcreate /dev/vdb1
vgcreate vg-brick2  /dev/vdb1
lvcreate -l 100%FREE -n brick2 vg-brick2

mkfs.xfs /dev/mapper/vg--brick2-brick2
mkdir -p /bricks/brick2
mount /dev/mapper/vg--brick2-brick2 /bricks/brick2/
echo "/dev/mapper/vg--brick2-brick2  /bricks/brick2   xfs     defaults        0 0" >> /etc/fstab


### Gluster3
fdisk /dev/vdb
pvcreate /dev/vdb1
vgcreate vg-brick3  /dev/vdb1
lvcreate -l 100%FREE -n brick3 vg-brick3

mkfs.xfs /dev/mapper/vg--brick3-brick3
mkdir -p /bricks/brick3
mount /dev/mapper/vg--brick3-brick3 /bricks/brick3/
echo "/dev/mapper/vg--brick3-brick3  /bricks/brick3   xfs     defaults        0 0" >> /etc/fstab


### Gluster3
fdisk /dev/vdb
pvcreate /dev/vdb1
vgcreate vg-brick4  /dev/vdb1
lvcreate -l 100%FREE -n brick4 vg-brick4

mkfs.xfs /dev/mapper/vg--brick4-brick4
mkdir -p /bricks/brick4
mount /dev/mapper/vg--brick4-brick4 /bricks/brick4/
echo "/dev/mapper/vg--brick4-brick4  /bricks/brick4   xfs     defaults        0 0" >> /etc/fstab

### Tạo peer
gluster peer probe glusterfs-01.example.com
gluster peer probe glusterfs-02.example.com
gluster peer probe glusterfs-03.example.com
gluster peer probe glusterfs-04.example.com


### Cài GlusterFS
dnf install centos-release-gluster -y 
dnf config-manager --set-enabled PowerTools
dnf install glusterfs-server
dnf install glusterfs glusterfs-fuse -y
systemctl enable glusterd
systemctl start glusterd

### Tạo volume

gluster volume create manila replica 2 glusterfs-01:/bricks/brick1/manila glusterfs-02:/bricks/brick2/manila --yes
gluster volume start manila


gluster volume remove-brick glusterfs-01:/bricks/brick5/manila


## Xử lý SSL

### GlusterFS1
cd /etc/ssl/
sudo openssl genrsa -out glusterfs.key 2048
sudo openssl req -new -x509 -key glusterfs.key -subj "/CN=glusterfs-01" -out glusterfs.pem

### GlusterFS2
cd /etc/ssl/
sudo openssl genrsa -out glusterfs.key 2048
sudo openssl req -new -x509 -key glusterfs.key -subj "/CN=glusterfs-02" -out glusterfs.pem

### GlusterFS3
cd /etc/ssl/
sudo openssl genrsa -out glusterfs.key 2048
sudo openssl req -new -x509 -key glusterfs.key -subj "/CN=glusterfs-03" -out glusterfs.pem

### GlusterFS4
cd /etc/ssl/
sudo openssl genrsa -out glusterfs.key 2048
sudo openssl req -new -x509 -key glusterfs.key -subj "/CN=glusterfs-04" -out glusterfs.pem

### Client
cd /etc/ssl/
sudo openssl genrsa -out glusterfs.key 2048
sudo openssl req -new -x509 -key glusterfs.key -subj "/CN=manila" -out glusterfs.pem

### Client
cd /etc/ssl/
sudo openssl genrsa -out glusterfs.key 2048
sudo openssl req -new -x509 -key glusterfs.key -subj "/CN=manila" -out glusterfs.pem


mkdir /tmp/ca/
cd /tmp/ca/
scp root@glusterfs-02:/etc/ssl/glusterfs.pem glusterfs-02.pem
scp root@glusterfs-03:/etc/ssl/glusterfs.pem glusterfs-03.pem
scp root@glusterfs-04:/etc/ssl/glusterfs.pem glusterfs-04.pem
scp root@cas-manila:/etc/ssl/glusterfs.pem cas-manila.pem

## copy file from client01 too ##
scp root@172.20.1.62:/etc/ssl/glusterfs.pem manila.pem

## key cho server
cat /etc/ssl/glusterfs.pem glusterfs-02.pem glusterfs-03.pem glusterfs-04.pem manila.pem cas-manila.pem > glusterfs.ca

### key cho client
cat /etc/ssl/glusterfs.pem glusterfs-02.pem glusterfs-03.pem glusterfs-04.pem > glusterfs-client.ca

### copy key tới server
sudo cp glusterfs.ca /etc/ssl/
scp glusterfs.ca root@glusterfs-02:/etc/ssl/
scp glusterfs.ca root@glusterfs-03:/etc/ssl/
scp glusterfs.ca root@glusterfs-04:/etc/ssl/

### Copy key tới client
scp glusterfs-client.ca root@172.20.1.62:/etc/ssl/glusterfs.ca
scp glusterfs-client.ca root@cas-manila:/etc/ssl/glusterfs.ca