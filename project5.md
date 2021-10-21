# Project 5: Dựng server Linux thành 1 Load-balancer cho nhiều server

Yêu cầu

* Tạo 2 Load-balancer chạy Haproxy (High availability cho nhau sử dụng Virtual IP)
* Quản lý VIP dùng keepalived
* Từ Client truy cập vào web, theo dõi trạng thái request vào ra thông qua WEB UI của Haproxy
* Test thử trường hợp 1 server Haproxy down
* Test thử trượng hợp 1 web server down
* Thay đổi log format và quản lý phần lưu log

### Mô hình LAB

![alt text](https://s3-ap-southeast-1.amazonaws.com/kipalog.com/sajsrfphe_Screenshot%202021-10-22%20020144.png)

* Mô hình gồm có 2 HAProxy Server và 2 Web Server
* Các Server đều sử dụng hệ điều hành CentOS 7
* HAProxy 1: 192.168.1.150
* HAProxy 2: 192.168.1.111
* Web Server 1: 192.168.1.43
* Web Server 2: 192.168.1.52
* VIP: 192.168.1.100  

### 1. Cài đặt HAProxy trên 2 server làm Load-balancer

#### Load-balacer 1

Cài đặt HAProxy package

` # yum install -y haproxy `

Chỉnh sửa file haproxy.cfg

` # vi /etc/haproxy/haproxy.cfg `

Thay thế dòng ` frontend  main *:5000 ` bằng dòng ` frontend  main *:80 `

Thêm vào 2 dòng sau ở phần backend app
```
server httpd1 192.168.159.43:80 check
server httpd2 192.168.159.52:80 check
```
![alt text](https://s3-ap-southeast-1.amazonaws.com/kipalog.com/wvj2tn963o_Screenshot%202021-10-22%20003605.png)

Thêm vào 5 dòng sau ở phần defaults để theo dõi HAProxy qua giao diện web
```
stats enable
stats auth admin:123456
stats uri /haproxy?stats
stats refresh 30s
stats show-node
```
![alt text](https://s3-ap-southeast-1.amazonaws.com/kipalog.com/jw1xerwaen_Screenshot%202021-10-22%20003646.png)

Start và enable HAProxy mỗi lần hệ thống boot lên
```
# systemctl enable haproxy
# systemctl start haproxy
```
Tắt SELinux và Firewalld

#### Load-balancer 2

Thực hiện cài đặt HAproxy tương tự như với Load-balancer 1

### 2. Cài đặt Keepalived để tạo Virtual IP

#### Load-balancer 1

Cài đặt gói Keepalived

` # yum install -y keepalived `

Chỉnh sửa file keepalived.conf

` # vi /etc/keepalived/keepalived.conf `

Sửa nội dung như sau, lưu lại và  thoát
```
vrrp_script chk_haproxy { 
script "killall -0 haproxy"		# check the haproxy process
interval 2					    # check every 2 seconds
weight 2 						# add 2 points to priority if OK
}

vrrp_instance VI_1 {
interface ens33				    # interface to monitor
state MASTER				    # MASTER on haproxy1, BACKUP on haproxy2
virtual_router_id 51
priority 101 				    # priority 101 # 101 on haproxy1, 100 on haproxy2

virtual_ipaddress {
192.168.1.100 				    # VIP
}

track_script {
chk_haproxy
}
}
```
Bổ sung cấu hình cho phép kernel có thể binding tới IP VIP, kiểm tra lại bằng lệnh ` # sysctl -p `

` # echo 'net.ipv4.ip_nonlocal_bind = 1' >> /etc/sysctl.conf `

Start và enable Keepalived mỗi lần hệ thống boot lên
```
# systemctl enable keepalived
# systemctl start keepalived
```
#### Load-balancer 2

Thực hiện cài đặt Keepalived tương tự như với Load-balancer 1, lưu ý file keepalived.conf

### 3. Cài đặt Apache trên 2 web server

#### Web server 1 

Cài đặt Apache

` # yum -y install httpd `

Khởi động và enable dịch vụ khi hệ thống boot lên
```
# systemctl enable httpd 
# systemctl start httpd
```
Tạo file index.html đơn giản
```
# echo "<center><h1>This is web1</h1></center>" > /var/www/html/index.html
```
Cấu hình firewall cho phép dịch vụ http
```
# firewall-cmd --permanent --zone=public --add-service=http
# firewall-cmd --reload
```
#### Web server 2

Thực hiện cài đặt tương tự như với Web server 1 

### 4. Kiểm tra kết quả 

#### Kiểm tra VIP trên LB1 và LB2

![alt text](https://s3-ap-southeast-1.amazonaws.com/kipalog.com/ji8jiwl8qh_Screenshot%202021-10-22%20004200.png)

![alt text](https://s3-ap-southeast-1.amazonaws.com/kipalog.com/uaba67uidd_Screenshot%202021-10-22%20004421.png)

>
VIP chỉ được gán vào interface ens33 của LB1 chứ không được gán vào LB2 dù ta đã khai báo trên cả 2. Lí do là ta khởi tạo LB1 giữ vai trò MASTER với priority 101, LB2 giữ vai trò BACKUP với priority 100. Giả sử HAProxy service trên LB1 không hoạt động, priority của LB1 sẽ bị trừ còn 98 (weight 2). Khi đó Keepalived sẽ chuyển LB2 từ BACKUP lên MASTER và LB2 sẽ nắm giữ VIP

Truy cập VIP trên trình duyệt ta tới được web server 1

![alt text](https://s3-ap-southeast-1.amazonaws.com/kipalog.com/4d3i6y112f_Screenshot%202021-10-22%20004539.png)

Refresh lại trang web ta tới được web server 2

![alt text](https://s3-ap-southeast-1.amazonaws.com/kipalog.com/bhlp8hxry0_Screenshot%202021-10-22%20004600.png)

>
Như vậy ta đã cấu hình Load-balancing cho 2 web server thành công với thuật toán Round-Robin. Khi LB1 down, ta vẫn có thể truy cập tới web server 1 và web server 2 nhờ LB2. Khi 1 web server down, ta vẫn có thể truy cập tới web server còn lại

#### Giao diện web của HAProxy

Truy cập ` # http://192.168.159.100/haproxy?stats `, nhập username và password đã thiết lập trong file haproxy.conf 

![alt text](https://s3-ap-southeast-1.amazonaws.com/kipalog.com/x0qeoy8tc9_Screenshot%202021-10-22%20004903.png)

Kết quả hiển thị giao diện web của HAProxy, tại đây ta có thể theo dõi trạng thái của các web server

![alt text](https://s3-ap-southeast-1.amazonaws.com/kipalog.com/1czntslpfp_Screenshot%202021-10-22%20005031.png)

Thử shutdown web server 2 và kiểm tra trạng thái trên giao diện web 

![alt text](https://s3-ap-southeast-1.amazonaws.com/kipalog.com/juuap95gpx_Screenshot%202021-10-22%20020535.png)

#### Quản lí log HAProxy 

Trên HAProxy Server 1, chỉnh sửa file rsyslog.conf ` # vi /etc/rsyslog.conf `

Thay đổi nội dung như sau, lưu lại và thoát
```
# Provides UDP syslog reception
$ModLoad imudp
$UDPServerAddress 127.0.0.1
$UDPServerRun 514
```
Tạo file haproxy.conf trong /etc/rsyslog.d, thêm nội dung như sau, lưu lại và thoát

` # vi /etc/rsyslog.d/haproxy.conf `

` if ($programname == 'haproxy') then -/var/log/haproxy.log `

Tạo file log HAProxy trong /var/log/haproxy.conf và cấp quyền
```
# touch /var/log/haproxy.log
# chmod 755 /var/log/haproxy.log
```
Khởi động lại rsyslog ' # systemctl restart rsyslog `

Bây giờ thử request 3 lần tới địa chỉ VIP và kiểm tra file log trong /var/log/haproxy.log

![alt text](https://s3-ap-southeast-1.amazonaws.com/kipalog.com/x74jocmxe7_Screenshot%202021-10-22%20021806.png)

Ở đây ta thấy log-format là **"%{+Q}r"** với Outputs **"GET / HTTP/1.1"**

Ta có thể change format log với nhiều kiểu khác

```
log-format "%{+X}Ts"     # Outputs: 5C5342A0

log-format "%{+E}HQ"     # Outputs: ?myparam=[some_brackets\]

log-format "%{+E,+Q}HQ"  # Outputs: "?myparam=[some_brackets\]"
```

