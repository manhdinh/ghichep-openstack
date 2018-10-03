## Tìm hiểu ELK

### 1. ELK stack case study

#### 1.1 Saleforce (cty cung cấp Customer Relationship Management platform)
Saleforce phát triển 1 plugin mới tên ELF (Event log files) để thu thập Saleforce logged data, cho phép theo dõi các hoạt động người dùng. Mục tiêu phân tích data là để hiểu hành vi và xu hướng người dùng trên Saleforce.

Plugin trên Github : https://github.com/developerforce/elf_elk_docker

ELF là cho phép dowload Event Log File để lấy index và hiển thị hóa dữ liệu bằng Kibana. ELF sử dụng Elasticsearch, Logstash và Kibana.

#### 1.2. CERN (European Organization for Nuclear Research)

Ở CERN, Elastic Stack được dùng cho các việc sau :
 
  - Messaging
  - Data monitoring
  - Cloud benchmarking
  - Infrastructure monitorng 
  - Job monitoring
  
  Nhiều biểu đồ Kibana được dùng bởi CERN cho 1 số hiển thị
  
#### 1.3 Green Man Gaming

Green Man Gaming là nền tảng online gaming nơi mà các nhà cung cấp game phát hành game của họ. Họ sử dụng Elastic Stack để thực hiện việc phân tích log, tìm kiếm và phân tích dữ liệu game

Họ sử dụng các Kibana dashboard để hiểu rõ hơn về số lượng người chơi game, bằng các thông số về đất nước và tiền tiêu thụ của người chơi

## 2. Một số so sánh

Open source:
 - Graylog: Visit https://www.graylog.org/ for more information
 - InfluxDB: Visit https://influxdata.com/ for more information\
 
Others:
 - Logscape: Visit http://logscape.com/ for more information
 - Logscene: Visit http://sematext.com/logsene/ for more information
 - Splunk: Visit http://www.splunk.com/ for more information
 - Sumo Logic: Visit https://www.sumologic.com/ for more information
Kibana competitors:
 - Grafana: Visit http://grafana.org/ for more information
 - Graphite: Visit https://graphiteapp.org/ for more information
Elasticsearch competitors:
 - Lucene/Solr: Visit https://lucene.apache.org/ or http://lucene.apache.org/solr/ for more information
 - Sphinx: Visit http://sphinxsearch.com/ for more information

## 3. Cài đặt Elastic Stack

### 3.1 Cài đặt Java

JDK cần được cài đặt cho việc truy cập tới Elasticsearch. Oracle Java 8 (Oracle JDK 1.8.0_73 trở đi) nên được cài đặt, tương ứng với Elasticsearch version 5.0 trở đi.

#### 3.1.1. Cài đặt Java trên Ubuntu 14.04
Cài đặt Java 8 sử dụng terminal và apt package : 

Thêm Oracle Java PPA (personal package archive) tới apt repo :
```sh
 add-apt-repository -y ppa:webupd8team/java
```

Update apt package database bao gồm các latest file và cài đặt Oracle Java 8:
```sh
apt-get update -y
apt-get -y install oracle-java8-installer
```

Trong quá trình cài đặt, pop-up sẽ hiện lên và bạn cần chấp thuận license :

![ELK](/ManhDV/ELK/images/elk-01.png)

Sau khi cài đặt xong Java thành công, dùng câu lệnh sau để kiểm tra phiên bản Java :
```sh
java --version
```

#### 3.2. Cài đặt Elasticsearch

Dowload Elasticsearch 5.1.1 deb package : 
```sh
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-5.1.1.deb
```

Install deb package : 
```sh
dpkg -i elasticsearch-5.1.1.deb
```

**Chú ý** : Elasticsearch sẽ được install trong thư mục `/usr/share/elasticsearch`. Init script được đặt tại `/etc/init.d/elasticsearch`. Log file được đặt trong thư mục `/var/log/elasticsearch`.

Cấu hình Elasticsearch boot cùng hệ thống :
```sh
update-rc.d elasticsearch defaults 9510
```

Kiểm tra và start service Elasticsearch :
```sh
service elasticsearch status
service elasticsearch start
```

Kiểm tra elasticsearch hoạt động với port : `http:///localhost:9200` bằng web browser hoặc với command :
```sh
curl -X GET http://localhost:9200
```

#### 3.3. Cài đặt Kibana

 - Elasticsearch được cài đặt trên port 9200
 - Kibana chạy trên port 5601


Cài đặt Kibana trên Ubuntu 14.04

Trước khi cài đặt Kibana, kiểm tra OS là 64 bit hay 32 bit :
```sh
uname -m
```

Dowload Kibana 5.1.1 deb package :
	- Với OS 64 bit :
```sh
wget https://artifacts.elastic.co/downloads/kibana/kibana-5.1.1-amd64.deb
```

	- Với OS 32 bit :
```sh
dpkg -i kibana-5.1.1-amd64.deb
```

 - Cài đặt deb file 
	- Với OS 64 bit :
```sh
dpkg -i kibana-5.1.1-i386.deb
```

**Chú ý** : Kibana sẽ được cài đặt tại thư mục `/usr/share/kibana`. File cấu hình sẽ được đặt tại `/etc/kibana`. Init script đặt tại `/etc/init.d/kibana`. Log file đặt tại `/var/log/kibana`.

Cấu hình Kibana boot cùng hệ thống :
```sh
update-rc.d kibana defaults 9510
```

Kiểm tra trạng thái kibana và start service :
```sh
service kibana status
service kinana start
```

Kiểm tra Kibina với web : `http://localhost:5601`
 
### 3.4. Cài đặt Logstash

Cài đặt Logstash trên Ubuntu 14.04

Dowload package deb Logstash 5.1.1 
```sh
wget https://artifacts.elastic.co/downloads/logstash/logstash-5.1.1.deb
```

Cài đặt package
```sh
dpkg -i logstash-5.1.1.deb
```

**Chú ý** : Logstash sẽ được cài đặt tại thư mục `/usr/share/logstash`. File cấu hình sẽ được đặt tại `/etc/logstash`. Init script đặt tại `/etc/init.d/logstash`. Log file đặt tại `/var/log/logstash`.

Kiểm tra trạng thái logstash và start service :

```sh
service logstash status
service logstash start
```

Cài đặt Logstash trên Windows:

 - Dowload logstash :
```sh
https://artifacts.elastic.co/downloads/logstash/logstash-5.1.1.zip
```

Giải nén file và click vào file cài đặt trong thư mục `bin`
 
### 3.5. Cài đặt Filebeat

 - Cài đặt Filebeat trên Ubuntu 14.04

Dowload package

 - Với OS 64 bit 
```sh
wget https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-5.1.1-amd64.deb
```

 - Với OS 32 bit
```sh
wget https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-5.1.1-i386.deb
```

Cài đặt package

 - Với OS 64 bit
```sh
dpkg -i filebeat-5.1.1-amd64.deb
```

 - Với OS 32 bit
```sh
dpkg -i filebeat-5.1.1-i386.deb
```

Cấu hình filebeat boot cùng hệ thống :
```sh
update-rc.d filebeat defaults 95 10
```

Kiểm tra trạng thái logstash và start service :
```sh
service filebeat status
service filebeat start
```

 - Cài đặt Filebeat trên Windows :

Dowload file cài đặt : 
 
 - Với OS 64bit :
```sh
https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-5.1.1-windows-x86_64.zip
```

 - Với OS 32bit : 
```sh
https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-5.1.1-windows-x86.zip
```

Giải nén file và mở command, cd tới thư mục dowload chứa file cài đặt và thực hiện câu lệnh :
```sh
.\install-service-filebeat.ps1
```

**Chú ý** : Nếu script execution bị tắt trên windows, cần đặt policy để thực hiện chạy script :
```sh
PowerShell.exe -ExecutionPolicy UnRestricted -File .\install-servicefilebeat.ps1.
```

#### 3.6. Cài đặt X-Pack

Đi cùng với Elastic Stack, một số tính năng cần được đi kèm như : security, monitoring, alerts... X-pack bao gồm 5 tính năng sau : 

 - Security
 - Alerts
 - Monitoring
 - Graphs
 - Reporting
 
Security. alert, monitoring được thể hiện dưới 3 cái tên : Shield, Watcher và Marvel.

### 4. Elasticsearch

#### 4.1. 5 tính năng nổi bật

 - Just give JSON : Elasticsearch lấy các tài liệu có dạng JSON làm input. Tất cả các thuộc tính của trường được tự động phát hiện và lập index theo mặc định. Elasticsearch tạo các ánh xạ (các string theo mặc định và có thể thay đổi được)
 - RESTful API : Với RESTfull API, khi sử dụng dữ liệu JSON thì có thể thực hiện được hành động quan trọng. Bạn có thể gửi document JSON để thêm vào index, delete, update một entry...
 - Real-time Data avaibility và analytics : Dữ liệu càng được index sớm thì việc phân tích và search càng nhanh khả dụng.
 - Distributed: Elasticsearch cho phép tạo multi node theo nhu cầu. Để mở rộng cluster chỉ cần thêm các node vào cluster.
 - High available : Cluster đủ thông minh để phát hiện ra node mới hoặc node bị fail để thêm/xóa khỏi cluster. Một node càng sớm được thêm hoặc xóa, dữ liệu được cân bằng lại theo cách vốn có.
 - Safety of your data come first : Bất kỳ thay đổi nào trên dữ liệu được ghi lại vào log transaction (thực hiện trên multiple node trong cluster). Điều này giúp giảm thiểu tối đa dữ liệu bị mất.
 - Multitenacy : Trên Elasticsearach, một alias cho index được tạo. Thông thường một cluster chứa nhiều index. Các alias này cho phép một góc nhìn đã được chỉnh sửa của một index để có thể đạt mutitenancy
 
#### 4.2. Kiến trúc của Elasticsearch

![elk](/ManhDV/ELK/images/elk-02.png)

**Index** chứa một hoặc nhiều **type**. Một type có thể hình dung như là một table đối với database. Một type có thể có một hoặc nhiều **documents**. Một document có thể có một hoặc nhiều **field**. Field là cặp key value.

Cluster có thể có 1 hoặc nhiều node và được định danh bởi tên. Tên mặc định của cluster là `elasticsearch`. Nếu không cung cấp tên, node ID  sẽ được chỉ định với UUID.

Một Index có thể lưu một lượng lớn dữ liệu và có thể vượt ngưỡng của phần cứng. Trong trường hợp đó, một index có thể chia thành nhiều shard.

Có 2 loại shard - primary và replica. Với mỗi document, khi được index, nó đầu tiên sẽ được thêm vào primary shard và sau đó được thêm tới một hoặc nhiều replica shard. Nếu setup nhiều hơn 1 node cho cluster, replica shard sẽ ở trên nhiều node.

Mặc định, Elasticsearch tạo 5 primary shard và 1 replica shard cho mỗi primary shard. Vì vậy, với mỗi index, nếu không chỉ định, tổng cộng sẽ có 10 shard được tạo. 

#### 4.3. Cấu hình khuyến cáo cho cluster

 - Nên cung cấp cluster name và tên riêng biệt cho mỗi node. VD :
```sh
cluster.name: my-cluster-name
node.name: node-1
```

 - Số master node tối thiểu
Đề phòng TH dữ liệu bị mất khi có sự cố, cluster có thể chọn 1 node master khác trong khi master node đầu tiên vẫn up, nhưng không thể hoạt động. Điều này có thể dẫn tới hiện tượng `split brain`, đề phòng tình huống này, nên setup số master node là `(Tổng số node/2) + 1 `

VD với số node trong cluster là 4 thì nếu setup giá trị :
```sh
discovery.zen.minimum_master_nodes: 3
```

Tất cả các node nên setup giá trị :
```sh
node.master: true
```

**Chú ý** : Không nên setup nhiều Elasticsearch instance trên cùng 1 máy local server.

#### 4.4. Document processing

Mỗi khi index được tạo, các shard và replica shard được tạo. Mỗi shard có thể có nhiều replica. Mỗi complete group được biết như replication group. Trong replication griup, primary shard hoạt động như là 1 entry point cho bất kỳ hành động đánh index cho document. Primary shard đảm bảo rằng document và hành động là hợp lệ. Nếu mọi thứ ok, hành động đông dánh index được thực hiện và sau đó primary shard replica cùng hành động đó trên tất cả các replica shard. Tất cả trách nhiệm thuộc về primary shard.

Không quan trọng rằng một document có được replicated tới tất cả replica shard; thay vào đó Elasticsearch duy trì một bản copy về những người nhận hành động. Danh sách này được gọi là `in-sync`, được giữ bởi master node. Bằng việc giữ danh sách này, master node đảm bảo rằng hành động đánh index được thực hiện bởi các shard và người dùng được công nhận. Processing model này được gọi là **data-replication model**, dựa trên **primary-backup model**

Trong trường hợp 1 primary shard fail, node chứa primary shard gửi message tới master để báo cáo việc này. Trong khoảng thời gian này không hành động đánh index được thực hiện và tất cả các shard phải chờ cho master xác định primary shard ra khỏi replicas.

Master node cũng kiểm tra trạng thái của tất cả các shard và nó có thể quyết định demote 1 primary shard dựa trên `poor health` (có thể do network fail hoặc bị disconnect). Master sẽ hướng dẫn các node khác bắt đầu dựng lên 1 shard mới để hệ thống có thể được restore về trạng thái khỏe mạnh là `green value`.

### 5. Logstash

Logstash có thể phân tích được bất kỳ định dạng nào của data : event data, timestamped data, app logs, transcational data, CSV data, file input... Logstash thu thập dữ liệu từ nhiều hệ thống về một hệ thống trung tâm nơi dữ liệu có thể được phân tích và xử lý, dữ liệu được thu thập với 1 format chung, có thể được dùng bởi Elasticsearch và Kibana

Logstash thực hiện **Extract**, **Transform**, và **Load** (ETL)

#### 5.1. Key feature của Logstash 

 - Opensource : Logstash hoàn toàn free và là một opensource tool với source code hoàn toàn khả dụng và miễn phí trên Github
 - Đi thành cùng bộ với Elasticsearch, Beats, và Kibana : Pipeline processing data mạnh mẽ có thể thu thập dữ liệu từ nhiều hệ thống sử dụng Beat, lưu trữ dữ liệu cho việc real-time search trên Elasticsearch, và hiển thi dữ liệu được lưu với Kibana.
 - Khả năng mở rộng : Logstash cung cấp đa dạng các input, filter, và output cho việc xử lý log với nhiều định dạng khác nhau. Nó cung cấp khả năng linh hoạt để tạo và phát triển các input, filter hoặc output cho Logstash.
 - Khả năng tương tác : Logstash cung cấp khả năng tương tác với nhiều thành phần và tool. Logstash có thể nhận data từ nhiều tool khác nhau và output data tới nhiều tool.
 - Kiến trúc data pipeline dạng pluggable : Logstash chứa hơn 200 plugins được phát triển bởi Elasticsearch và cộng đồng. Logstash được thiết kế như một framework thông thường nơi bạn có thể mix và match input hoặc filer hoặc output và nó sẽ xử lý dữ liệu tương ứng. Nó sử dụng một format của file cấu hình cho nhiều plugin đa dạng.

#### 5.2. Kiến trúc Logstash

Logstash pipeline bao gồm input, filter và output plugin. Tuy nhiên trong mô hình thông thường, Logstash chỉ nhận data input mà không tiến hành chỉnh sửa dữ liệu, model thông thường như sau :

![elk](/ManhDV/ELK/images/elk-03.png)
 
#### 5.3. Cấu trúc file cấu hình logstash

File cấu hình của logstash chứa 3 section của các dạng plugin chúng ta muốn dùng. VD như sau :
```sh
input {
}
filter {
}
output {
}
```

Mỗi section chứa cấu hình của 1 hoặc nhiều plugin. Để cấu hình plugin, cung cấp tên plugin bên trong section, chỉ định các value pair. Với mỗi section, nếu bạn cấu hình multi plugin, thì trình tự xuất hiện sẽ tuân theo thứ tự trong file cấu hình.

**Chú ý**

Ký hiệu **=>** là một toán tử gán giá trị cho khóa trong file cấu hình.

Trước khi đi sâu vào các plugin, hãy hiểu về các value type, định nghĩa như value cài đặt trong file cấu hình, và cách đặt điều kiện trong cấu hình.

#### 5.3.1. Value types

Mỗi khi Logstash chứa set cài đặt đã được dùng. Một số setting bắt buộc phải chỉ định để đánh dấu theo các trường bắt buộc. Đối với mỗi thiết lập, chúng ta định nghĩa một giá trị, tương ứng với các kiểu giá trị khác nhau được hỗ trợ bởi Logstash. 

 - Array
 
Tập hợp của một hoặc nhiều giá trị.
Với một giá trị đơn, cú pháp như sau : 
```sh
Key => "value"
```

Với nhiều giá trị, cú pháp như sau :
```sh
Key => ["value1","value2","value3"]

Nếu bạn chỉ định cùng một thiết lập nhiều lần trên một mảng, giá trị xuất hiện sẽ như sau :
```sh
Key => "value1"
Key => "value2"
Key =>["value2","value3","value4"]

Key chứa 5 giá trị : value, value1, 2, 3, 4.

 - Boolean

Giá trị được dùng là `true` hoặc `false`

VD : 
```sh
Key => true
Key1 => false
```

Chú ý : Giá trị của 1 Boolean type không cần đặt trong dấu ""/

 - Byte
 
Một byte là một trường dạng chuỗi (kèm với dấu ""), dùng để đại diện một đơn vị của byte. Nó được dùng bởi cả International System of Units (SI Units) (kB, MB hoặc GB) và binary unit (KiB, MiB hoặc GiB) để tính toán byte. Nó được dùng để định giá các giá trị theo đơn vị. Trong một số trường hợp, có thể chấp nhận dấu cách giữa giá trị của key và đơn vị. Tương tự, SI unit được dựa trên - 1000, trong khi các giá trị binary dựa trên - 1024. Ví dụ : 1 kB = 1000 bytes, trong khi 1 KiB = 1024 bytes.

Ví dụ : 
```sh
size => "2467KiB"
size => "9872miB"
Key => "452 GB"
```

Chú ý, nếu không chỉ định unit, thì value sẽ đại diện số byte :
```sh
Key => "1234"
```

 - Codec
 
Codec không phải là value type nhưng được dùng để thể hiện data. Nó được dùng để giải mã dữ liệu tới và mã hóa dữ liệu trước khi tới output. Sử dụng Codec giúp không cần phải có bộ lọc để xác định loại dữ liệu : 
```sh
codec => "plain"
```

 - Comment
 
Dùng để mô tả trong file cấu hình. Cú pháp như sau :
```sh
Key => "value" #It is string value type
#Hope you are learning
```

 - Hash

1 hash là một tổ hợp của key-value pair, trong đó cả key và value đều trong dấu "". Các entry của key-value không được ngăn cách bởi dấu phẩy, mà được ngăn cách bởi dấu cách :
```sh
match => {
"field1" => "value1"
"field2" => "value2"
...
}
```

 - Number

Một number phải chứa giá trị nguyên hoặc giá trị thập phân :
```sh
number => 44
amount => 1.28
```

 - String
 
String chứa một giá trị có thể được chứa trong dấu '' hoặc "". Nếu một string value chứa cùng dấu "", cần có dấu escape với dấu / :
```sh
name => "aaagw"
escape => "value"eu"
single => 'Hello It's nice to see you'
```

#### 5.3.2. Sử dụng Conditional

Conditional được dùng để kiểm tra các điều kiện dựa trên hành động nào được thực hiện. Trong TH của các file cấu hình, chúng ta kiểm tra điều kiện trên plugin dựa trên cấu hình được dùng. Nó kiểm soát cách tương tự như cách ngôn ngữ lập trình khác :
```sh
if EXPRESSION {
...
} else if EXPRESSION {
...
} else {
...
}
```

Các expression chứa các operator như các comparison operator, boolean operator và unary operator. Comparison operator được chia nhỏ thành : "equality operator", "regex operator", "inclustion operator" :

 - Equality operator : Chứa chuỗi các operator sau : ==, !=, <, >, <= và >=
 - Regex operator : Chứa list : =~ và !`
 - Inclusion operator : Chứa list : in và not in
 - Boolean operator : Chứa list : and, or, nand và xxor
 - Unary operator : Chứa giá trị : !
 
#### 5.3.3. Khai thác các plugin input

 - stdin
 
 - file

File plugin là một trong những plugin thông dụng nhất để lấy input từ 1 file với cơ chế tương tự như `tail -f`. Plugin còn cho biết địa điểm cuối nơi file được đọc, gửi các file update với data, phát hiện file rotaion và cung cấp các option để đọc file từ đầu hoặc cuối file.

Nó sẽ giữ thông tin về vị trí gần nhất nơi nhận dữ liệu từ các file trong một file gọi là `sincedb`. Mặc định, nó được đặt tại thư mục `$HOME`, nhưng vị trí có thể thay đổi với thiết lập `sincedb_path`. Khoảng thời gian theo dõi file có thể thay đổi bằng việc sử dụng cấu hình `sincedb_write_interval`.

Cấu hình cơ bản của cho plugin `file` như sau :
```sh
file {
path => ...
}
```

Trong plugin này, chỉ có cấu hình `path` là bắt buộc.

 - path
 
Chỉ định địa điểm thư mục hoặc filename nơi file được đọc. Bạn có thể thêm tên thư mục, tên của file, hoặc filename patter để plugin tự tìm. Có thể có một hoặc nhiều pattern/location.

Chú ý : 

Các cấu hình thiết lập thêm vào như sau : 

	- add_field : Dùng để thêm 1 field tới data đầu vào
	- close_older : Dùng để đóng input file nếu nó được chỉnh sửa nhiều hơn trong cấu hình chỉ định. Cấu hình này giải phóng các hành động file I/O và kiểm tra nhiều lần file để xem file có bị thay đổi không. Nếu file vẫn được tail và không có dữ liệu vào và khiến các cấu hình chi định tới giới hạn, file đó sẽ được đóng để các file khác có thể được mở. Nó sẽ được queue và mở lại khi file được update hoặc thay đổi.
	- codec : Dùng để mã hóa các incoming data và để thông dịch format của data được coi như là input.
	- delimiter : Dùng như dấu phân tách để xác định các dòng khác nhau.
	- discover_interval : Dùng để xác định tần suất `path` được mở rộng để tìm kiếm các tập tin mới được tạo bên trong cấu hình path.
	- enable_metric : Dùng để nhận metric value từ từng plugin cho việc báo cáo của 1 plugin.
	- exclude : Dùng để loại trừ bất kỳ file hoặc file pattern không được đọc như một input.
	- id : Dùng để cung cấp UUID tới 1 plugin , có thể được dùng để theo dõi thông tin của một plugin và có thể được dùng cho việc debug.
	- ignore_older : Dùng để từ chối việc đọc file mà không được chỉnh sửa từ mốc thời gian được nhắc đến trong cấu hình. Tuy nhiên nếu file được chỉnh sửa hoặc update với nội dung mới thì file vẫn được đọc :
	- max_open_file : Nó được dùng để định nghĩa số lượng file tối đa được mở trong 1 thời điểm. Nếu nhiều hơn số file quy định được đọc, option này sẽ đóng file mà không được chỉnh sửa muộn nhất. Setting này có thể gây ra các vấn đề về hiệu năng với OS.
	- sincedb_path : Dùng để định nghĩa file location cho việc ghi `sincedb` file cho việc theo dõi các file log 
	- sincedb_write_internval : Chỉ định khoảng thời gian mà `sincedb` file sẽ được ghi, bao gồm vị trí đọc gần nhất cho việc theo dõi nhiều log file.
	- start_position : Dùng để định nghĩa vị trí đọc file từ ban đầu hay ở cuối file. Nó chỉ được thiết lập khi file được đọc lần đầu và không có entry trong `sincedb` file. Nếu file của bạn chứa các dữ liệu cũ hơn, bạn có thể đọc file bằng việc chỉ định vị trí đọc là đọc từ lúc ban đầu.
	- tags : Dùng để thêm tag vào các incoming data, phần lớn được dùng bởi `conditional`, giúp thực hiện một chuối hành động với mỗi tag.
	- type : dùng để thêm trường `field` cho các incoming data. Rất hữ dụng khi bạn có incoming data từ nhiều nguồn khác nhau . Nó được dùng để lọc thông tin theo nguồn. 
 
Các value type và default value cho các cấu hình như sau :


|Setting|Value type|Default value|
|-------|----------|-------------|
|add_field |Hash |{}|
|close_older |Number |3600|
|codec |Codec |"plain"|
|delimiter |String |"\n"|
|discover_interval |Number |15|
|enable_metric |Boolean |true|
|exclude |Array |No default value|
|id |String |No default value|
|ignore_older |Number |No default value|
|max_open_files |Number |No default value|
|path |Array |No default value|
|sincedb_path |String |$HOME/.sincedb*|
|sincedb_write_interval |Number |15|
|start_position |String |"end"|
|stat_interval |Number |1|
|tags|Array |No default value|
|type |String |No default value|

Cấu hình file :
```sh
input {
	file {
		path => ["/var/log/elasticsearch/*","/var/messages/*.log"]
		add_field => ["[location]", "%{longitude}" ]
		add_field => ["[location]", "%{latitude}" ]
		exclude => ["*.txt]
		start_position => "beginning"
		tags => ["file-input"]
		type => "filelogs"
		}
	}
```

File cấu hình chỉ ra các file cần đọc trong thư mục `elasticsearch` và tất cả các file có đuôi `.log`. Giả sử file chứa `latitude` và `longtitude` và thêm trường `location`. Các text file bị loại trừ.

Cấu hình tiếp theo có thể chia thành 2 trường với các giá trị `path` sẽ khác nhau :
```sh
input {
file {
path => "/var/log/elasticsearch/*"
tags => ["elasticsearch"]
type => "elasticsearch"
}
file {
path => "/var/messages/*.log"
tags => ["messages"]
type => "message"
}
}
```

 - UDP
 
#### 5.3.4 Khai thác các filter plugin

 - grok

Grok plguin được dùng để filter trong Logtash. Grok có thể tái cấu trúc cảu dữ liệu theo cách bạn muốn. Nó tổ hợp các text pattern thành một cấu trúc giúp bạn ánh xạ log và nhóm chúng thành các trường. Logstash đi kèm 120 grok pattern, hoặc bạn có thể tạo pattern của riêng mình để match với grok.

Grok pattern có thể tham khảo tại : https://github.com/logstash-plugins/logstash-patterns-core/tree/master/patterns.

Cú pháp của grok pattern như sau : 
```sh
%{SYNTAX:SEMANTIC}
```

`SYNTAX` là tên của pattern để ánh xạ dữ liệu và `SEMANTIC` là định danh hoặc tên trường cung cấp với pattern được ánh xạ.

Với grok, bạn có thể dùng regex hoặc tạo các pattern tùy chỉnh. Thư viện regex được dùng là Oniguruma, cú pháp regex tham khảo tại : https://github.com/kkos/oniguruma/blob/master/doc/RE. BẠn có thể tạo ra 1 pattern file chứa các pattern ánh xạ với dữ liệu. Brok ánh xạ dữ liệu từ trái sang phải và ánh xạ pattern từng cái một.

VD : 
file log như sau :
```sh
Log line: Jun 19 02:11:30 This is sample log.
```

Sử dụng grok pattern có sẵn để tạo một grok pattern dùng để ánh xạ cho các dòng log phía trước :
```sh
%{CISCOTIMESTAMP:timestamp} %{GREEDYDATA:log}
```

Grok pattern trên có thể hiểu như sau : 
```sh
CISCOTIMESTAMP %{MONTH} +%{MONTHDAY}(?: %{YEAR})? %{TIME}
GREEDYDATA .*
```

Tham khảo 2 website sau để hiểu về cách xây dựng pattern để ánh xạ dữ liệu cần lấy :
	- http://grokdebug.herokuapp.com/
	- http://grokconstructor.appspot.com/
	
Cấu hình cơ bản cho grok như sau :
```sh
grok {
}
```

Trong plugin này thì không setting nào là bắt buộc. Các cấu hình thiết lập như sau : 
	- add_field : Dùng để thêm trường vào dữ liệu vào
	- add_tag : Thêm tag vào dữ liệu vào. Tag có thể là tĩnh hoặc động dựa vào từ khóa bên trong dữ liệu vào.
	- break_on_match : Dùng để thoát việc tìm kiếm cho các các filter hoặc pattern nếu nó được đặt là `true`. Nó sẽ kết thúc filter trên các ánh xạ của pattern. Nếu đặt là `false`, nó sẽ match với grok pattern.
	- keep_empty_capture : Dùng để giữ các event field (nếu bị trống nuế đặt là `true`)
	- match : Dùng để ánh xạ field với value. Value được cung cấp như là một pattern hoặc multiple pattern (không thể cung cấp như 1 array)
	- named_captures_only : Dùng để giữa chỉ các trường được xác định bằng cách sử dụng grok pattern, nếu đặt bằng `true`.
	- overwrite : Dùng để ghi đè value của trường.
	- patterns_dir : Dùng để nhắc đến thư mục bạn tạo pattern tùy chỉnh. Có thể nhắc đến 1 hoặc nhiều thư mục.
	- patterns_files_glob : Dùng để chọn tất cả các file từ thư mục chỉ định.
	- periodic_flush : Dùng để gọi tới `flush` method trên khoảng thời gian cụ thể.
	- remove_field : Dùng để remove 1 field từ dữ liệu đầu vào.
	- remove_tag :  Dùng để remove 1 tag từ dữ liệu đầu vào.
	- tag_on_failure : Dùng để tạo 1 tag với các message failure nếu event không match với grok pattern hoặc không pattern thành công việc ánh xạ với value.
	- tag_on_timeout : Dùng để tạo tag nếu grok regex nhận timeout
	- timeout_millis : Dùng để xóa regex nếu nó nhận time được chỉ định trong cài đặt này. Nó tương ứng với mỗi pattern nếu có nhiều pattern cho grok. Được tính theo milli s

Giá trị theo bảng sau :

|Setting |Value type| Default value|
|--------|----------|--------------|
|add_field |Hash |{}|
|add_tag |Array |[]|
|break_on_match |Boolean |true|
|keep_empty_captures |Boolean |false|
|match |Hash |{}|
|named_captures_only |Boolean |true|
|overwrite |Array |[]|
|patterns_dir |Array |[]|
|patterns_files_glob| String |"*"|
|periodic_flush| Boolean |false|
|remove_field| Array |[]|
|remove_tag| Array |[]|
|tag_on_failure| Array |["_grokparsefailure"]|
|tag_on_timeout| String |"_groktimeout"|
|timeout_millis| Number |2000|

Cấu hình ví dụ : 
```sh
filter {
grok {
add_field => {"current_time" => "%{@timestamp}" }
match => { "message" => "%{CISCOTIMESTAMP:timestamp} %{HOST:host}
%
{WORD:program}: \[%{NUMBER:duration}\] %{GREEDYDATA:log}" }
remove_field => ["host "]
remove_tag => ["grok","test"]
}
}
```

Nếu pattern không match, nó sẽ gán một tag `_grokparsefailure`. Nó cũng sẽ remove trường `hosst` từ message.

#### 5.3.6. Khai thác output plugin

Plugin `elasticsearch` được dùng để gửi output từ Logstash tới Elasticsearch, nơi dữ liệu được lưu trữ. Cấu hình cơ bản của elasticsearch như sau :
```sh
elasticsearch {
}
```

Trong plugin này, không setting nào là bắt buộc . Các setting như sau : 
	- action : Dùng để thể hiện nhiều hành động trên document được lưu trên Elasticsearch. Các operation đa dạng như : `index` (dùng để index một document), `delete` (dùng để xóa 1 document dựa trên ID), create (dùng để tạo và index một document với ID là duy nhất) và `update` (dùng để update document dựa trên ID).
	- cacert : Dùng để cung cấp path của `.cer` hoặc `.pem` file, dùng để chứng thực tới các chứng chỉ server cho các truy cập an toàn.
	- codec : Dùng để mã hóa dữ liệu trước khi gửi đi như là output
	- doc_as_upsert : dùng để bật update mode cho mỗi document. Khi ở `upsert` mode, nếu 1 document chứa 1 ID đã được thực hiện, nó sẽ update value của ID. Nếu document ID không tồn tại, nó sẽ tạo document với ID đó.
	- document_id : được dùng để chỉ định ID cho document. Thông thường, nó được tăng một cách tự động.
	- document_type : Dùng để cung cấp `type` nơi mà document được lưu trữ. Một `index` có thể chứa nhiều type. Để gửi output tới type chỉ định hoặc type khác, thì cần có thiết lập này.
	- hosts : Dùng để chỉ định host IP để liên lạc với Elastiscsearch node. Nó sẽ chỉ định host nào nên được kết nối để gửi dữ liệu. Bạn có thể chỉ định một hoặc nhiều host 1 lần. Port mặc định là 9200.
	- index : Được dùng để chỉ định tên của index mà data sẽ được ghi vào. Index name có thể là tĩnh hoặc động khi nguồn gốc có thể tới từ một trường giá trị hoặc sử dụng regex.
	- path : Được dùng khi bạn sử dụng một proxy để kết nói tới Elasticsearch node. Với lựa chọn này bạn có thể chỉ định `path` nơi mà Elasticsearch khả dụng 
	- proxy : Được dùng để chỉ định địa chỉ của proxy trong khi kết nối tới Elasticsearch node.
	 - ssl : Được dùng để bật **Secured Socket Layer (SSL)** transmisstion hoặc ** Transport Layer Security (TLS)**, sẽ tạo một kênh kết nối an toàn tới Elasticsearch cluster.
	
Chú ý :

Không bao gồm các master node chuyên dụng của Elasticsearch trong thuộc tính host, nếu không nó sẽ gửi dữ liệu đầu vào tới master node

Các giá trị của các setting :
|Setting |Value type |Default value|
|--------|-----------|-------------|
|action |String |''index''|
|cacert |Filesystem path |No default value|
|codec |Codec |''plain''|
|doc_as_upsert |Boolean |false|
|document_id| String| No default value|
|document_type |String |No default value|
|hosts| Array |["127.0.0.1"]|
|index |String |logstash-%{+YYYY.MM.dd}|
|path| String| /|
|proxy| <<,>> |No default value|
|ssl| Boolean |No default value|

Cấu hình ví dụ :
```sh
output {
elasticsearch {
cacert => "/usr/share/logstash/cert.pem"
doc_as_upsert => true
document_type => "elasticsearch"
hosts => ["localhost:9200","127.0.0.3:9201"]
index => "logstash"
ssl => true
}
}
```

Trong cấu hình tiếp theo, chúng ta sẽ chỉ định certificate path và bật tính năng `upsert` cho document. `type` sẽ được đề cập đến đến để lưu trữ output data. Chúng ta cũng sẽ chỉ định node trong Elasticsearch cluster để kết nối và ghi output data bên trong `index` Logstash.

### 5.4. Sử dụng Plugin Logstash

Phần này sẽ hướng dẫn cách list/install/update/remove một plugin bằng cách sử dụng câu lệnh với `logstash-plugin` script, được đặt trong thư mục `bin` của Logstash :

Cú pháp sử dụng cmd `plugin` như sau : 
```sh
bin/logstash-plugin [OPTIONS] SUBCOMMAND [ARG]
```

Các sub-command có thể dùng là : list, install, remove, update, pack, unpack, và generate.

 - List các plugin

Để list tất cả các plugin trong Logstash :
```sh
bin/logstash-plugin list
```

Hiển thị các option có thể có :
```sh
bin/logstash-plugin list --help
```

Các option dùng đc là :  --installed, --verbose, --group NAME.

Để tìm plugin đã cài đặt sử dụng tên : 
```sh
bin/logstash-plugin list kafka
```

Hiển thị version của plugin đã cài đặt :
```sh
bin/logstash-plugin list --verbose logstash-input-kafka
```

Hiển thị các plugin đã cài đặt thành 1 group (Input, Filter, Output và Codec)
```sh
bin/logstash-plugin list --group filter
```

 - Cài đặt Plugin

Hiển thị các option khả dụng khi cài đặt có thể view với câu lệnh :
```sh
bin/logstash-plugin install --help
```

Các option có là : -version, --[no-]verify, --preserve, --development, và --local.

Để cài đặt 1 plugin, sử dụng câu lệnh sau :
```sh
bin/logstash-plugin install logstash-filter-dissect
```

Chú ý :
Sử dụng quyền root để thực hiện câu lệnh.

Để cìa đặt 1 version đặc biệt, sử dụng câu lệnh sau :
```sh
bin/logstash-plugin install --version 1.0.8 logstash-filter-dissect
```

Để xác định 1 plugin là hợp lệ trước khi cài đặt, sử dụng câu lệnh :
```sh
bin/logstash-plugin install --verify logstash-filter-dissect
```

Để cài đặt 1 plugin mà không cần xác định là hợp lệ hay không trước khi cài đặt, sử dụng câu lệnh :
```sh
bin/logstash-plugin install --no-verify logstash-dilter-dissect
```

Để cài đặt plugin local, sử dụng câu lệnh sau :
```sh
bin/logstash-plugin install --local logstash-filter-dissect
```

 - Remove 1 plugin
 
```sh
bin/logstash-plugin remove logstash-filter-dissect
```

 - Update 1 plugin
 
Xem các option có thể có với câu lệnh update :
```sh
bin/logstash-plugin update --help
```

Các option là : --[no-]verify và --local

Để update tất cả các plugin : 
```sh
bin/logstash-plugin update
```

Để update 1 plugin chỉ định :
```sh
bin/logstash-plugin update logstash-filter-dissect
```

Để xác định 1 plugin là hợp lệ trước khi update, sử dụng câu lệnh :
```sh
bin/logstash-plugin update --verify logstash-filter-dissect
```

Để cài đặt 1 plugin mà không cần xác định là hợp lệ hay không trước khi update, sử dụng câu lệnh :
```sh
bin/logstash-plugin update --no-verify logstash-filter-dissect
```

Để update 1 plugin local :
```sh
bin/logstash-plugin update --local logstash-filter-dissect
```

 - Đóng gói 1 plugin
 
Các option có thể có trong việc đóng gói 1 plugin :
```sh
bin/logstash-plugin pack --help
```

Các option : --tgz, --zip, --[no-]clean, và --overwrite.

Đẻ đóng gói 1 plugin với GZipped Tar format :
```sh
bin/logstash-plugin pack --tgz
```

Để đóng gói plugin với ZIP format :
```sh
bin/logstash-plugin pack --zip
```

Để xóa một dump đã tạo của plugin : 
```sh
bin/logstash-plugin pack --clean
```

Để không xóa một dump đã tạo của plugin : 
```sh
bin/logstash-plugin pack --no-clean
```

Để overwrite 1 package file đã tạo trước đó :
```sh
bin/logstash-plugin pack --overwrite
```

 - Giải nén package
 
Giải nén file GZipped TAR :
```sh
bin/logstash-plugin unpack --tgz filename
```

Giải nén fie ZIP :
```sh
bin/logstash-plugin unpack --zip filename
```

### 5.4. Logstash command line tool

Các option sử dụng Logstash :
```sh
bin/logstash --help
```

 - Hiển thị tên của Logstash instance :
```sh
bin/logstash --node.name NODENAME
```

 - Để chạy Logstash với file cấu hình hoặc thư mục chứa file cấu hình :
```sh
bin/logstash -f CONFIGPATH
```

hoặc :
```sh
bin/logstash --path.config CONFIGPATH
```

Để chạy logstash với 1 cấu hình đặc biệt :
```sh
bin/logstash -e "input { stdin { type => stdin } }"
```

Để chỉ định 1 số lượng worker để thực hiện đồng thời : 
```sh
bin/logstash –w 12 or bin/logstash –-pipeline.workers 12
```

Số mặc định là 8.

Để chỉ định số lượng tối đa event của 1 worker sẽ kết nối trước khi thực hiện việc filter và output plugin, sử dụng câu lệnh sau :
```sh
bin/logstash -b 50 or bin/logstash --pipeline.batch.size 50
```

Số mặc định là 125

Để chỉ địuh thời gian chờ trước khi kiểm tra 1 event mới (delay theo ms):
```sh
bin/logstash -u 10
```

hoặc :
```sh
bin/logstash --pipeline.batch.delay 10
```

Giá trị mặc định là 250 ms.

Để buộc Logstash thoát để tắt thậm chí nó chứa các event trong RAM :
```sh
bin/logstash --pipeline.unsafe_shutdown
```

Chỉ định  thư mục Logstash chứa dữ liệu : 
```sh
bin/logstash --path.data PATH
```

Giá trị mặc định là `$LS_HOME/data`

Chỉ định đường dẫn để tìm thư mục plugin trên Logstash :
```sh
bin/logstash -p PATH or bin/logstash --path.plugins PATH
```

Chú ý :

`PATH` được đinh nghĩa `PATH/logstash/TYPE/NAME.rb`, với `TYPE` là group name (INPUT, OUTPUT, FILTER VÀ CODEC) và `NAME` là tên của plugin.

Để ghi log trong 1 thư mục cho việc chạy Logstash :
```sh
bin/logstash -l PATH or bin/logstash --path.logs PATH
```

Để chỉ định log level cho log của Logstash :
```sh
bin/logstash --log.level LEVEL
```

Các log LEVEL : fatal, warn, error, debug, info (default value) và trace.

Để print `config` ruby code như debug log :
```sh
bin/logstash --config.debug
```

Chú ý : Để dùng --config.debug thì --log.level=debug phải đc bật.

Để hiển thị version của Logstash:
```sh
bin/logstash -V or bin/logstash --version
```

Để xác định rằng file cấu hình Logstash là đúng :
```sh
bin/logstash -f file.conf --config.test_and_exit or bin/logstash -f file.conf -t
```

Chú ý : Grok pattern không được xác thực. Chỉ cú pháp hợp lệ mới được cho phép. Để bật chế độ auto-reload của Logstash config file : 
```sh
bin/logstash -f file.conf -r or bin/logstash -f file.conf --config.reload.automatic
```

Giá trị mặc định là `false`

Để chỉ định khoảng thời gian reload cho việc kiểm tra file cấu hình Logstash thay đổi :
```sh
bin/logstash -f file.conf --config.reload.interval 5
```

Giá trị mặc định là 3s.

Để chỉ định thư mục chứa file cấu hình cho Logstash :
```sh
bin/logstash --path.settings SETTINGS_DIR
```

Giá trị mặc định là `$LS_HOME/config`

### 5.5. Các thủ thuật với Logstash

#### 5.5.1. Các trường và giá trị của nó

Trong file cấu hình logstash, bạn có thể tham khảo một trường bằng tên của nó và có thể sau đó đẩy value của 1 trưởng vào bên trong trường khác. Nếu bạn muốn tham khảo tới `top-level field`, sử dụng tên trường trực tiếp. Nếu bạn muốn tham khảo tới `nested field`, sử dụng cú pháo`[top-level field][nested field]`. Để tham khảo trường, Logstash sử dụng `sprintf` format, giúp người dùng tham khảo được giá trị của trường.

`sprintf` format như sau :
```sh
{[top-level field][nested field].....}
```

VD :
```sh
output {
elasticsearch {
document_type => "%{@version}"
index => "logstash_%{type}_%{+YYYY-MM-dd-H}"
}
}
```

`index` name không thể chứa các ký tự sau : \, /, *, ?, ", <, >, |, hoặc ,. Đồng thời `index` name phải viết bằng chữ thường.

#### 5.5.2. Thêm 1 grok pattern tùy chỉnh

Cú pháp của grok pattern như sau :
```sh
%{SYNTAX:SEMANTIC}
```

Cú pháp cơ bản để định nghĩa 1 pattern trong 1 grok file sử dụng regex như sau :
```sh
PATTERNNAME Regular-Expression_Syntax
```

VD : 
```sh
ALPHANUMERIC ([a-zA-Z0-9-]+)
```

Cú pháp cơ bản khi định nghĩa 1 pattern trong 1 grok file sử dụng 1 grok pattern có sẵn như sau :
```sh
PATTERNNAME %{EXISTING_GROK-PATTERN)
```

VD :
```sh
RECORDTIME %{TIME}
```

#### 5.5.3. Logstash không show bất kỳ output nào 
Đây là 1 vấn đề thông thường khi sử dụng Logstash. Trong TH này, Logstash đã chạy nhưng chúng ta không thể thấy bất kỳ output nào được in ra. Tình huống này có thể xảy ra ở 1 vài hoàn cảnh như sau :

#### 5.5.3.1. Khi input đã được đọc hoàn toàn
Trong tình huống này, chúng ta sử dụng `input` plugin như 1 file và có 1 file được quy định hoặc các file để đọc dữ liệu như là file input để tạo `sincedb` file, giúp Logstash biết có bao nhiêu input file được thực hiện và thông qua. Trong tình huống này, nếu input file được đọc một cách hoàn toàn, sau đó thực hiện chạy Logstash lần nữa với tập hợp các input file sẽ không show ra bất kỳ giá trị nào.

Nhìn vào trong `sincedb` file :
```sh
948594 0 2049 47484
```

Hàng này là inode (Unix) hoặc identifier (Windows), số thiết bị chính (major device numbers) , số thiết bị phụ (minor device numbers) và byte offset.

Với hàng inode, với major và minor device numbers giúp biết được file được đọc bởi `file` input plugin. Byte offset định nghĩa số lượng byte đọc trong file.

Có 2 solution để giải quyết : 
	- Solution 1 : Xóa file `sincedb` tại `$HOME/.sincedb_*` và chạy lại Logstash.
	- Solution 2 : Sử dụng thiết lập hoặc tham số của `sincedb_path` như sau : `sincedb_path => "/dev/null"` (Solution này ko đc khuyến khích vì khi Logstash restart, file sẽ được thông qua từ lúc khởi đầu, dẫn đến dữ liệu bị trùng lặp)
	
#### 5.5.3.2. Khi input file không được sửa từ 1 ngày
Chúng ta sử dụng input như 1 file và chỉ định 1 file như là input. Khi input file không được chỉnh sửa trong 86400 (1 ngày), sau đó Logstash sẽ chạy nhưng không hiển thị output.

Các solutiond dể giải quyết : 
	- Solution 1 : Chỉnh sửa file để có thể đọc bởi Logstash
	- Solution 2 : Sử dụng `touch` để chỉnh sửa timestamp. Chỉ tác dụng nếu `sincedb` path là `/dev/null`
	
### 5.6. Logstash Config cho việc phân tích log

Log file của Tomcat và Cataline log chứa app exeotion, error và stack trace message. Log file chứa log event với nhiều log level : s INFO, WARN, ERROR, DEBUG, và FATAL.

Cataline log :
```sh
Mar 10, 2016 10:04:37 PM org.apache.catalina.startup.Catalina load INFO:
Initialization processed in 433 ms
Mar 10, 2016 10:04:37 PM org.apache.catalina.core.StandardService
startInternal INFO: Starting service Catalina
```

Tomcat log :
```sh
2016-03-10 22:04:40,892 INFO localhost-startStop-1
support.AnnotationConfigWebApplicationContext:208 - Registering annotated
classes: [class matrix.api.config.RESTConfig]
2016-03-10 22:05:07,248 ERROR http-bio-8080-exec-1
handler.ExceptionHandlerAdvice:57 - We have encountered an internal error.
org.springframework.dao.EmptyResultDataAccessException: Incorrect result size:
expected 1, actual 0
at org.springframework.dao.support.DataAccessUtils (Support.java:71)
at org.springframework.jdbc.core.Jdbc.queryForObject(Jdbc.java:489)
at org.impl.HealthCheckDaoImpl.getResult(HealthCheckDaoImpl.java:30))
at org.springframework.web.Handler.invokeForRequest(Handler.java:132)
```

Tạo 1 grok pattern để ánh xạ Catalina và Tomcat log. Tạo 1 file tên là `grok pattern` bên trong `/usr/share/logstash/patterns`