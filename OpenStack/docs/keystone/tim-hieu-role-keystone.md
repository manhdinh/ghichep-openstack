# Quản lý các Role trên Keystone

 *	[1 Role trên Keystone là gì?](#1)
 *	[2 Phân loại các Role ](#2)
 *	[3 Các use-case áp dụng ](#3)
 *	[4 Các câu lệnh ](#4)
 *	[5 Bài lab với role ](#5)

## 1. Role trên Keystone là gì?

Giống như các dịch vụ khác trong Openstack, keystone bảo vệ API bằng việc sử dụng Role-based Access control (RBAC).

Từ phiên bản Rocky, keystone cung cấp mặc định 3 role là : `admin`, `member` và `reader`. Người vận hành có thể gán các role tới bất kì ai (group hoặc user) trên bất kỳ đối tượng nào (system, domain hoặc project). 

Các role mặc định cho phép người vận hành ủy thác được nhiều chức năng tới team, customer hoặc user mà không cần phải duy trì các policy tùy chỉnh.

### 1.1. Định nghĩa Role

Các role mặc định đều bao hàm những role khác. Role `admin` bao hàm `member` role, và `member`  role bao hàm `reader` role.
Điều này có nghĩa là các user với `admin` role tự động có `member` và `reader` role và user với `member` role tự động có `reader` role. Việc này sẽ giảm sự phức tạp của việc chính sách bằng cách rút ngắn các chuỗi kiểm tra, giảm sự phân công cho role và hình hành hệ thống phân cấp tự nhiên giữa các role mặc định.

Ví dụ, với một policy yêu cầu role `reader` có thể thể hiện là :

```sh
"identity:list_foo": "role:reader"
```

Thay vì : 

```sh
"identity:list_foo": "role:admin or role:member or role:reader"
```