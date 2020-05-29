# Quản lý Security Group trên Keystone

 *	[1 Security Group là gì?](#1)
 *	[2 Cơ chế hoạt động ](#2)
 *	[3 Các use-case áp dụng ](#3)
 *	[4 Các câu lệnh ](#4)
 *	[5 Bài lab với security group ](#5)

## 1. Security Group là gì  <a name="1"> </a>

Security group là tập hợp các rule IP filter được áp dụng tới tất cả các máy ảo bên trong project, mục tiêu là kiểm soát việc truy cập mạng tới máy ảo.

Các Group rule được đặt theo project, các thành viên trong project có thể chỉnh sửa các rule mặc định cho group của họ và thêm các rule mới.

Mặc định thì security group và quota của nó được quản lý bởi Neutron.


## 2. Cơ chế hoạt động của Security Group
Mục tiêu : 
	- Các tính chất cơ bản của Security group
	- Mô hình hoạt động của secutiry group 
	- Cách security group tương tác với IPTABLES
	- Các cấu hình quan trọng của secutiry group trên OpenStack.
	
### 2.1.Các tính chất cơ bản của Security group

Các đặc tính mặc định của Neutron Security Group : 

 - Với Ingress traffic (traffic từ ngoài tới máy ảo) :
	 - Khi không có rule nào được đặt ra, tất cả các traffic đều bị DROP
	 - Chỉ các traffic phù hợp với các rule của security group thì mới được ALLOW
	
 - Với Egress traffic (traffic từ máy ảo đi ra ngoài)
	 - Khi không có rule nào được đặt ra, tất cả các egress traffic đều bị DROP
	 - Chỉ các traffic phù hợp với các rule của security group thì mới được ALLOW
	 - Khi một secutiry group mới được tạo, Các rule ALLOW mọi egress traffic sẽ được tự động tạo.
	 
 - "Default security group" được chỉ định cho mỗi project.
	 - Với mỗi default security, một rule mặc định sẽ được tạo cho phép giao tiếp nội bộ giữa các host dùng chung default security group.
	 
### 2.2. Mô hình hoạt động của Secutiry group

Kiến trúc của Secutiry group 