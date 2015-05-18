# Grok-filter-logstash

#Define lại các log gửi đến sử dụng grok filter

## Mở đầu
**Mục đích**: Tách một trường trong bản tin log nhận được. Việc này nhằm phục vụ cho việc query và vẽ các biểu đồ được chính xác nhất.

Lấy một ví dụ cụ thể: Đây là bản tin log của ssh (***/var/log/auth.log***) mà server nhận được:

`Failed password for root from 54.179.13.13 port 39174 ssh2`

Mình chỉ muốn lấy mỗi trường **IP (54.179.13.13)** để đếm số lần IP này cố gắng tấn công dò mật khẩu ssh vào server mình. Do đó mình cần phải "define" lại bản tin đến thành nhiều trường riêng biệt như sau:

```sh
Failed password for uvdc from 172.16.69.1 port 59738 ssh2
=>
Failed %{WORD:auth_method} for %{USER:username} from %{IP:src_ip} port %{INT:src_port} ssh2

```

*Mẹo* Sử dụng trang này để kiểm tra `http://grokdebug.herokuapp.com/`

Kết quả:
Bản tin nhận được lúc đầu:

<img src="http://i.imgur.com/goVH1Am.png">

Bản tin nhận được sau khi được định dạng lại:

<img src="http://i.imgur.com/WfW18Vm.png">

Như đã thấy trên hình, mình đã tách được IP truy cập đến thành 1 trường riêng biệt là `src_ip`, port thành `src_port` ...

---

#Áp dụng để...

Định dạng lại bản tin log của nginx (/var/log/nginx/access.log), apache (/var/log/ 
apache2/access.log), ssh (/var/log/auth.log)

- Cài đặt chi tiết xem tại [đây](https://www.digitalocean.com/community/tutorials/how-to-install-elasticsearch-logstash-and-kibana-4-on-ubuntu-14-04). 

- Sau khi cài đặt thành công ta có:

**Tại server:**

- Logstash cài đặt ở `/opt/logstash`
- Logstash sử dụng port 5000 để nhận log từ logstash forwarder phía client
- Các file cấu hình logstash nằm ở `/etc/logstash/conf.d`
- Input file là `/etc/logstash/conf.d/01-lumberjack-input.conf`
- Output file là `/etc/logstash/conf.d/30-lumberjack-output.conf`

**Tại Client:**

- Cài đặt logstash forwarder để đẩy log đến server (file cấu hình `/etc/logstash-forwarder.conf`)

--- 
##Thực hiện thế nào?

1. Tạo Thư mục `patterns` tại **Server**

```sh
sudo mkdir -p /opt/logstash/patterns
sudo chown logstash:logstash /opt/logstash/patterns
```

---
#### 2. Với Nginx

- #####Tại **Client**

`sudo vi /etc/logstash-forwarder.conf`

```sh
{
  "network": {
        "servers": [ "172.16.69.210:5000" ],  # "servers": [ "IP_ELK_SERVER:PORT" ]
        "timeout": 15,
        "ssl ca": "/etc/pki/tls/certs/logstash-forwarder.crt" 
  },

  "files": [
{
      "paths": [
       "var/log/nginx/access.log"
       ],
      "fields": { "type": "nginx-access" }
    }
 ]
}

```

`sudo service logstash-forwarder restart`

- #####Tại **Server**

`sudo vi /opt/logstash/patterns/nginx`

```sh
NGUSERNAME [a-zA-Z\.\@\-\+_%]+
NGUSER %{NGUSERNAME}
NGINXACCESS %{IPORHOST:clientip} %{NGUSER:ident} %{NGUSER:auth} \[%{HTTPDATE:timestamp}\] "%{WORD:verb} %{URIPATHPARAM:request} HTTP/%{NUMBER:httpversion}" %{NUMBER:response} (?:%{NUMBER:bytes}|-) (?:"(?:%{URI:referrer}|-)"|%{QS:referrer}) %{QS:agent}

```

`sudo chown logstash:logstash /opt/logstash/patterns/nginx`

chú ý: **NGINXACESS** tức là pattern sẽ được sử dụng để "grok-filer" trong logstash

- Bây giờ tạo file filter cho nginx

`sudo vi /etc/logstash/conf.d/11-nginx.conf`

```sh
filter {
  if [type] == "nginx-access" {
    grok {
      match => { "message" => "%{NGINXACCESS}" }
    }
  }
}

```

`sudo service logstash restart`

***Chú ý*** Tại server: logstash hoạt động như sau `input => filer => output` tương đương với `01-lumberjack-input.conf =>  => 11-nginx.conf 30-lumberjack-output.conf`

---

### 3. Với APACHE:

- ##### Tại **Client**

Đẩy file apache `access.log` lên server

`sudo vi /etc/logstash-forwarder.conf`

```sh
{
  "network": {
        "servers": [ "172.16.69.210:5000" ],  # "servers": [ "IP_ELK_SERVER:PORT" ]
        "timeout": 15,
        "ssl ca": "/etc/pki/tls/certs/logstash-forwarder.crt" 
  },

  "files": [
{
      "paths": [
       "var/log/apache2/access.log"
       ],
      "fields": { "type": "apache-access" }
    }
 ]
}

```

`sudo service logstash-forwarder restart`

- #### Tại **SERVER**

`sudo vi /etc/logstash/conf.d/12-apache.conf`

```sh
filter {
  if [type] == "apache-access" {
    grok {
      match => { "message" => "%{COMBINEDAPACHELOG}" }
    }
  }
}
```

`sudo service logstash restart`

**Chú ý**: Mặc định Apache pattern đã là một mẫu có sẵn trong logstash nên không cần bước tạo pattern như với nginx


---

####4. Với SSH:

- ##### Tại **Client**

`sudo vi /etc/logstash-forwarder.conf`

```sh
{
  "network": {
        "servers": [ "172.16.69.210:5000" ],  # "servers": [ "IP_ELK_SERVER:PORT" ]
        "timeout": 15,
        "ssl ca": "/etc/pki/tls/certs/logstash-forwarder.crt" 
  },

  "files": [
{
      "paths": [
       "/var/log/auth.log"
       ],
      "fields": { "type": "ssh" }
    }
 ]
}

```

`sudo service logstash-forwarder restart`


- #####Tại **Server**

`sudo vi /opt/logstash/patterns/ssh`

```sh
SSHFAILED Failed %{WORD:auth_method} for %{USER:username} from %{IP:src_ip} port %{INT:src_port} ssh2

SSHACC Accepted %{WORD:auth_method} for %{USER:username} from %{IP:src_ip} port %{INT:src_port} ssh2

```

`sudo chown logstash:logstash /opt/logstash/patterns/ssh`

- Tạo file filter cho ssh

`sudo vi /etc/logstash/conf.d/13-ssh.conf`

```sh
filter {
  if [type] == "ssh" {
    grok {
      match => { "message" => "%{SSHFAILED}" }
      match => { "message" => "%{SSHACC}" }
    } 
  }
}

```

`sudo service logstash restart`

--- 

Tham khảo

- http://logstash.net/docs/1.4.2/filters/grok

- https://github.com/elastic/logstash/blob/v1.4.2/patterns/grok-patterns

- https://www.digitalocean.com/community/tutorials/adding-logstash-filters-to-improve-centralized-logging
