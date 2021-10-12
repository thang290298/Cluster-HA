<h1 align="center">Các thành phần cấu tạo nên HAProxy</h1>

# Cấu trúc HAProxy
- Đường dẫn `file` cấu hình HAProxy được lưu lại `/etc/haproxy/haproxy.cfg` có cấu trúc như sau:
```
global
    # Các thiết lập tổng quan

defaults
    # Các thiết lập mặc định

frontend
    # Thiết lập điều phối các request

backend
    # Định nghĩa các server xử lý request
```
- Các thiết lập tại `global` được sử dụng để định nghĩ cách HAProxy được khởi tạo như số lượng kết nối tối đa, đường dẫn log, số process...
- Sau đó các thiết lập tại mục `Default` sẽ được thực hiện áp dụng cho tất cả các mục `frontend`,`backend` nằm phía sau ( hoàn toàn có thể định nghĩa lại các giá trị mặc định `frontend` và `backend`). Có thể có nhiều mục `frontend`, `backend` được định nghĩa trong file cấu hình.
- Mục `Frontend` được định nghĩa để thục hiện điều hướng tới các request nhận được tới  các `backend`
- Mục `backend` sử dụng để định nghĩa các danh sách máy chủ dịch vụ (có Web server, Database, …) đây là nơi `request` được xử lý.

## 1. Global
- Được đặt trên cùng file cấu hình HAProxy, được định danh bằng từ khóa `global` và mục này luôn đứng riêng 1 dòng và được định nghĩa 1 lần duy nhất trong file cấu hình. Các thiết lập bên dưới global định nghĩa các thiết lập bảo mật, các điều chỉnh về hiệu năng áp dụng trên toàn HAProxy (áp dụng tại mức tiến trình HAProxy hoạt động)
```sh
global
    log         127.0.0.1 local2
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon
    stats socket /var/lib/haproxy/stats
```
- `maxconn`: Chỉ định giới hạn số kết nối mà HAProxy có thể thiết lập. Sử dụng với mục đích bảo vệ load balancer khởi vấn đề tràn ram sử dụng.
- `log`: Bảo đảm các cảnh báo phát sinh tại HAProxy trong quá trình khởi động, vận hành sẽ được gửi tới syslog.
- `stats socket`: Định nghĩa `runtime api`, có thể sử dụng để `disable server` hoặc `health checks`, thấy đổi `load balancing weights` của server. [Xem thêm](https://www.haproxy.com/blog/dynamic-configuration-haproxy-runtime-api/)

- `user / group` chỉ định quyền sử dụng để khởi tạo tiến trình HAProxy. Linux yêu cầu xử lý bằng quyền root cho nhưng port nhở hơn 1024. Nếu không định nghĩa user và group, HAProxy sẽ tự động sử dụng quyền root khi thực thi tiến trình.

## 2. Default
- Cấu hình tăng dần, phức tạp , khó đọc, các thiết lập cấu hình tại mục `default` giúp giảm các trùng lặp. thiết lập `default` sẽ áp dụng cho cả `frontend và backend` nằm sau nó. Hoàn toàn có thể thiết lập lại từng mục `backend` và `frontend`
- Ví dụ, có thể thiết lập `mode http` tại mục `default`, khi đó toàn bộ các mục `frontend, backend listen` sẽ đều dùng `mode http` làm mặc định.

```sh
mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option                  http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    option  http-server-close
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000
```
- `timeout connect` chỉ định thời gian HAProxy đợi thiết lập kết nối TCP tới `backend server`. Hậu tố `s` tại `10s` thể hiện khoảng thời gian 10 giây, nếu bạn không có hậu tố `s`, khoảng thời gian sẽ tính bằng `milisecond` - `ms` - mili giây. [Xem thêm](https://www.haproxy.com/documentation/hapee/latest/onepage/#4-timeout%20client)

- `timeout server` chỉ định thời gian chờ kết nối tới `backend server`.

**Một số lưu ý:**

- Khi thiết lập `mode tcp` thời gian `timeout server` phải bằng `timeout client`.

- `log global`: Chỉ định ‘frontend’ sẽ sử dụng log settings mặc định (trong mục `global`).

- `mode`: Thiết lập `mode` định nghĩa HAProxy sẽ sử dụng TCP proxy hay HTTP proxy. Cấu hình sẽ áp dụng với toàn `frontend` và `backend` khi bạn chỉ mong muốn sử dụng 1 mode mặc định trên toàn `backend` (Có thể thiết lập lại giá trị tại `backend`).

- `maxconn`: Thiết lập chỉ định số kết nối tối đa, mặc định bằng `2000`.

- `option httplog`: Bổ sung format log dành riêng cho các request http bao gồm (connection timers, session status, connections numbers, header v.v). Nếu sử dụng cấu hình mặc định các tham số sẽ chỉ bao gồm địa chỉ nguồn và địa chỉ đích.

- `option http-server-close`: Khi sử dụng kết nối dạng keep-alive, tùy chọn cho phép sử dụng lại các đường ống kết nối tới máy chủ (có thể kết nối đã đóng) nhưng đường ống kết nối vẫn còn tồn tại, thiết lập sẽ giảm độ trễ khi mở lại kết nối từ phía client tới server.

- `option dontlognull`: Bỏ qua các log format không chứa dữ liệu.

- `option forwardfor`: Sử dụng khi mong muốn `backend server` nhận được IP thực của người dùng kết nối tới. Mặc định backend server sẽ chỉ nhận được IP của HAProxy khi nhận được request. Header của request sẽ bổ sung thêm trường `X-Forwarded-For` khi sử dụng tùy chọn.

- `option redispatch`: Trong mode HTTP, khi sử dụng kỹ thuật `stick session`, client sẽ luôn kết nối tới 1 backend server duy nhất, tuy nhiên khi backend server xảy ra sự cố, có thể client không thể kết nối tới backend server khác (Trong bài toán load balancer). Sử dụng kỹ thuật cho phép HAProxy phá vỡ kết nối giữa client với backend server đã xảy ra sự cố. Đồng thời, client có thể khôi phục lại kết nối tới backend server ban đầu khi dịch vụ tại backend server đó trở lại hoạt động bình thường.

- `retries`: Số lần thử kết nối lại backend server trước khi HAProxy đánh giá backend server xảy ra sự cố.

- `timeout check`: Kiểm tra thời gian đóng kết nối (chỉ khi kết nối đã được thiết lập)

- `timeout http-request`: Thời gian chờ trước khi đóng kết nối HTTP.

- `timeout queue`: Khi số lượng kết nối giữa client và haproxy đạt tối đã (maxconn), các kết nối tiếp sẽ đưa vào hàng đợi. Tùy chọn sẽ làm sạch hàng chờ kết nối.


## 3. Frontend

Mục `frontend` định nghĩa địa chỉ IP và port mà client có thể kết nối tới. Bạn có thể có nhiều mục frontend tùy ý, chỉ cần đặt label của chúng khác nhau (`frontend <tên>`)

```
frontend www.mysite.com
    bind 10.0.0.3:80
    bind 10.0.0.3:443 ssl crt /etc/ssl/certs/mysite.pem
    http-request redirect scheme https unless { ssl_fc }
    use_backend api_servers if { path_beg /api/ }
    default_backend web_servers
```

- `bind`: IP và Port HAProxy sẽ lắng nghe để mở kết nối. IP có thể `bind` tất cả địa chỉ sẵn có hoặc chỉ 1 địa chỉ duy nhất, port có thể là một port hoặc nhiều port (1 khoảng hoặc 1 list).

- `http-request redirect`: Phản hỏi tới client với đường dẫn khác. Ứng dụng khi client sử dụng http và phản hồi từ HAProxy là https, điều hướng người dùng sang giao thức https.

- `use_backend`: Chỉ định backend sẽ xử lý request nếu thỏa mãn điều kiện (Khi sử dụng ACL).

- `default_backend`: Backend mặc định sẽ xử lý request (Nếu request không thỏa mẵn bất kỳ điều hướng nào)

## 4. Backend

Mục `backend` định nghĩa tập server sẽ được cân bằng tải khi có các kết nối tới (VD tập các server chạy dịch vụ web giống nhau).

```
backend web_servers
    balance roundrobin
    cookie SERVERUSED insert indirect nocache
    option httpchk HEAD /
    default-server check maxconn 20
    server server1 10.10.10.86:80 cookie server1
    server server2 10.10.10.87:80 cookie server2
```

- `balance`: Kiểm soát cách HAProxy nhận, điều phối request tới các backend server. Đây chính là các thuật toán cân bằng tải.

- `cookie`: Sử dụng cookie-based. Cấu hình sẽ khiến HAProxy gửi cookie tên `SERVERUSED` tới client, liên kết `backend server` với client. Từ đó các request xuất phát từ client sẽ tiến tục nói chuyện với server chỉ định. Cần bổ sung thêm tùy chọn `cookie` trên server line.

- `option httpchk`: Với tùy chọn, HAProxy sẽ sử dụng `health check` dạng HTTP (Layer 7) thay vì tìm kiếm trả kết nối dạng TCP (Layer 4). Và khi server không phản hồi request http, HAProxy sẽ thực hiện TCP check tới IP Port. `Health check` sẽ tự động loại bỏ các backend server lỗi, khi không có backend server sẵn sàng xử lý request, HAProxy sẽ trả lại phản hồi 500 Server Error. Mặc đinh HTTP check sẽ kiểm tra root path / (Có thể thay đổi). Và nếu phản hồi health check là 2xx, 3xx sẽ được coi là thành công.

- `default-server`: Bổ sung tùy chọn cho bất kỳ backend server thuộc `backend` section (VD: health checks, max connections, v.v). Điều này kiến cấu hình dễ dàng hơn khi đọc.

- `server`: Tùy chọn quan trọng nhất trong `backend` section. Tùy chọn đi kèm bao gồm `tên`, `IP:Port`. Có thể dùng domain thay cho IP. Theo ví dụ, tùy chọn `maxconn 20` sẽ được bổ sung vào tất cả backend server, tức mỗi server sẽ chỉ phục vụ 20 kết nối đồng thời.
