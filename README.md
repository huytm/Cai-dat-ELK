# Cai-dat-ELK

#Sử dụng ELK để thu thập log trên ubuntu 14.04
####A. Mở đầu 
Bài viết này sử dụng ***logstash 1.4.2*** và ***kibana3***. Logstash là một opensoure tool dùng để thu thập, phân loại và lưu trữ log để sử dụng trong tương lai. Kibana 3 là web interface dùng để tìm kiếm và hiển thị log. Cả 2 công cụ này đều sử dụng dựa vào ***elasticsearch***. Do đó Elasticsearch, Logstash, Kibana kết hợp với nhau tạo thành bộ công cụ ***ELK***

Log tập trung rất có lợi cho việc khai thác các thông tin về máy chủ hoặc ứng dụng của bạn. như đã nói ở trên, logstash là công cụ để thu thập log. Có nhiều cách để chuyển tiếp log từ phía client lên ELK server tuy nhiên trong bài viết này mình giới hạn sử dụng ***rsyslog*** thuần túy.

Tóm gọn lại như sau

- Logstash xử lý các log đến
- Elasticsearch lưu trữ tất cả các log
- Kibana Web interface để tìm kiếm và hiển thị log
- Logstash forwarder: Cài đặt phía client và sẽ gửi log đến logstash

Ba thành phần đầu tiên sẽ cài đặt trên 1 server, gọi là ELK server, Tại phía client sẽ sử dụng rsyslog để đẩy log đến server

Sử dụng ubuntu 14.04, lượng CPU, RAM và HDD phụ thuộc vào số lượng log mà máy chủ xử lý, trong bài viết này mình thu thập khoảng 1000 events/s nên RAM = 4Gb, CPU = 2

-----
###B.Bắt đầu cài đặt

####TẠI SERVER

#####1. Cài đặt các gói bổ trợ 

`sudo apt-get install python-software-properties software-properties-common -y`

#####2. Cài đặt JAVA 7
Elasticsearh và Logstash yêu cầu Java 7.

- add repo

```sh
sudo add-apt-repository ppa:webupd8team/java -y
echo debconf shared/accepted-oracle-license-v1-1 select true | sudo debconf-set-selections
echo debconf shared/accepted-oracle-license-v1-1 seen true | sudo debconf-set-selections
```

- Update packet

`sudo apt-get update`

- Install bản stable của Oracle Java 7

`sudo apt-get -q -y install oracle-java7-installer`

- Thiết lập môi trường

`sudo bash -c "echo JAVA_HOME=/usr/lib/jvm/java-7-oracle/ >> /etc/environment"`

- Kiểm tra lại

`java -version`

<img src="http://i.imgur.com/G1BgwkV.png">

#####2. Cài đặt Elasticsearch
**Note**: Logstash 1.4.2 yêu cầu sử dụng Elasticsearch 1.1.1

Để cài đặt Elasticsearch 1.1.1 ta thực hiện các bước sau

- import Elasticsearh public GPG key:

`wget -O - http://packages.elasticsearch.org/GPG-KEY-elasticsearch | sudo apt-key add -`

- Tạo file Elastisearch source list:

`echo 'deb http://packages.elasticsearch.org/elasticsearch/1.1/debian stable main' | sudo tee /etc/apt/sources.list.d/elasticsearch.list`

- Update packet:

`sudo apt-get update`

- Install Elasticsearch bằng câu lệnh

`sudo apt-get -y install elasticsearch=1.1.1`

- Cấu hình lại elasticsearch

`sudo vi /etc/elasticsearch/elasticsearch.yml`

- Khởi động elasticsearch
`sudo service elasticsearch restart`

- Để Elasticsearch khởi động cùng với server sử dụng lệnh sau

`sudo update-rc.d elasticsearch defaults 95 10`

----------
#####2. Cài đặt Kibana
**Note**: Logstash 1.4.2 khuyến cáo sử dụng kibana 3.0.1

- Cài đặt các gói cần thiêt

`apt-get install apache2 unzip -y`

- Download và extract kibana 3

```sh
cd ~; wget http://download.elasticsearch.org/kibana/kibana/kibana-latest.zip
unzip kibana-latest.zip
```

- Tạo root folder và khởi động kibana webserver

```sh
sudo mkdir -p /var/www/kibana
sudo cp -R ~/kibana-latest/* /var/www/kibana/
```

- Cấu hình kibana webserver

```sh
vi /etc/apache2/conf-enabled/kibana.conf
Alias /kibana /var/www/kibana
<Directory /var/www/kibana>
  Order allow,deny
  Allow from all
</Directory>
```

- Khởi động lại apache2

`sudo service apache2 restart`

- ***Cài đặt apache2-utils để xác thực user đăng nhập vào kibana***

```sh
sudo apt -y install apache2-utils
vi /etc/apache2/sites-available/auth-basic.conf
<Directory /var/www/kibana>
    AuthType Basic
    AuthName "Basic Authentication"
    AuthUserFile /etc/apache2/.htpasswd
    require valid-user
</Directory>
```

- Tạo user để đăng nhập ( tham số -c tức là add 1 user, nếu add user thứ 2 sẽ ghi đè lên user1. Muốn add nhiều user sử dụng tham số -b)

```sh
htpasswd -c /etc/apache2/.htpasswd admin
New password: # set password
Re-type new password: # confirm
Adding password for user admin
```

```sh
mkdir /var/www/html/auth-basic 
a2ensite auth-basic 
service apache2 reload
service apache2 restart
```


#####2. Cài đặt Logstash

- Tạo file logstash source list:

`echo 'deb http://packages.elasticsearch.org/logstash/1.4/debian stable main' | sudo tee /etc/apt/sources.list.d/logstash.list`

- Update packet

`sudo apt-get update`

- Cài đặt logstash

`sudo apt-get install -y logstash=1.4.1-1-bd507eb --force-yes`

- Cấu hình logstash

```sh
input {
  tcp {
    port => 9292
    type => syslog
  }
  udp {
    port => 9292
    type => syslog
  }
}
filter {
  if [type] == "syslog" {
    grok {
      match => { "message" => "%{SYSLOGTIMESTAMP:syslog_timestamp} %{SYSLOGHOST:syslog_hostname} %{DATA:syslog_program}(?:\[%{POSINT:syslog_pid}\])?: %{GREEDYDATA:syslog_message}" }
      add_field => [ "received_at", "%{@timestamp}" ]
      add_field => [ "received_from", "%{host}" ]
    }
    syslog_pri { }
    date {
      match => [ "syslog_timestamp", "MMM  d HH:mm:ss", "MMM dd HH:mm:ss" ]
    }
  }
}
output {
  elasticsearch { host => localhost }
  stdout { codec => rubydebug }
}
```

http://semicomplete.com/presentations/logstash-scale11x/images/tiered-outputs-to-inputs.jpg

Như đã nói ở các bài viêt trước, logstash xử lý log như sau: **input => filer  => output***

<img src="http://semicomplete.com/presentations/logstash-scale11x/images/tiered-outputs-to-inputs.jpg">

ở đây mình dùng input là syslog gửi qua tcp và udp sau đó putput vào elasticsearch để tiện cho việc lưu trữ và sử dụng sau này

-Khởi động lại logstash

`service logstash restart`

####Tại phía client

- Cấu hình rsyslog để đẩy log đến server

`vi /etc/rsyslog.d/50-default.conf`

- add dòng sau vào cuối file

```sh
*.*         @@logstashserver:port
Ví dụ với mô hình của mình
*.* 	    @@172.16.69.210:9292 
```

- Khởi động lại rsyslog

`sudo service rsyslog restart`
