<h1 align="center">vận hành, xử lý sự cố pacemaker corosync
</h1>

# Phần I. Chuẩn bị môi trường
- Sử dụng tiếp kết quả bài lab pacemaker - haproxy -3 node - galera mariadb để thực hiện các case với  pacemaker corosync.
- Xem hướng dẫn trển khai cài đặt pacemaker - haproxy -3 node - galera mariadb [tại đây](https://github.com/thang290298/Cluster-HA/blob/main/02-LAB/02-Haproxy-Pacemaker-Cluster-Galera-3-node-Apache.md)
```sh
[root@MariaDB-1 ~]# pcs status
Cluster name: ha_cluster
Stack: corosync
Current DC: MariaDB-3 (version 1.1.23-1.el7_9.1-9acf116022) - partition with quorum
Last updated: Fri Oct 15 00:50:51 2021
Last change: Sun Oct  3 23:13:43 2021 by root via crm_resource on MariaDB-2

3 nodes configured
2 resource instances configured

Online: [ MariaDB-1 MariaDB-2 MariaDB-3 ]

Full list of resources:

 Virtual_IP     (ocf::heartbeat:IPaddr2):       Started MariaDB-3
 Loadbalancer_HaProxy   (systemd:haproxy):      Started MariaDB-3

Daemon Status:
  corosync: active/enabled
  pacemaker: active/enabled
  pcsd: active/enabled
[root@MariaDB-1 ~]#
```
# Phần II. Vận hành
## 1. PCS command line
- Ngừng hoạt động trên 1 node

```sh
pcs cluster stop <node ..>
```

- Ngừng hoạt động cả cluster

```sh
pcs cluster stop --all
```

- Không cho phép node khởi động cùng OS

```sh
pcs cluster disable <node ..>
```

- Không cho phép cluster khởi động cùng OS

```sh
pcs cluster disable --all
```

- Khởi động cluster trên 1 node

```sh
pcs cluster start <node ..>
```

- Khởi động cả cluster

```sh
pcs cluster start --all
```

- Cho phép node khởi động cùng OS

```sh
pcs cluster enable <node ..>
```

- Cho phép cluster khởi động cùng OS

```sh
pcs cluster enable --all
```

### 2.2. Bổ sung thêm node vào cluster

> **`Thực hiện trên node mới`**

```sh
+ Cài đặt cấu hình IP đúng theo mô hình kết nối.

+ Bổ sung vào /etc/host của node mới và các node cũ.

+ Cài đặt các dịch vụ giống như các node khác (database, source code, webserver...)
```

- Cài đặt gói pacemaker pcs

```sh
yum -y install pacemaker pcs

systemctl start pcsd 
systemctl enable pcsd
```

- Đặt mật khẩu `hacluster` giống các node thuộc cluster

```sh
[root@node4 ~]# passwd hacluster
Changing password for user hacluster.
New password:
Retype new password:
passwd: all authentication tokens updated successfully.
[root@node4 ~]#
```

```
systemctl start pcsd.service
systemctl enable pcsd.service
```

> **`Thực hiện trên các node cũ`**

- Chứng thực node mới

```sh
pcs cluster auth node4
```

```sh
[root@node1 ~]# pcs cluster auth node4
Username: hacluster
Password:
node4: Authorized
[root@node1 ~]#
```

- Bổ sung node vào cụm

```sh
pcs cluster node add node4
```

```sh
[root@MariaDB-1 ~]# pcs cluster node add node4
Disabling SBD service...
node4: sbd disabled
Sending remote node configuration files to 'node4'
node4: successful distribution of the file 'pacemaker_remote authkey'
MariaDB-1: Corosync updated
MariaDB-2: Corosync updated
MariaDB-3: Corosync updated
Setting up corosync...
node4: Succeeded
Synchronizing pcsd certificates on nodes node4...
node4: Success
Restarting pcsd on the nodes in order to reload the certificates...
node4: Success
[root@MariaDB-1 ~]#
```

- Khởi động node vừa join

```
pcs cluster start node4
pcs cluster enable node4
```

```
[root@MariaDB-1 ~]# pcs cluster start node4
node4: Starting Cluster (corosync)...
node4: Starting Cluster (pacemaker)...
[root@MariaDB-1 ~]# pcs cluster enable node4
node4: Cluster Enabled
```

- Kiểm tra

```
[root@MariaDB-1 ~]# pcs status
Cluster name: ha_cluster
Stack: corosync
Current DC: MariaDB-3 (version 1.1.23-1.el7_9.1-9acf116022) - partition with quorum
Last updated: Fri Oct 15 00:51:22 2021
Last change: Fri Oct 15 00:51:16 2021 by hacluster via crmd on MariaDB-3

4 nodes configured
2 resource instances configured

Online: [ MariaDB-1 MariaDB-2 MariaDB-3 node4 ]

Full list of resources:

 Virtual_IP     (ocf::heartbeat:IPaddr2):       Started MariaDB-3
 Loadbalancer_HaProxy   (systemd:haproxy):      Stopping[ MariaDB-3 node4 ]

Daemon Status:
  corosync: active/enabled
  pacemaker: active/enabled
  pcsd: active/enabled
[root@MariaDB-1 ~]#
```
<h3 align="center"><img src="../Images/Lab/67.png"></h3>

## 3. Loại bỏ node khỏi cluster
Tại node thuộc cluster (không phải node muốn loại bỏ):
```sh
pcs cluster node remove node4
```
```sh
[root@MariaDB-1 ~]# pcs cluster node remove node4
node4: Stopping Cluster (pacemaker)...
node4: Successfully destroyed cluster
MariaDB-1: Corosync updated
MariaDB-2: Corosync updated
MariaDB-3: Corosync updated
[root@MariaDB-1 ~]# pcs status
Cluster name: ha_cluster
Stack: corosync
Current DC: MariaDB-3 (version 1.1.23-1.el7_9.1-9acf116022) - partition with quorum
Last updated: Fri Oct 15 01:09:32 2021
Last change: Fri Oct 15 01:09:20 2021 by root via crm_node on MariaDB-1

3 nodes configured
2 resource instances configured

Online: [ MariaDB-1 MariaDB-2 MariaDB-3 ]

Full list of resources:

 Virtual_IP     (ocf::heartbeat:IPaddr2):       Started MariaDB-3
 Loadbalancer_HaProxy   (systemd:haproxy):      Started MariaDB-3

Daemon Status:
  corosync: active/enabled
  pacemaker: active/enabled
  pcsd: active/enabled
[root@MariaDB-1 ~]#
```

## 4. Di chuyển resource khỏi node
- Lựa chọn resource cần di chuyển, ở đây sẽ di chuyển `resource Loadbalancer_HaProxy` từ node `MariaDB-3` sang `MariaDB-1`
```sh
[root@MariaDB-1 ~]# pcs status
Cluster name: ha_cluster
Stack: corosync
Current DC: MariaDB-3 (version 1.1.23-1.el7_9.1-9acf116022) - partition with quorum
Last updated: Fri Oct 15 01:10:48 2021
Last change: Fri Oct 15 01:09:20 2021 by root via crm_node on MariaDB-1

3 nodes configured
2 resource instances configured

Online: [ MariaDB-1 MariaDB-2 MariaDB-3 ]

Full list of resources:

 Virtual_IP     (ocf::heartbeat:IPaddr2):       Started MariaDB-3
 Loadbalancer_HaProxy   (systemd:haproxy):      Started MariaDB-3

Daemon Status:
  corosync: active/enabled
  pacemaker: active/enabled
  pcsd: active/enabled
```
- Tạo ràng buộc di chuyển, câu lệnh không thực sự di chuyển resource mà tạo ràng buộc cho Resource phải tới node đích khi khởi động lại hoặc khi xảy ra lỗi.
```sh
pcs resource move Loadbalancer_HaProxy MariaDB-1
```
- Kiểm tra, ta thấy xuất hiện rằng buộc Resource: Loadbalancer_HaProxy Enable on MariaDB-1
```sh
[root@MariaDB-2 ~]# pcs constraint
Location Constraints:
  Resource: Loadbalancer_HaProxy
    Enabled on: MariaDB-1 (score:INFINITY) (role: Started)
Ordering Constraints:
  start Virtual_IP then start Loadbalancer_HaProxy (kind:Optional)
Colocation Constraints:
  Virtual_IP with Loadbalancer_HaProxy (score:INFINITY)
Ticket Constraints:
[root@MariaDB-2 ~]#
```
- Khởi động lại resource
```sh
pcs resource restart Loadbalancer_HaProxy
```
- Trạng thái 
```sh
pcs resource clear Loadbalancer_HaProxy
```
## 5. Ngừng dịch vụ tại 1 node chỉ định
- Trạng thái cluster
```sh
[root@MariaDB-1 ~]# pcs status
Cluster name: ha_cluster
Stack: corosync
Current DC: MariaDB-3 (version 1.1.23-1.el7_9.1-9acf116022) - partition with quorum
Last updated: Fri Oct 15 01:21:51 2021
Last change: Fri Oct 15 01:21:41 2021 by hacluster via crmd on MariaDB-3

4 nodes configured
2 resource instances configured

Online: [ MariaDB-1 MariaDB-2 MariaDB-3 node4 ]

Full list of resources:

 Virtual_IP     (ocf::heartbeat:IPaddr2):       Started MariaDB-1
 Loadbalancer_HaProxy   (systemd:haproxy):      Started MariaDB-1

Daemon Status:
  corosync: active/enabled
  pacemaker: active/enabled
  pcsd: active/enabled
[root@MariaDB-1 ~]#
```
- Chỉ định ngừng dịch vụ Loadbalancer_HaProxy tại node1 để thực hiện hoạt động bảo trì hoặc nâng cấp.
```sh
MariaDB-pcs resource ban Loadbalancer_HaProxy MariaDB-1
```
Cho phép dịch vụ hoạt động trở lại

```
pcs resource clear Loadbalancer_HaProxy
```

- Khởi động lại resource

```
pcs resource restart <Resource name>
```

- Xóa resource

```
pcs resource delete <Resource name>
```

### 2.6. Lỗi Resource (Blocked)

Thực hiện cleanup resource, trạng thái resource sẽ trở lại bình thường.

```
pcs resource cleanup <Resource name>
```


### 2.7. Di chuyển IP VIP

- IP VIP cấu hình dạng clone nên không thể di chuyển theo cách thông thường.
Di chuyển dịch vụ có ràng buộc với IP VIP (kiểm tra băng câu lệnh `pcs constraint` ), để pacemaker di chuyển IP VIP theo tài nguyên ràng buộc. Trong ví dụ này sẽ di chuyển dịch vụ `Loadbalancer_HaProxy` (ràng buộc IP VIP (Virtual_IP) luôn hoat động trên cùng node với http- Loadbalancer_HaProxy).
