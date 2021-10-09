<h1 align="center">High Availability: Tìm hiểu về cách quản lý các resource trong pacemaker</h1>

## 1. Tạo resource

- Để tạo mới một resource trong cluster, sử dụng câu lệnh với cú pháp:
```sh
pcs resource create	resource_id(name) standard:provider:type|type [resource options]
```
- Ví dụ:
```
pcs resource create Virtual_IP ocf:heartbeat:IPaddr2 ip=10.10.13.30 cidr_netmask=24 op monitor interval=30s
```
  - Trong đó:
    - pcs resource create: yêu cầu khởi tạo resource
    - Virtual_IP: tên resource hoặc resource_id
    - ocf:heartbeat:IPaddr2: khai báo kiểu resource agent
    - ip=10.10.13.30, cidr_netmask=24: các giá trị của resource
    - op monitor interval=30s: Khai báo các tùy chọn cho resource

- Khi khởi tạo resource các giá trị mặc định được gán là `standard=ocf` và `provider=heartbeat` nên vẫn có thể sử dụng câu lệnh sau đây để tạo resource
```sh
pcs resource create Virtual_IP IPaddr2 ip=10.10.13.30 cidr_netmask=24 op monitor interval=30s
```
Sau khi tạo mới một resource, nó ngay lập tức sẽ được hoạt động.


- Để xóa một resource, sử dụng cú pháp
```
pcs resource delete resource_id
```
ví dụ: 
```sh
pcs resource delete Virtual_IP
```

Câu lệnh sẽ tiến hành xóa đi resource hay resource_id có tên là Virtual_IP

## 2. Các tính chất của resource

- Các tính chất quy định của 1 resource nhằm thông báo cho Cluster rằng kịch bản (resource agents) nào sẽ được sử dụng cho resource, gồm:
  - `resource_id`: Tên của resource
  - `standard`: Tiêu chuẩn của kịch bản (resource agents) tuân thủ theo
  - `provider`: Khai báo lớp của resource agents (nhà cung cấp kịch bản này)
  - `type`: Tên của resource agents muốn sử dụng được cung cấp bởi provider

- Liệt kê danh sách resource:
```sh
pcs resource list
```

## 3. Các tham số cụ thể về resource

- Để biết được các tham số cụ thể của resource thuộc một kiểu nào đó, sử dụng câu lệnh như sau:
```sh
pcs resource describe resource_type
```

- Ví dụ: 
```sh
pcs resource describe ocf:heartbeat:nginx
```
kết quả:
```
ocf:heartbeat:nginx - Manages an Nginx web/proxy server instance

This is the resource agent for the Nginx web/proxy server.
This resource agent does not monitor POP or IMAP servers, as
we do't know how to determine meaningful status for them.

The start operation ends with a loop in which monitor is
repeatedly called to make sure that the server started and that
it is operational. Hence, if the monitor operation does not
succeed within the start operation timeout, the nginx resource
will end with an error status.

The default monitor operation will verify that nginx is running.
```

## 4. Các tùy chọn cho resource

- Ngoài các tham số cụ thể cho resource có thể thiết lập thêm các giá trị tùy chọn cho resource theo các giá trị như sau:

| Meta options | Default | Mô tả |
|--------------|-------|------|
| priority | 0| Nếu không phải tất cả các tài nguyên đều có thể hoạt động, cluster sẽ dừng các resource có độ ưu tiên thấp để giữ lại các resource có độ ưu tiên cao đang hoạt động|
| target-role | Started| Quy định việc cố gắng giữ resource trong node nào đó. Bao gồm các giá trị: <br> <ul><li>Stopped: Buộc resource phải dừng hoạt động</li><li>Started: Cho phép resource sẽ được khởi động</li><li>Master: Cho phép resource sẽ được khởi động nếu phù hợp với node nào đó</li></ul>|
| is-managed | true | Là cluster cho phép khởi động hoặc dừng resource. Giá trị cho phép: true, false |
| resource-stickiness | 0 | Giá trị thể hiện có bao nhiêu resource được chỉ định ở tại nơi mà resource này đang hoạt động |
| migration-threshold | INFINITY(disabled) | Có bao nhiêu lỗi có thể xảy ra cho resource này trong một node trước khi node này được xác định là node không có đủ điều khiện để lưu trữ resource |
| multiple-active | stop_start | Quy định những gì mà cluster sẽ thực hiện khi phát hiện một resource đang chạy đồng thời trên nhiều node khác nhau. Các giá trị cho phép như sau: <br> <ul><li>block - quy định resource không được quản lý. Sẽ tạm dừng hoạt động resource. Đưa ra cảnh báo </li><li>stop_only: Dừng tất cả các trường hợt hoạt động</li><li>stop_start: Dừng tất cả các trường hợp hoạt động và thử khởi động lại</li></ul> |

- Thực hiện thay đổi giá trị mặc định:
```sh
pcs resource defaults resource-stickiness=69
```
- Có thể thiết lập giá trị cho các meta-options này khi đã tạo hoặc tạo mới một resource bằng 1 trong 2 cú pháp câu lệnh sau:
  - Khi tạo mới một resource:
  ```
  pcs create resource_id type value meta meta_options
  ```
Ví dụ:
```sh
pcs resource create  Virtual_IP ocf:heartbeat:IPaddr2 ip=10.10.13.30 cidr_netmask=24 meta resource-stickiness=69
```
  - Đối với resource đã tồn tại trước đó
```sh
cs resource meta resource_id meta_options
  ```
  Ví dụ:
  ```sh
pcs resource meta Virtual_IP resource-stickiness=69
```

## 5. Các nhóm resource

- Một điều quan trọng trong và phổ biến `cluster` là một tập các resource cần được cài đặt ở cùng một vị trí trên một node nào đó và được khởi động một cách tuần tự và ngược lại đối với việc tắt đi. Để thực hiển đơn giản hóa cấu hình này dẫm đếm khái niệm nhóm (Group) trong Pacemaker xuất hiện
- Bạn có thể tạo ra một resource group bằng câu lệnh khai báo các resource nằm bên trong group sẽ tạo. Nếu như một group mà bạn chuẩn bị tạo ra đã thự sự tồn tại trên hệ thống thì các resource sẽ được tiếp tục thêm vào group đã tồn tại đó. Ngược lại, thì một group mới sẽ được tạo ra. Cú pháp của câu lệnh tạo group này rất quan trọng bởi việc thứ tự khởi động các resource trong group phụ thuộc vào việc bạn viết câu lệnh này như thế nào. Đơn giản sẽ là, việc khai báo resource nào trước trong câu lệnh thì nó sẽ được khởi động trước và việc dừng lại sẽ hoạt động theo thứ tự ngược lại

- Cú pháp của câu lệnh như sau:
```sh
pcs resource group add group_name resource_id ... resource_id_n
```
 - Trong đó: 
   - group_name: Là tên group bạn sẽ định tạo ra
   - resource_id: Là tên của resource, resource này sẽ được khởi động đầu tiên ...
   - resource_id_n: Là rên của resource thứ n mà bạn muốn thêm vào group

- Thêm resource vào một group đã tồn tại khi tạo mới resource đó theo cú pháp:
```sh
pcs resource create resource_id type --group group_name
```
- Trong đó:
  - group_name: là tên group bạn muốn thêm resource chuẩn bị tạo vào đó.

- Để gỡ bỏ một resource trong group, ta sử dụng câu lệnh sau:
```sh
pcs resource group remove group_name resource_id
```

> ### Lưu ý: Nếu resource_id không tồn tại trong group_name thì group_name này sẽ bị xóa bỏ.

- Ví dụ: 
```sh
pcs resource group add Web Virtual_IP Web_Cluster
```

câu lệnh trên sẽ tạo ra một resource group có tên là `Web`. Và cách thức hoạt động cụ thể như sau:
- Resource có tên là `Virtual_IP` sẽ khởi động trước sau đó mới khởi động `Web_Cluster`
- Resource có tên là `Web_Cluster` sẽ dừng hoạt động trước sau đó mới dừng hoạt động của `Virtual_IP`

Vì vậy, nếu:
- `Virtual_IP` không thể chạy ở bất cứ trên node nào đó thì `Web_Cluster` cũng không thể hoạt động
- `Web_Cluster` không thể hoạt động ở bất cứ đâu thì `Virtual_IP` chưa chắc đã không hoạt động được

## 6. vận hành các resource 
- Để đảm các resource hoạt đông một các ổn định nhất, có thế thực hiện các hoạt động giám sát tới 1 định nghĩa của resource . Nếu không khai báo hoạt động giám sát cho 1 resource, mặc định nó sẽ được tạo ra với 1 khoảng thời gian lặp lại giám sát được quy định bởi resource agents. Nếu resource không cung cấp khoảng thời gian giám sát mặc định thì hoạt động giám sát được tạo ra tạo ra với khoảng thời gian là 60S
- Các thuộc tính hoạt động giám sát bao gồm

| Meta options | Mô tả |
|--------------|-------|
| id | Giá trị đại diện à là duy nhất cho hành động giám sát|
| name | Hành động thực thi, bao gồm: `monitor`,`start`,`stop`|
| interval | Quy định thời gian thực thi hành động giám sát. Default = 0 |
| timeout | Quy định thời gian chờ trước khi quyết đinh hoạt động giám sát bị lỗi. |
| on-fail | Hành động thực thi mà hoạt động giám sát luôn bị thất bại với các giá trị cho phép như sau:: <br><ul><li>`ignore`: Bỏ qua việc hoạt động bị lỗi. Nói các khác: xem như chưa xảy ra vấn đề gì</li><li>`Block`: Không thực hiện thêm bất kỳ hành động nào trên hệ thống</li><li>`Stop`: Dừng resource lại và không tiến hành khởi động trên bất kỳ node nào thuộc cluster</li><li>`Restart`: thực hiện khởi động lại resource</li><li>`Fence`: `STONITH` trên node mà resource đó bị lỗi </li><li>`Standby`: Di chuyển tất cả resource ra khỏi node có resource bị lỗi</li></ul> |
| enable| Nếu giá trị thiết lập là `false` thì hoạt động gián sát sẽ được xem như chưa hề tồn tại. Có thể có 2 giá trị là `true` hoặc `false`|

- Ví dụ:
  - Ta đã từng thực hiện tạo 1 `resource` như sau:
  ```sh
  pcs resource create Virtual_IP ocf:heartbeat:IPaddr2 ip=10.10.13.30 cidr_netmask=24 op monitor interval=30s
  ```
  - Tuy nhiên có thể thêm các tùy chọn ngay sau khi tạo resource :
  ```sh
  pcs resource op add resource_id operation_actions operation_properties
  ```
  - trong đó:
    - operation_actions: là các giá trị: `monitor`, `start`, `stop`
    - operation_properties: là các giá trị được cho theo bảng bên trên, ví dụ: `interval`, `fence`, ...
  - Có thể thiết lập các biến đó với giá trị mặc định mà mình đã quy định:
  ```sh
  pcs resource op defaults [options]
  ```
  - Ví dụ cụ thể
  ```sh
  pcs resource op defaults timeout=240s
  ```

## 7. Hiển thị cấu hình của resource

### Liệt kê danh sách tất cả **`Resource`**
```sh
pcs resource show
```
  -  Kết quả:
  ```sh
  [root@MariaDB-3 ~]# pcs resource show
  Virtual_IP     (ocf::heartbeat:IPaddr2):       Started MariaDB-1
  Loadbalancer_HaProxy   (systemd:haproxy):      Started MariaDB-1
  Virtual_IP2    (ocf::heartbeat:IPaddr2):       Stopped
  [root@MariaDB-3 ~]#
  ```
### Sử dụng câu lệnh sau để hiện thị tất cả những gì liên quan tới resource:
```sh
pcs resource show --full
```
  - Kết quả:
  ```sh
  [root@MariaDB-3 ~]# pcs resource show --full
  Resource: Virtual_IP (class=ocf provider=heartbeat type=IPaddr2)
  Attributes: cidr_netmask=24 ip=10.10.13.30
  Meta Attrs: resource-stickiness=69
  Operations: monitor interval=30s (Virtual_IP-monitor-interval-30s)
              start interval=0s timeout=20s (Virtual_IP-start-interval-0s)
              stop interval=0s timeout=20s (Virtual_IP-stop-interval-0s)
  Resource: Loadbalancer_HaProxy (class=systemd type=haproxy)
  Operations: monitor interval=5s timeout=5s (Loadbalancer_HaProxy-monitor-interval-5s)
              start interval=0s timeout=100 (Loadbalancer_HaProxy-start-interval-0s)
              stop interval=0s timeout=100 (Loadbalancer_HaProxy-stop-interval-0s)
  Resource: Virtual_IP2 (class=ocf provider=heartbeat type=IPaddr2)
  Attributes: cidr_netmask=24 ip=10.10.1.1
  Meta Attrs: resource-stickiness=69
  Operations: monitor interval=10s timeout=20s (Virtual_IP2-monitor-interval-10s)
              start interval=0s timeout=20s (Virtual_IP2-start-interval-0s)
              stop interval=0s timeout=20s (Virtual_IP2-stop-interval-0s)
  [root@MariaDB-3 ~]#
  ```

### Hiển thị cấu hình **`Resource`** chỉ định:
```sh
pcs resource show resource_id
```
  - Ví dụ:
  ```sh
  pcs resource show Virtual_IP
  ```
  - Kết quả:
  ```sh
  [root@MariaDB-3 ~]# pcs resource show Virtual_IP
  Resource: Virtual_IP (class=ocf provider=heartbeat type=IPaddr2)
  Attributes: cidr_netmask=24 ip=10.10.13.30
  Meta Attrs: resource-stickiness=69
  Operations: monitor interval=30s (Virtual_IP-monitor-interval-30s)
              start interval=0s timeout=20s (Virtual_IP-start-interval-0s)
              stop interval=0s timeout=20s (Virtual_IP-stop-interval-0s)
  [root@MariaDB-3 ~]#
  ```

## 8. Chỉnh sửa các tham số cụ thể của resource
### Cú pháp cập nhật cấu hình Resource
```sh
pcs resource update resource_id resource_options
```
  - Ví dụ:
```sh
pcs resource update Virtual_IP cidr_netmask=32
```
  - Kết quả:
```
 [root@MariaDB-3 ~]# pcs resource show Virtual_IP
 Resource: Virtual_IP (class=ocf provider=heartbeat type=IPaddr2)
  Attributes: cidr_netmask=32 ip=10.10.13.30
  Meta Attrs: resource-stickiness=69
  Operations: monitor interval=30s (Virtual_IP-monitor-interval-30s)
              start interval=0s timeout=20s (Virtual_IP-start-interval-0s)
              stop interval=0s timeout=20s (Virtual_IP-stop-interval-0s)
[root@MariaDB-3 ~]#
```
## 9. Kích hoạt, vô hiệu hóa nhóm các resource
- Để khởi động một resource
```sh
pcs resource enable resource_id
```
- Để dừng hoạt động của một resource
```sh
pcs resource disable resource_id
```
## 10. Xóa các cảnh báo của các resource
- Trong quá trình các `resource` hoạt động, đôi khi sẽ xuất hiện các cảnh báo lỗi. Và bạn muốn đặt lại trạng thái của nó thì dùng câu lệnh sau để đặt lại toàn bộ trạng thái của `resource` và `failcount` của resource:
```sh
pcs resource cleanup resource_id
```