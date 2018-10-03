# Quản lý dịch vụ Identity (Keystone)

 *	[1 Một số khái niệm](#1)
 *	[2 Quản lý Project ](#2)
 *	[3 Quản lý user ](#3)
 *	[4 Quản lý role ](#4)
 *	[5 Phân quyền user trong file policy.json ](#5)
 *	[6 Log xác thực của keystone ](#6)

## 1 Một số khái niệm  <a name="1"> </a>

 -	Authentication : Quá trình xác thực thông tin user. Để xác thực các yêu cầu khi truyền đến, Keystone phê duyệt thông tin xác thực người dùng. Lúc đầu, các thông tin xác thực này bao gồm username hoặc password, hoặc usernmae và API key. Khi các thông tin xác thực người dùng được phê duyệt, một **token** xác thực sẽ được cung cấp. Người dùng cung cấp token này cho các yêu cầu lần sau.
 
 - Credentials : Data của người dùng mà được Keystone xác thực. Ví dụ, username và password, username và API key, hoặc token xác thực mà Keystone cung cấp.
 
 - Domain : Một API v3 entity của Keystone. Domain là tổ hợp của các project và user để chỉ rõ ra quyền hạn trong việc quản lý các đối tượng xác thực. Domain có thể đại diện cho cá nhân, tổ chức hoặc operator-owned space. Họ thể hiện các hành động quản trị tới các system user một cách trực tiếp. User có thể được phân quyền admin cho một domain. Một domain admin có thể tạo project, user và group trong domain và gán các quyền tới user và group trong một domain.
 
 - Endpoint : Một địa chỉ mạng có thể truy cập được, thông thường là một URL, thông qua đó mà bạn có thể truy cập được vào một service. 
 
 - Group : Một API v3 entity của Keystone. Group là tổ hợp các user được sở hữu bởi một domain. Khi một group được gán quyền bởi domain hoặc project, sẽ áp dụng cho tất cả các user trong group. Việc thêm user vào group, hoặc bỏ user ra khỏi group, sẽ gán hoặc loại bỏ role và authentication của user liên quan đến project và domain.
 
 - Openstackclient : Giao diện cmd cho một vài dịch vụ bao gồm của Identity API. VD, user có thể chạy command **openstack service create** hoặc **openstack endpoint create** để đăng ký service.
 
 - Project : Tổ hợp mà các group hoặc các tài nguyên riêng biệt hoặc các đối tượng xác thực. Dựa vào service operator, một project có thể map tới một cusomter, account, organization hoặc tenant.
 
 - Region : Một API v3 entity của Keystone. Đại diện cho việc phân chia thông thường trong quá trình triển khai Openstack.
 
 - Role : Chỉ định ra quyền của user hoặc và cho phép thực hiện một tổ hợp các hành động. Keystone cung cấp token cho user bao gồm một list các role. Khi người dùng gọi tới một service, service sẽ diễn giải user role, và xác định các hành động hoặc tài nguyên cho user ứng với mỗi role được gán.
 
 - Service : Openstack service, ví dụ Nova, Swift, Glance... cung cấp một hoặc nhiều endpoint, thông qua đó mà user có thể truy cập được vào tài nguyên và thực hiện các hành động.
 
 - Token : Chuỗi ký tự số và chữ cái cho phép truy cập các Openstack API và các tài nguyên. Một token có thể bị thu hồi và có giá trị trong một khoảng thời gian hữu hạn
 
 - User : Đại diện cho một cá nhân (person), hệ thống (system), hoặc một dịch vụ (service) dùng Openstack serrvice. Keystone validate các yêu cầu đầu vào được tạo bởi user. User có thể login và truy cập tài nguyên bằng việc sử dụng token đã được cung cấp. User có thể được ấn định vào một project cụ thể và hành động như thể user đó được chứa trong project.
 
## 2 Quản lý Project <a name="2"> </a>

 -	List project 
 
`openstack project list`

 - Tạo project mới với domain default
 
`openstack project create --description 'description' PROJECT-NAME --domain default`
 
 -	Update project. Update tên, miêu tả và trạng thái của một project.
 
	- Tạm thời disable project
	
	`openstack project set PROJECT_ID --disable`
	
	- Enable project
	
	`openstack project set PROJECT_ID --enable`
	
	- Update tên mới cho project
	
	`openstack project set PROJECT_ID --name project-new-name`
	
	- Kiểm tra thông tin project
	
	`openstack project show PROJECT-ID`
	
 -	Xóa project
 
`openstack project delete PROJECT-ID`

## 3 Quản lý user <a name="3"> </a>

 -	List user
 
`openstack user list`

 - Tạo user mới
 
`openstack user create --project new-project --password PASSWORD new-user`

 - Update thông tin về user
 
	- Tạm thời disable user
	
	`openstack user set USER_NAME --disable`
	
	- Enable user
	
	`openstack user set USER_NAME --enable`
	
	- Upadte thông tin cho user
	
	`openstack user set USER_NAME --name user-new --email new-user@example.com`
	
 -	Xóa user 
 
`openstack user delete USER_NAME`

 -	Show thông tin chi tiết về user
 
`openstack user show USER_NAME`

## 4 Quản lý role <a name="4"> </a>

 -	List role
 
`openstack role list`

 - Tạo role
 
`openstack role create new-role`

Nếu dùng Keystone v3, phải có thêm tùy chọn `--domain`

 -	Gán role
 
`openstack role add --user USER_NAME --project RPOJECT_ID ROLE_NAME`

 - Show thông tin về role được gán cho user.
 
`openstack role assignment list --user USER_NAME --project PROJECT_ID --names`

 - Remove role ra khỏi user
 
`openstack role remove --user USER_NAME --project PROJECT_ID ROLE_NAME`

 - List thông tin role được gán
 
`openstack role list --user USER_NAME --project PROJECT_ID`

## 5 Phân quyền user trong file policy.json <a name="5"> </a>

Một user có thể có nhiều role trong các project khác nhau hoặc có nhiểu role trong cùng một project. File `policy.json` kiểm soát quyền hạn mà user có thể thực hiện được với từng project. Để giới hạn quyền mà user có thể làm với service, cần sửa đổi trong file `policy.json`.

Ví dụ, muốn giới hạn user `manhdv01` có thể tạo volume. Thay đổi quyền trong `/etc/cinder/policy.json`

`"volume:create": "role:manhdv01",

## 6 Log xác thực của keystone <a name="6"> </a>

 Các log về xác thực của keystone sẽ được ghi lại trong file log : `/var/log/http/error.log`.
 
Ví dụ : Login với user `manhdv01`, và thực hiện tạo volume. User `manhdv01` đã bị giới hạn quyền tạo volume nên log sẽ báo như sau : 

![keystone](![ops](/ManhDV/OpenStack/images/keystone-log.png)

