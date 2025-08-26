# Install radosgw using Cephadm

Mô hình triển khai có 3 con server, nằm trên 3 dải mạng, một dải để ssh và ra internet, một dải ceph external và một dải internal cho ceph.

![image.png](Install%20radosgw%20using%20Cephadm%2019b191b7124780c68cf7d85767294d8e/image.png)

Kiểm tra network được cấu hình đúng cho 3 con trên 3 dài mạng

Update package manager cho **con thứ nhất**

```bash
apt update
```

Tải cephadm. Cần cài phiên bản nào thì sửa biến môi trường CEPH_RELEASE cho phù hợp

```bash
CEPH_RELEASE=18.2.0
curl --silent --remote-name --location https://download.ceph.com/rpm-${CEPH_RELEASE}/el9/noarch/cephadm
chmod +x cephadm
```

Tiếp theo chọn phiên bản Ceph muốn cài đặt. Ví dụ quincy

```bash
./cephadm add-repo --release quincy
```

Sau đó install cephadm 

```bash
./cephadm install
```

Chạy đúng khi câu lệnh cephadm có thể gõ trực tiếp

![image.png](Install%20radosgw%20using%20Cephadm%2019b191b7124780c68cf7d85767294d8e/image%201.png)

### Bootstrap a new cluster

Thao tác vẫn trên node 1

```bash
sudo cephadm bootstrap --mon-ip <external> --cluster-network <CLUSTER_NET>

```

Lệnh này sẽ bootstrap cluster, đặt và tạo các file config, khởi tạo mon node đầu tiên, chỉ định cluster_network

Ví dụ:

```bash
sudo cephadm bootstrap --mon-ip 10.125.100.75 --cluster-network 10.125.100.0/24
```

### Install ceph-common to use *ceph* and other essential commands

```bash
cephadm install ceph-common
```

### Add the other two hosts into the cluster

Khi bootstrap container, cephadm sẽ tạo ra user ceph và public ssh key sẽ được lưu vào `/etc/ceph/ceph.pub`

Copy key này sang 2 user root của 2 node còn lại

```bash
ssh-copy-id -f -i /etc/ceph/ceph.pub root@host2
ssh-copy-id -f -i /etc/ceph/ceph.pub root@host3
```

Sau đó install podman hoặc docker trên 2 node còn lại

```bash
apt install -y podman
```

Sau đó thêm add host vào cluster

```bash
ceph orch host add <host_name> <host_public_network_IP>
```

lưu ý đoạn này lấy internal IP

Verify xem các host đã được add thành công chưa bằng 

```bash
ceph orch host ls
```

![image.png](Install%20radosgw%20using%20Cephadm%2019b191b7124780c68cf7d85767294d8e/image%202.png)

Sau khi deploy xong, cephadm sẽ deploy 3 monitor trên 3 node.  Kiểm tra bằng `ceph -s` 

![image.png](Install%20radosgw%20using%20Cephadm%2019b191b7124780c68cf7d85767294d8e/image%203.png)

Configure public network cho cluster

```bash
ceph config set mon public_network IP_ADDRESS_WITH_SUBNET
```

### Add Storage devices and OSD

Mỗi block device sẽ được quản lý bởi một OSD trong ceph, dùng câu lệnh sau để apply tất cả block device với OSD

```bash
ceph orch apply osd --all-available-devices
```

Hoặc có thể thêm từng OSD một bằng cách

```bash
 ceph orch daemon add osd <HOSTNAME>:<device>
```

Ví dụ

![image.png](Install%20radosgw%20using%20Cephadm%2019b191b7124780c68cf7d85767294d8e/image%204.png)

Kiểm tra bằng `ceph osd tree` hoặc `ceph -s` 

![image.png](Install%20radosgw%20using%20Cephadm%2019b191b7124780c68cf7d85767294d8e/image%205.png)

### Deploy RadosGW

Để deploy rgw dùng câu lệnh

```bash
ceph orch apply rgw *<name>* [--realm=*<realm-name>*] [--zone=*<zone-name>*] --placement="*<num-daemons>* [*<host1>* ...]"
```

Trước hết chưa vội cấu hình realm và zone, để mặc định

![image.png](Install%20radosgw%20using%20Cephadm%2019b191b7124780c68cf7d85767294d8e/image%206.png)

![image.png](Install%20radosgw%20using%20Cephadm%2019b191b7124780c68cf7d85767294d8e/image%207.png)