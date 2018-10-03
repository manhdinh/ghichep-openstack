# Cài đặt CheckMK-RAW trên Centos7

## 1. Chuẩn bị

 -	Thư mục lưu trữ.
 
 Check_MK lưu data vào /opt/omd vậy nên cần chuẩn bị không gian lưu trữ đủ cho thư mục này.
 
 -	SMTP cho outgoing email
 
 Cài đặt SMTP để có thể gửi mail. Cài đặt này sẽ được đi liền trong quá trình cài đặt Check-MK

 - Sử dụng NTP
 
 Cài đặt NTP để các function trong Check_MK đồng bộ về thời gian.
 
## 2. Cài đặt 

 - Cài đặt repo EPEL cho Centos7
 
`rpm -Uvh https://dl.fedoraproject.org/pub/epel/7/x86_64/e/epel-release-7-9.noarch.rpm`

 - Cài đặt repo OMD cho Centos7
 
```sh
rpm -Uvh "https://labs.consol.de/repo/stable/rhel7/i386/labs-consol-stable.rhel7.noarch.rpm"`
yum update
yum install omd-1.30.x86_64
```

 - Chạy lệnh setup các gói cần thiết cho OMD
 
`omd setup`

 - Tạo hệ thống giám sát riêng 
 
`omd create manhdvmonitor`

 - Cập nhập file hashlib.py cho OMD, nếu không sẽ bị lỗi khi đăng nhập OMD-WEB.
 
```sh
rm -rf /omd/versions/1.30/lib/python/hashlib.py
cp /usr/lib64/python2.7/hashlib.py /omd/versions/1.30/lib/python/hashlib.py
```

 - Đăng nhập với user `manhdvmonitor` và khởi động hệ thống OMD
 
```
su - manhdvmonitor
omd start
```

 - Kiểm tra trạng thái của OMD với câu lệnh
 
`omd status`

 - Đăng nhập vào web quản trị OMD với đường dẫn : `http://IP-OMDserver/manhdvmonitor`. User/pass mặc định là  **omdadmin/omd**.
 
![OMD](/ManhDV/Monitor/images/omd-web.png)



 
