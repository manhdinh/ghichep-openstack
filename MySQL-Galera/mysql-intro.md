# Giới thiệu sơ bộ về MySQL nói chung

## I. MySQL là gì

 - Mysql là một hệ thống quản lý database
 
 Một database là một tập hợp có cấu trúc của dữ liệu. Nó có thể là một danh dánh đơn giản hoặc là một số lượng lớn thông tin về mạng lưới một công ty. Để thêm, truy cập và thực hiện dữ liệu được lưu trữ trong một cơ sở dữ liệu, bạn cần một hệ thống quản lý database như MySQL server.
 
 - MySQL relational database
 
 Một relational databse lưu trữ dữ liệu vào các bảng riêng biệt chứ không đẩy tất cả dữ liệu vào một kho lưu trữ lớn. Cấu trúc database được tổ chức tới các physical file cho để tăng tốc độ. Logical model rất linh hoạt với các đối tượng như database, table, view, row và column.
 
 Phần `SQL` trong MySQL đại diện cho `Structured Query Language`. SQL là ngôn ngữ tiêu chuẩn dùng để truy cập database.
 
## II. Kiến trúc

### 1. Kiến trúc tổng quan của một RDBMS (Relational Database Management System)

Các tầng kiến trúc của một database

![MYSQL](/ManhDV/MySQL-Galera/images/mysql01.png)

### 1.1 Tầng Application

 Tầng application là giao diện hệ thống dành cho người dùng, nó cung cấp phương thúc để từ bên ngoài có thể tương tác được với database server. Thông thường, người dùng được phân loại thành 4 group :
 - **Sophisticated user** : tương tác với hệ thống bằng cách tạo request trực tiếp với việc sử dùng ngôn ngữ truy vấn database.
 - **Specialized user** : là người lập trình ứng dụng, viết các ứng dụng databse đặc biệt khác với data-processing framework truyền thống.
 - **Naive user** : unsophisticated user, người tương tác với hệ thống bằng chương trình ứng dụng được viết trước đó
 - **Database Administrator** : có toàn bộ quyền kiểm soát với hệ thống database. Họ có nhiều trách nhiệm, bao gồm định nghĩa schema, cấp quyền hạn truy cập...
 
 Tất cả các hệ thống database cung cấp các dịch vụ mở rộng với từng nhóm group.
 
### 1.2 Tầng Logical 

 Các chức năng chính của RDBMS được thể hiện qua tầng logical. Sơ đồ sau là là một high level abstraction.
 
![MYSQL](/ManhDV/MySQL-Galera/images/mysql02.png) 

 Có 4 module được thể hiện trong RDBMS.
 
### 1.3 Tầng Physical
 
 RDBMS có trahcj nhiệm trong việc lưu trữ thông tin, các thông tin được giữ trong storage thứ 2 và được truy cập thông qua storage manager. Các dạng của dữ liệu được giữ trong hệ thống là :
  
  - **Data file** : chứa dữ liệu người dùng trong database
  - **Data dictionary** : chứa metadata về cấu trúc của database
  - **Indices** : cung cấp các truy cập tới các data item chứa các giá trị cụ thể
  - **Statiscal data** : chứa thông tin thống kê về data trong database. Được dùng bởi query processor để chọn cách hiệu quả để thực hiện một query.
  - **Log information** : được dùng để theo dỗi các query được xử lý như là reocvery manager có thể dùng thông đi để khôi phục hoàn toàn một database trong trường hợp hệ thống có sự cố.
 
### 2. Kiến trúc của MYSQL

![MYSQL](/ManhDV/MySQL-Galera/images/mysql03.png) 

### 2.1 Tầng Application

Tầng Application của MySQL là nơi mà các client và user tương tác với MySQL RDBMS. Có 3 thành phần trong tầng này có thể được nhìn thấy trong mô hình kiến trúc MySQL. Các thành phần này minh họa các dạng khác nhau của user mà tương tác với MySQL RDBMS, đó là administrator, client và query user. **Administrator** sử dụng giao diện và các tiện ích thuộc admin. Trong MySQL, một số tiện ích như `mysqladmin` có thể thực hiện một số tác vụ như shutdown server, tạo hoặc xóa database, `isamchk` và `myisamchk` thực hiện phân tích bảng và database tới server khác. **Client** giao tiếp với MySQL RDBMS thông qua interface hoặc các tiện ích. Client interface sử dụng MySQL API cho nhiều ngôn ngữ lập trình như C API, DBI API cho Perl, PHP API, Java API, Python API, MySQL C++ API và Tcl. Query user tương tác với MySQL RDBMS thông qua giao diện query `mysql`. `mysql` là một chương trình tương tác cho phép query user đặt các câu lệnh SQL tới server và xem kết quả.

### 2.2 Tầng Logical 

### 2.2.1 Query Processor

 Phần lớn các tương tác trong hệ thống xảy ra khi một user muốn nhìn hoặc dùng dữ liệu được lưu trữ bên dưới. Các truy vấn này, được xác định bằng việc dùng một ngôn ngữ data-manipulation (ví dụ SQL), được được phân tích bởi một bộ xử lý truy vấn. Bộ xử lý này, có thể  như là một kiến trúc đường dẫn và lọc, các kết quả của thành phần phía trước là một input hoặc là một yêu cầu cho thành phần tiếp theo.
 
### 2.2.1.1 Embedded DML Precompiler

 Khi một request được nhận từ một client treent tầng application, embbedded DML (Data Manipulation Languge) precompiler chịu trách nhiệm để trích xuất các câu lệnh SQL có liên quan trong các client API command, hoặc phiên dịch các client command sang các câu lệnh SQL tương ứng. Đây là bước đầu tiên trong quá trình thực tế của một ứng dụng client được viết bằng ngôn ngữ lập trình như là C++ hoặc Perl, trước khi được biên soạn truy vấn SQL. Các client request có thể tới từ các command được xử lý từ một API hoặc một chương trình ứng dụng. MySQL sử dụng thành phần này nhằm chuyển đổi các yêu cầu từ ứng dụng client MySQL sang dạng mà MySQL có thể hiểu được.
 
### 2.2.1.2 DDL Compiler

 Các yêu cầu truy cập tới MySQL database nhận từ người quản trị được thực hiện bởi DDL (Data Definition Language) compiler. DDL compliler biên soạn các command (là các câu lệnh SQL) để tương tác trực tiếp với database. 
 
### 2.2.1.3 Query Parser

 Sau khi các câu lệnh SQL có liên quan thu được từ việc giải mã các client request hoặc administrative request, bước tiếp theo sẽ phân tích cú pháp truy vấn MySQL. Trong bước này, mục tiêu của query parser là tạo một cây cấu trúc phân tích cú pháp trong query để có thể hiểu được dễ dàng các thành phần phía sau trong pipline.
 
### 2.2.1.4 Query Preprocessor
 
 Query parse tree thu được từ query parse sau đó sẽ được dùng bởi query processor để kiểm tra cú pháp và ngữ nghĩa của truy vấn MySQL để query có hợp lệ hay không. Nếu nó là một query hợp lệ, các query progress down pipline. Nếu không hợp lệ, query không được tiếp tục và client sẽ được báo là query có lỗi.

### 2.2.1.5 Security/Integration Manager

 Một khi MySQL query được cho là hợp lệ, MySQL server cần kiểm tra access control list cho client. Security integration manager sẽ kiểm tra client có quyền truy cập đến database MySQL nào và có quyền với table và record nào. 
 
### 2.2.1.6 Query Optimizer

 Sau khi kiểm tra là client có đủ quyền truy cập tới table cụ thể, query được tối ưu, MySQL sử dụng query optimizer để xử lý các SQL query nhanh nhất có thể. Nhiệm vụ của MySQL là phân tích quá trình query để xem có thể thực hiện những tối ưu nào giúp quá trình query diễn ra nhanh hơn. 
 
### 2.2.1.7 Execution Engine

 Khi MySQL query được tối ưu, query có thể được xử lý. Quá trình này được thực hiện bởi query execution engine, sẽ thực hiện việc xử lý các câu lệnh SQL và truy cập tới tầng physical của MySQL database. Người quản trị database có thể xử lý các command trong database để thực hiện các task cụ thể như repair, recovery, copying và backup, khi nhận từ DDL compiler.
 
### 2.2.1.8 Scalability / Evolvability

 Kiến trúc theo dạng lớp của tầng logical của MySQL RDBMS hỗ trợ việc phát triển của hệ thống. Nếu pipline bên dưới của query processor thay đổi, các tầng khác trong RDBMS không bị ảnh hưởng. Đó là bởi vì kiến trúc có những những thành phần tương tác phụ tối thiểu tới các lớp phía trên và dưới nó, có thể nhìn thấy từ sơ đồ kiến trúc. Chỉ các thành phần phụ trong query processor tương tác với các lớp khác là embedded DML preprocessor, DDL compiler và query parser (giai đoạn đầu của pipline) và execution engine (cuối của pipline). Vì vậy nếu query preprocessor security/integration manager và/hoặc query optimizer bị thay thế, nó không ảnh hưởng tới đầu ra của query processror
 
### 2.2.2 Transaction Management

### 2.2.2.1 Transaction manager
 
 Một transaction là một đơn vị làm việc có một hoặc nhiều MySQL command trong nó. Transaction manager chịu trách nhiệm cho việc transaction được ghi lại và xử lý một cách tự động. Nó thực hiện việc này thông qua sự trợ giúp của log manager và concurency control-manager. Hơn nữa, transaction manager chịu trách nhiệm cho việc giải quyết bất kỳ tình huống deadlock nào xảy ra. 
 
### 2.2.2.2 Concurrency - Control Manager

 Concurrency control-manager đảm bảo việc transaction được xử lý riêng rẽ và độc lập
 
### 2.2.3 Recovery Manager

### 2.2.3.1 Log Manager

 - Log tất các hoạt động được xử lý trong database
 - Lưu trữ các log hoạt động như MySQL command
 - Trong trường hợp hệ thống có lỗi và crash việc thực hiện các command này sẽ mang databse trở lại trạng thái ổn định cuối cùng.
 
### 2.2.3.2 Reovery Manager

 - Chịu trách nhiệm cho việc phục hồi database từ trạng thái ổn định cuối cùng.
 - Sử dụng các log được tạo bởi log manager.
 
### 2.2.4 Storage Management

 Tất cả việc truy cập vào storage đều thông qua một số bộ đệm. Các bộ đệm nằm tại bộ nhớ chính và bộ nhớ ảo, được quản lý bởi Buffer Manager. Manager này kết hợp với 2 đối tượng quản lý liên quan đến storage : Resource Manager và Storage Manager.
 
### 2.2.4.1 Storage Manager
 
 - Hoạt động như một giao diện với OS
 - Công việc chính là ghi dữ liệu một cách hiệu quả vào disk
 - Storage Manager ghi vào disk tất cả dữ liệu trong các user table, index, log cũng như internal system data.
 
### 2.2.4.2 Buffer Manager

 - Phân phối tài nguyên memory
 - Quyết định bao nhiêu memory được phân cho mỗi bộ đệm
 - Quyết định bao nhiêu bộ đệm được phân cho mỗi bộ nhớ
 
### 2.2.4.3 Resource Manager

 - Chấp nhận các request từ execution engine
 - Yêu cầu chi tiết từ buffer manager
 - Nhận các reference của data với memory từ buffer manager
 - Quay vòng data từ lớp trên
 