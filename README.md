# Hướng dẫn cài đặt, cấu hình Swift      (Object Storage)
## I. Thông tin lap
Bài lap sẽ gồm 3 phần:
  - Phần 1: Cài đặt Swift với mô hình 2 node (Proxy node và Storage node)
  - Phần 2: Mở rộng thêm một Storage node từ mô hình trên (Proxy node và 2 Storage node)
  - Phần 3: Mở rộng dung lượng của mỗi Storage node

Cụ thể sẽ trình bày trong từng phần
## II. Các bước cài đặt
### 1. Cài đặt Swift với mô hình 2 node
Mô hình...
IP...
Chuẩn bị:
  - 2 máy ảo chạy ubuntu server tương ứng với 2 node.
  - Với Proxy node, cần cài sẵn OpenStack All In One theo hướng dẫn [*tại đây*](https://github.com/vietstacker/icehouse-aio-ubuntu/blob/master/hd-caidat-openstack-icehouse-aio.md)
  - Với Storage node, cài đặt ubuntu server 12.04 hoặc 14.04 với cấu hình tối thiểu như sau:

##### Thực hiện các bước sau trên cả hai node:
Tạo thư mục chứa swift
  
    mkdir -p /etc/swift

Tạo file cấu hình /etc/swift/swift.conf và chèn nội dung sau:
  
    [swift-hash]
    # random unique string that can never change (DO NOT LOSE)
    swift_hash_path_prefix = xrfuniounenqjnw
    swift_hash_path_suffix = fLIbertYgibbitZ

##### Trên Storage node:
Cài đặt các gói cần thiết:
  
    apt-get install swift swift-account swift-container swift-object xfsprogs -y

Tạo phân vùng đĩa mới để lưu trữ cho swift

    fdisk /dev/sdb
    mkfs.xfs /dev/sdb1
    echo "/dev/sdb1 /srv/node/sdb1 xfs noatime,nodiratime,nobarrier,logbufs= 8 0 0" >> /etc/fstab
    mkdir -p /srv/node/sdb1
    mount /dev/sdb1 /srv/node/sdb1
    chown -R swift:swift /srv/node

  **Note:** Trong trường hợp làm như trên mà khi restart máy không tự động mount được ổ cứng thì mở file /etc/fstab, thay dòng */dev/sdb1 /srv/node/sdb1 xfs noatime,nodiratime,nobarrier,logbufs= 8 0 0* bằng */dev/sdb1 /srv/node/sdb1 xfs _netdev 0 0* rồi reboot. <br>

Tiếp theo, tạo file cấu hình cho dịch vụ đồng bộ:
  
    vim /etc/rsyncd.conf

Chèn nội dung sau, cần chú ý đặt IP cho đúng:

    uid = swift
    gid = swift
    log file = /var/log/rsyncd.log
    pid file = /var/run/rsyncd.pid
    address = 192.168.10.128
    
    [account]
    max connections = 2
    path = /srv/node/
    read only = false
    lock file = /var/lock/account.lock
    
    [container]
    max connections = 2
    path = /srv/node/
    read only = false
    lock file = /var/lock/container.lock
    
    [object]
    max connections = 2
    path = /srv/node/
    read only = false
    lock file = /var/lock/object.lock

Ở chế độ mặc định, dịch vụ đồng bộ bị ẩn đi, ta phải sửa file /etc/default/rsync để bật nó lên. Mở file /etc/default/rsync, tìm đến và thay dòng

    RSYNC_ENABLE=false

bằng

    RSYNC_ENABLE=true

Sau đó bật dịch vụ đồng bộ lên:

    service rsync start

  *Note:* Dịch vụ rsync yêu cầu không có sự xác thực, do đó chạy nó trên máy local, mạng private. <br>

Tạo thư mục swift recon cache và gán quyền:

    mkdir -p /var/swift/recon
    chown -R swift:swift /var/swift/recon

##### Trên Proxy node:
Cài đặt dịch vụ swift-proxy:

    apt-get install swift swift-proxy memcached python-keystoneclient python-swiftclient python-webob -y

Mặc định memcached sẽ lắng nghe trên cổng local, sửa file /etc/memcached.conf:

    -l 127.0.0.1

sửa thành:

    -l 192.168.10.128

Restart dịch vụ:

    service memcached restart

Tạo file /etc/swift/proxy-server.conf

    [DEFAULT]
    bind_port = 8080
    user = swift
    
    [pipeline:main]
    pipeline = healthcheck cache authtoken keystoneauth proxy-server
    
    [app:proxy-server]
    use = egg:swift#proxy
    allow_account_management = true
    account_autocreate = true
    
    [filter:keystoneauth]
    use = egg:swift#keystoneauth
    operator_roles = Member,admin,swiftoperator
    
    [filter:authtoken]
    paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
    
    # Delaying the auth decision is required to support token-less
    # usage for anonymous referrers ('.r:*').
    delay_auth_decision = true
    
    # auth_* settings refer to the Keystone server
    auth_protocol = http
    auth_host = controller
    auth_port = 35357
    
    # the service tenant and swift username and password created in Keystone
    admin_tenant_name = service
    admin_user = swift
    admin_password = Welcome123
    
    [filter:cache]
    use = egg:swift#memcache
    
    [filter:catch_errors]
    use = egg:swift#catch_errors
    
    [filter:healthcheck]
    use = egg:swift#healthcheck

**Note:** Do đã cài đặt OpenStack AIO nên admin_password = Welcome123 <br>

Chuyển đến thư mục swift để tạo ring:

    cd /etc/swift
    swift-ring-builder account.builder create 18 3 1
    swift-ring-builder container.builder create 18 3 1
    swift-ring-builder object.builder create 18 3 1
  
Sau đó add ổ sdb1 vào zone 1:

    swift-ring-builder account.builder add z1-192.168.10.129:6002/sdb1 100
    swift-ring-builder container.builder add z1-192.168.10.129:6001/sdb1 100
    swift-ring-builder object.builder add z1-192.168.10.129:6000/sdb1 100

Tạo các file ring:

    swift-ring-builder account.builder rebalance
    swift-ring-builder container.builder rebalance
    swift-ring-builder object.builder rebalance

Đẩy các ring này đến Storage node:

    scp container.ring.gz object.ring.gz account.ring.gz root@192.168.10.129:/etc/swift

Để chắc chắn, chuyển quyền sở hữu các file config cho user swift:

    chown -R swift:swift /etc/swift

Restart các dịch vụ:
  
    swift-init proxy start
    swift-init main start
    service rsyslog restart
    service memcached restart

##### Trở lại Storage node:
Bật toàn bộ dịch vụ:

    for service in swift-object swift-object-replicator swift-object-updater swift-object-auditor swift-container swift-container-replicator swift-container-updater swift-container-auditor swift-account swift-account-replicator swift-account-reaper swift-account-auditor; do service $service start; done

hoặc đơn giản hơn với câu lệnh:

    swift-init all start

##### Kiểm tra dịch vụ:
Vào trình duyệt web trên máy thật gõ địa chỉ của Proxy node, ở đây là 10.145.48.128. Sau khi đăng nhập, chọn Project > Object Store > Containers thực hiện upload  object. Nếu không báo lỗi gì có nghĩa là đã cài đặt thành công!


  
