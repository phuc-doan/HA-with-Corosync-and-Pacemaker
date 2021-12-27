# HƯỚNG DẪN CÀI ĐẶT HA CHO WEB-SERVER BẰNG PACEMAKER + COROSYNC, DRDB TRÊN CENTOS 7
 **APACHE , CNTT**
## 1. Chuẩn bị 🤝🤝
- 2 server sử dụng OS CentOS
- 2 ổ cứng có cùng dung lượng được gắn vào các node
- Cấu hình hostname cho các server
- Mở port 7788 trên các server
### Cụ thể:
- Node 1
```
OS: CentOS 7 64 bit
Device: /dev/sdb - 8GB
Hostname: node1
IP: 192.168.100.196
Gateway: 192.168.100.1
Network: 192.168.100.0/24
```
- Node 2
```
OS: CentOS 7 64 bit
Device: /dev/sdb - 8GB
Hostname: node2
IP: 192.168.100.197
Gateway: 192.168.100.1
Network: 192.168.100.0/24
```
### Mô hình 🖥🖥

![image](https://user-images.githubusercontent.com/83824403/147487809-f0063692-e644-407b-a14f-26d581794001.png)

#### Trước khi cài đặt, chúng ta phải cấu hình hostname cho mỗi node và ghi chúng vào hosts
```
[root@node1 ~] hostnamectl set-hostname node1
[root@node2 ~] hostnamectl set-hostname node2
```

- Ghi thêm vào hosts của mỗi server
```
vi /etc/hosts
[...]
192.168.100.196 node1
192.168.100.197 node2
```


### 2. Các bước thực hiện ✈✈
#### 2.1 Cài đặt Apache Web-server


- Cài đặt Apache trên cả 2 node:
```
yum -y install httpd*
```

- Cấu hình tường lửa cho phép người dùng bên ngoài có thể truy cập vào qua giao thức http:
```
firewall-cmd --permanent --add-service=http
firewall-cmd --reload
```
#### 2.2 Cài đặt🔑  Pacemaker  và Corosync ✈✈ 


- Cài đặt các gói cluster trên cả 2 node bằng yum:

```
yum install -y pacemaker pcs fence-agents-all psmisc policycoreutils-python
```
- Cũng như phần trên, chúng ta cấu hình tường lửa cho phép dịch vụ HA:
```
firewall-cmd --permanent --add-service=high-availability
firewall-cmd --reload
```
#### 2.3 Cấu hình Cluster với Pacemaker và Corosync 
- Trên cả 2 node, chúng ta đặt password cho user hacluster để xác thực với nhau, 2 mật khẩu phải trùng khớp
```
echo "redhat" | passwd --stdin hacluster
```
- Tiếp theo, chúng ta khởi động dịch vụ trên cả 2 node:
```
systemctl start pcsd
systemctl enable pcsd
```
- Trên node1, chúng ta xác thực 2 node với nhau bằng lệnh:
```
[root@node1 ~]# pcs cluster auth node1 node2 -u hacluster -p redhat
```


```
node1: Authorized
node2: Authorized
```
- Sau khi xác thực, chúng ta tạo 1 cluster trên node 1 có tên là mycluster để chúng có thể tạo và đồng bộ các file cấu hình với nhau. 🖥

```
[root@node1 ~]# pcs cluster setup --name mycluster node1 node2
```

```
Shutting down pacemaker/corosync services...
Redirecting to /bin/systemctl stop  pacemaker.service
Redirecting to /bin/systemctl stop  corosync.service
Killing any remaining services...
Removing all cluster configuration files...
node1: Succeeded
node2: Succeeded
Synchronizing pcsd certificates on nodes node1, node2...
node1: Success
node2: Success
Restaring pcsd on the nodes in order to reload the certificates...
node1: Success
node2: Success
```
- Khởi động và kích hoạt cluster mới tạo trên node 1 bằng lệnh:
```
[root@node1 ~]# pcs cluster start --all
[root@node1 ~]# pcs cluster enable --all
```
- Xem lại trạng thái của cluster trên các node:
```
pcs status
```
##### Tắt Quorum và STONITH
```
pcs property set stonith-enabled=false
pcs property set no-quorum-policy=ignore
pcs property set default-resource-stickiness="INFINITY"
```
#### 2.4 Cài đặt DRDB 🔐🔐
- Để cài đặt DRDB mới nhất, chúng ta thêm repo của DRDB (các thao tác này làm trên cả 2 node):
```
rpm -ivh http://www.elrepo.org/elrepo-release-7.0-2.el7.elrepo.noarch.rpm
```
- Thêm Public key cho DRDB
```
rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-elrepo.org
```
- Tiếp theo, chúng ta cài đặt DRDB trên cả 2 node:
```
yum -y install drbd*-utils kmod-drbd*
```
- Nếu quá trình cài đặt trên không thành công hoặc xảy ra lỗi hãy chèn lại key và tiếp tục cài đặt:
```
rpm --import /etc/pki/rpm-gpg/*
```
- Kích hoạt DRDB trên cả 2 node
```
modprobe drbd
```
- Kiểm tra DRDB đã hoạt động hay chưa:
```
lsmod | grep drbd
```

```
drbd                  405309  0
libcrc32c              12644  2 xfs,drbd
Nếu tường lửa được kích hoạt hãy thêm rule cho phép DRDB:
firewall-cmd --permanent --add-port=7788/tcp
firewall-cmd --reload
2.5 Cấu hình DRDB
Để cấu hình DRDB, chúng ta tạo một file có tên là testdata1.res với nội dung:
vi /etc/drbd.d/testdata1.res
resource testdata1 {
protocol C;        
on node1 {
                device /dev/drbd0;
                disk /dev/vgdrbd/vol1;
                address 192.168.100.196:7788;
                meta-disk internal;
        }
on node2 {
                device /dev/drbd0;
                disk /dev/vgdrbd/vol1;
                address 192.168.100.197:7788;
                meta-disk internal;
        }
} 
```

#### Giải thích:
```
resource testdata1: Tên của resource
Protocol C: Các resource được cấu hình để synchronous replication. Chi tiết tại đây
node1, node2: Danh sách các node và các tùy chọn bên trong
device /dev/drbd0: Xác định thiết bị logic được DRBD sử dụng (Nên đặt giống nhau ở trên 2 server)
disk /dev/vgdrbd/vol1: Xác định thiết bị vật lý dùng để tạo ra thiết bị logic bên trên, và không nhất thiết phải cùng trên trên 2 server.
address 192.168.100.197:7788: Xác định địa chỉ IP và Port của mỗi server
meta-disk internal: Cho phép sử dụng Meta-data mức nội bộ
``` 
- Sao chép file cấu hình sang node 2:
```
[root@node1 ~] scp /etc/drbd.d/testdata1.res node2:/etc/drbd.d/
```
- Trên cả 2 node, chúng ta tạo một LVM để lưu trữ các dữ liệu mà chúng ta đã định nghĩa ở file cấu hình bên trên /dev/vgdrbd/vol1. Ở đây, /dev/sdb được sử dụng để tạo LVM
pvcreate /dev/sdb
 ```
 Physical volume "/dev/sdb" successfully created
vgcreate vgdrbd /dev/sdb
  Volume group "vgdrbd" successfully created
lvcreate -n vol1 -l100%FREE vgdrbd
  Logical volume "vol1" created.
  ```
### 2.6 Khởi động DRDB meta-data storage 👌
- Sau khi cấu hình xong các bước bên trên, chúng ta chuyển tiếp đến phần khởi động thiết bị DRBD. Trên cả 2 node chúng ta thực hiện lần lượt các lệnh sau và đọc thông báo kết quả:
```
[root@node1 ~] drbdadm create-md testdata1
[root@node2 ~] drbdadm create-md testdata1
```
```
Above commands will give an output like below.
  --==  Thank you for participating in the global usage survey  ==--
The server's response is:
you are the 10680th user to install this version
initializing activity log
NOT initializing bitmap
Writing meta data...
New drbd meta data block successfully created.
success
```
- Tiếp đến, chúng ta khởi động DRBD và kích hoạt khởi chạy cùng hệ thống:
```
systemctl start drbd
systemctl enable drbd
```
### 2.7 Xác định DRDB Primary node 👍
- Xác định node DRBD Primary trên node1:
```
[root@node1 ~] drbdadm primary testdata1 --force
```
- Kiểm tra kết quả quá trình đồng bộ:

```
[root@node1 ~]# cat /proc/drbd
version: 8.4.7-1 (api:1/proto:86-101)
GIT-hash: 3a6a769340ef93b1ba2792c6461250790795db49 build by phil@Build64R7, 2016-01-12 14:29:40
 0: cs:Connected ro:Primary/Secondary ds:UpToDate/UpToDate C r-----
    ns:1048508 nr:0 dw:0 dr:1049236 al:8 bm:0 lo:0 pe:0 ua:0 ap:0 ep:1 wo:f oos:0
[root@node1 ~]# drbd-overview
 0:testdata1/0  Connected Primary/Secondary UpToDate/UpToDate
 ```
### 2.8 Tạo và mount filesystem thiết bị DRDB
- Làm bước này chỉ khi bước trên thành công, khi kiểm tra thấy DRBD báo Connected
```
[root@node1 ~]# drbd-overview
 0:testdata1/0  Connected Primary/Secondary UpToDate/UpToDate
 ```
- Chúng ta tạo filesystem và ghi dữ liệu vào nó:
```
[root@node1 ~]# mkfs.ext3 /dev/drbd0
[root@node1 ~]# mount /dev/drbd0 /mnt
[root@node1 ~]# echo "Welcome to my Website" > /mnt/index.html
[root@node1 ~]# umount /mnt
```
### 2.9 Tạo cluster resource 🚗 và đặt các thuộc tính 🔑
- Tiếp đến, chúng ta tạo ra các tài nguyên như VIP, Httpd, DrbdData, DrbdDataClone, DrbdFS trên node1 để khi có yêu cầu sẽ bring up nó lên trên các node khác trong trường hợp có sự cố xảy ra:
- Tạo cho VIP
```
[root@node1 ~]# pcs resource create VirtIP ocf:heartbeat:IPaddr2 ip=192.168.100.123 cidr_netmask=32 op monitor interval=30s
```
- Tạo cho dịch vụ Httpd
```
[root@node1 ~]#  pcs resource create Httpd ocf:heartbeat:apache configfile=/etc/httpd/conf/httpd.conf op monitor interval=1min

```
- Tạo sự ràng buộc giữa Httpd và VIP
- Chúng ta cấu hình 2 tài nguyên này ràng buộc với nhau. Giải thích câu lệnh: Khi VIP khởi động xong thì Httpd mới khởi động, và 2 tài nguyên này là một cặp.
```
[root@node1 ~]# pcs constraint colocation add Httpd with VirtIP INFINITY
[root@node1 ~]# pcs constraint order VirtIP then Httpd

Adding VirtIP Httpd (kind: Mandatory) (Options: first-action=start then-action=start)
``` 
- Tạo sự thống nhất giữa tài nguyên DRBD trên các node
- Khi một node1 đang Active, thì node2 ở chế độ Passive lưu trữ và đồng bộ dữ liệu từ node1. Để làm như vậy, chúng ta gộp tài nguyên ở 2 node bằng cấu hình CIB
```
[root@node1 ~]# pcs cluster cib drbd_cfg
```
- Khai báo tài nguyên DRBD
```
[root@node1 ~]#  pcs -f drbd_cfg resource create DrbdData ocf:linbit:drbd drbd_resource=testdata1 op monitor interval=60s
```
**Chú ý: testdata1 là tên của tài nguyên được khai báo trong file cấu hình DRBD.** 🖥🖥
- Tạo bản sao tài nguyên DRBD
- Bản sao này cho phép tài nguyên có thể chạy trên các node cùng một thời điểm
```
[root@node1 ~]#  pcs -f drbd_cfg resource master DrbdDataClone DrbdData master-max=1 master-node-max=1 clone-max=2 clone-node-max=1 notify=true
```
- Chúng ta commit những gì vừa cấu hình bên trên vào CIB chính:
```
[root@node1 ~]# pcs cluster cib-push drbd_cfg



CIB updated
```
- Kiểm tra trạng thái của các resource
```
[root@node1 ~]# pcs status resources
 VirtIP (ocf::heartbeat:IPaddr2):       Started node1
 Httpd  (ocf::heartbeat:apache):        Started node1
 Master/Slave Set: DrbdDataClone [DrbdData]
     Masters: [ node1 ]
     Slaves: [ node2 ]
   ```
- ✈ Tạo tài nguyên filesystem DRBD
```
[root@node1 ~]# pcs cluster cib fs_cfg
```
- Chúng ta tạo 1 tài nguyên filesystem để mount thiết bị DRBD
```
[root@node1 ~]#  pcs  -f fs_cfg resource create DrbdFS Filesystem device="/dev/drbd0" directory="/var/www/html" fstype="ext3"
```
- Như ở trên, chúng ta cũng sẽ cấu hình ràng buộc khi stop, start các tài nguyên
```
[root@node1 ~]# pcs  -f fs_cfg constraint colocation add DrbdFS with DrbdDataClone INFINITY with-rsc-role=Master
[root@node1 ~]# pcs  -f fs_cfg constraint order promote DrbdDataClone then start DrbdFS
Adding DrbdDataClone DrbdFS (kind: Mandatory) (Options: first-action=promote then-action=start)
[root@node1 ~]# pcs -f fs_cfg constraint colocation add Httpd with DrbdFS INFINITY
[root@node1 ~]# pcs -f fs_cfg constraint order DrbdFS then Httpd


Adding DrbdFS Httpd (kind: Mandatory) (Options: first-action=start then-action=start)
```
- Cập nhật các thay đổi bằng lệnh:
```
[root@node1 ~]# pcs cluster cib-push fs_cfg
CIB Updated
```
- Kiểm tra lại trạng thái của các tài nguyên:
```
[root@node1 ~]# pcs status resources
 VirtIP (ocf::heartbeat:IPaddr2):       Started node1
 Httpd  (ocf::heartbeat:apache):        Started node1
 Master/Slave Set: DrbdDataClone [DrbdData]
     Masters: [ node1 ]
     Slaves: [ node2 ]
 DrbdFS (ocf::heartbeat:Filesystem):    Started node1
 ```
- Mặc định thì thời gian timeout cho start, stop và monitor các tài nguyên là 20s. Chúng ta chỉnh lại thời gian này bằng lệnh:
```
root@node1 ~]# pcs resource op defaults timeout=240s
[root@node1 ~]# pcs resource op defaults
timeout: 240s
```
#### Bây giờ, chúng ta có thể truy cập vào địa chỉ VIP: 192.168.100.123 để có thể kiểm tra sự hoạt động của cluster.
### 2.10 Test trường hợp fail-over bằng tay ✋✋
- Hiện tại, tất cả các tài nguyên được sử dụng trên node1. Hãy stop chúng lại để tạo ra trường hợp fail-over và sang node2 kiểm tra các tài nguyên:
```
[root@node1 ~]# pcs cluster stop node1
node1: Stopping Cluster (pacemaker)...
node1: Stopping Cluster (corosync)...
```
**Khi node1 đã stop ❌ , chúng ta sang node2 để kiểm tra:**
```
[root@node2 ~]# pcs status resources
 VirtIP (ocf::heartbeat:IPaddr2):       Started node2
 Httpd  (ocf::heartbeat:apache):        Started node2
 Master/Slave Set: DrbdDataClone [DrbdData]
     Masters: [ node2 ]
     Stopped: [ node1 ]
 DrbdFS (ocf::heartbeat:Filesystem):    Started node2
 ```
### Như vậy là chúng ta đã hoàn thành quá trình cài đặt cluster bằng Pacemaker và Corosync.✔✔❤

