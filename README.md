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
Cài đặt các gói cần thiết 
  
  apt-get install swift swift-account swift-container swift-object xfsprogs -y

Tạo phân vùng đĩa mới để lưu trữ cho swift

  fdisk /dev/sdb
  mkfs.xfs /dev/sdb1
  echo "/dev/sdb1 /srv/node/sdb1 xfs noatime,nodiratime,nobarrier,logbufs= 8 0 0" >> /etc/fstab
  mkdir -p /srv/node/sdb1
  mount /dev/sdb1 /srv/node/sdb1
  chown -R swift:swift /srv/node



  
  
  
