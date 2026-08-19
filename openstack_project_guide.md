# Hướng Dẫn Từng Bước: Xây Dựng Private Cloud OpenStack (Multi-Node DevStack)

Tài liệu này hướng dẫn chi tiết cách cấu hình hệ thống OpenStack đa nút (Multi-Node DevStack) và hoàn thành các Mô-đun trong đồ án **Project 2: Building a Private Cloud Based on OpenStack**.

---

## 1. Kiến Trúc Hệ Thống (Multi-Node Cluster qua Tailscale VPN)

Các thiết bị của chúng ta không nằm trong cùng một mạng LAN vật lý mà được kết nối qua mạng **Tailscale VPN (Virtual Private Network)**. Điều này cho phép các node giao tiếp trực tiếp một cách an toàn thông qua lớp mạng ảo mã hóa (mặc định dải IP của Tailscale nằm trong khoảng `100.64.0.0/10`):

*   **Control Node (Mini PC - Tailscale IP `100.99.10.20`)**: Chạy các dịch vụ quản lý như Keystone, Glance, Horizon, Neutron Server, Heat, Cinder API/Scheduler, Swift, MySQL và RabbitMQ. Tắt dịch vụ hypervisor/chạy VM (`n-cpu`) để tiết kiệm tài nguyên.
*   **Compute Node (Ubuntu 26 - Tailscale IP `100.99.10.30`)**: Chạy Nova Compute (`n-cpu`) và Neutron L2 Agent (`q-agt`). Đây là nơi các máy ảo (Instance) chạy thực tế.
*   **Client (macOS - Tailscale IP `100.99.10.40`)**: Dùng để SSH điều khiển, chạy `openstack` CLI và mở Dashboard Horizon trên trình duyệt.


> [!IMPORTANT]
> **Lưu ý về MTU khi dùng Tailscale**: Tailscale VPN có MTU mặc định là **1280**. Khi bọc các gói tin mạng ảo OpenStack (VXLAN overlay) bên trong đường truyền Tailscale, chúng ta cần trừ đi 50 bytes header của VXLAN. Vì vậy, MTU của mạng OpenStack được cấu hình tối đa là **1200** (`global_physnet_mtu=1200` trong `local.conf`). Điều này giúp tránh hiện tượng phân mảnh gói tin gây lỗi mạng (chậm mạng, drop connection khi tải file từ Swift).

> [!IMPORTANT]
> Hãy thay thế các địa chỉ IP `100.99.10.20` (Mini PC) và `100.99.10.30` (Ubuntu 26) trong tài liệu này bằng địa chỉ IP Tailscale thực tế của bạn.

---

## 2. Các Bước Cài Đặt Hệ Thống

### Bước 2.1: Chuẩn Bị Môi Trường Trên 2 Node (Mini PC & Ubuntu 26)
Thực hiện các lệnh sau trên **cả hai máy** Linux để cập nhật hệ thống và cài đặt các gói cần thiết:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y git python3-pip openvswitch-switch
```

Nếu hệ thống đang bật UFW (Tường lửa), bạn nên tắt đi hoặc cấu hình cho phép giao tiếp giữa hai node để tránh lỗi kết nối cơ sở dữ liệu và RabbitMQ:
```bash
sudo ufw disable
```

### Bước 2.2: Cài Đặt Control Node (Mini PC)
1. SSH vào Mini PC từ macOS thông qua Tailscale IP:
   ```bash
   ssh user@100.99.10.20
   ```
2. Clone repository DevStack phiên bản `stable/2024.2`:
   ```bash
   git clone https://opendev.org/openstack/devstack -b stable/2024.2
   cd devstack
   ```
3. Tạo file `local.conf` bằng cách sao chép nội dung từ file [`local_controller.conf`](file:///Users/caolegiaphu/Documents/cloud_aws/final_project/local_controller.conf) mà chúng ta đã chuẩn bị:
   ```bash
   # Chép file local_controller.conf sang Mini PC đặt tên là local.conf
   ```
   > [!TIP]
   > Hãy kiểm tra kỹ tham số `HOST_IP` trong file `local.conf` phải khớp với IP Tailscale của Mini PC (`100.99.10.20`).
4. Chạy script cài đặt (quá trình này mất khoảng 20 - 30 phút):
   ```bash
   ./stack.sh
   ```

### Bước 2.3: Cài Đặt Compute Node (Ubuntu 26)
1. SSH vào máy Ubuntu 26 từ macOS thông qua Tailscale IP:
   ```bash
   ssh user@100.99.10.30
   ```
2. Clone DevStack bản tương ứng:
   ```bash
   git clone https://opendev.org/openstack/devstack -b stable/2024.2
   cd devstack
   ```
3. Tạo file `local.conf` bằng cách sao chép nội dung từ file [`local_compute.conf`](file:///Users/caolegiaphu/Documents/cloud_aws/final_project/local_compute.conf):
   > [!TIP]
   > Kiểm tra kỹ: `HOST_IP` phải là IP Tailscale của máy Ubuntu 26 (`100.99.10.30`) và `SERVICE_HOST` phải là IP Tailscale của Control Node - Mini PC (`100.99.10.20`).
4. Chạy cài đặt sau khi Control Node đã chạy xong:
   ```bash
   ./stack.sh
   ```

---

## 3. Cấu Hình Trên Máy macOS Client

Để quản lý và tương tác với cụm OpenStack từ xa, bạn cấu hình macOS như sau:

### Bước 3.1: Cài Đặt OpenStack CLI trên macOS
Mở terminal trên macOS và cài đặt thư viện client thông qua Python virtual environment để tránh xung đột hệ thống:

```bash
# Tạo và kích hoạt môi trường ảo Python
python3 -m venv openstack_cli_env
source openstack_cli_env/bin/activate

# Cài đặt client
pip install --upgrade pip
pip install python-openstackclient python-heatclient python-swiftclient
```

### Bước 3.2: Tải File Xác Thực (OpenRC)
1. Mở trình duyệt trên macOS truy cập Horizon Dashboard bằng Tailscale IP của Mini PC: `http://100.99.10.20/dashboard`
2. Đăng nhập bằng tài khoản:
   *   Domain: `default`
   *   User: `admin`
   *   Password: `<Mật khẩu bạn đặt ở ADMIN_PASSWORD>`
3. Tại góc trên bên phải, nhấn vào thông tin User -> chọn **OpenStack RC File** -> tải file `admin-openrc.sh` về máy.
4. Di chuyển file đó vào thư mục dự án và nạp (source) thông tin xác thực trên terminal macOS:
   ```bash
   source admin-openrc.sh
   # Nhập mật khẩu admin khi được yêu cầu
   ```
5. Kiểm tra kết nối CLI bằng cách liệt kê các dịch vụ:
   ```bash
   openstack service list
   ```

### Bước 3.3: Định Tuyến Truy Cập Floating IP từ macOS Qua Tailscale
Mặc định, dải IP floating của các VM trong OpenStack là `172.24.4.0/24`. Dải này được định tuyến nội bộ bên trong Control Node qua bridge `br-ex`. Có 2 cách để máy macOS truy cập được dải này qua Tailscale:

#### Cách 1: Sử dụng Tailscale Subnet Router (Khuyên dùng - chuyên nghiệp nhất)
Tailscale cho phép biến Control Node (Mini PC) thành một router chuyển tiếp dải mạng ảo vào mạng VPN.
1. Trên Control Node (Mini PC), cấu hình quảng bá dải floating IP:
   ```bash
   sudo tailscale up --advertise-routes=172.24.4.0/24
   ```
2. Đăng nhập vào trang quản trị **Tailscale Admin Console** (https://login.tailscale.com) -> tìm thiết bị Mini PC của bạn -> Chọn mục **Edit Route Settings** -> Bật checkbox chấp nhận route `172.24.4.0/24`.
3. Sau khi bật, máy macOS (và bất kỳ máy nào khác trong mạng Tailnet của bạn) sẽ tự động ping/SSH được tới các IP `172.24.4.x` mà không cần cấu hình thủ công gì thêm!

#### Cách 2: Thêm Route Tĩnh Thủ Công Trên macOS
Nếu không muốn dùng Subnet Router, bạn có thể định tuyến thủ công trên macOS bằng cách trỏ dải floating IP qua IP Tailscale của Mini PC:
```bash
sudo route -n add 172.24.4.0/24 100.99.10.20
```
> [!NOTE]
> Để cách này hoạt động, hãy chắc chắn rằng tính năng IP Forwarding đã được bật trên Mini PC:
> ```bash
> sudo sysctl -w net.ipv4.ip_forward=1
> ```

---

## 4. Hướng Dẫn Thực Hiện Từng Mô-Đun Dự Án

### Module 1: Horizon & Core Services
**Mục tiêu**: Làm quen với giao diện Horizon và kiểm tra trạng thái hoạt động của Heat/Swift.

1. **Kiểm tra trạng thái dịch vụ Heat và Swift**:
   Thực hiện các lệnh này trên terminal macOS (sau khi đã `source admin-openrc.sh`):
   ```bash
   # Kiểm tra Heat
   openstack orchestration service list
   
   # Kiểm tra Swift (Không báo lỗi và trả về danh sách trống là thành công)
   openstack container list
   ```
2. **Khởi chạy và xóa một máy ảo mẫu**:
   *   **Giao diện Horizon**: Vào *Compute -> Instances -> Launch Instance*. Chọn image mặc định (ví dụ `cirros`), chọn flavor `m1.tiny`, chọn mạng mặc định `shared`. Nhấn Launch. Khi VM ở trạng thái `Active`, thử nhấn chuột vào VM và xem console log.
   *   **CLI trên macOS**:
       ```bash
       # Tạo instance mẫu
       openstack server create --image cirros-0.6.2-x86_64-disk --flavor m1.tiny --network private demo-cli-vm
       
       # Xem danh sách máy ảo
       openstack server list
       
       # Xóa instance mẫu
       openstack server delete demo-cli-vm
       ```

---

### Module 2: Xây Dựng Topology Mạng Đầu Tiên
**Mục tiêu**: Tạo hai mạng cô lập DMZ và Private, định tuyến ra ngoài thông qua Router chung.

Sử dụng macOS CLI để thực thi các lệnh tạo mạng một cách nhanh chóng và chính xác:

```bash
# 1. Tạo mạng dmz-net và subnet tương ứng
openstack network create dmz-net
openstack subnet create --network dmz-net --subnet-range 10.0.1.0/24 --dns-nameserver 8.8.8.8 dmz-subnet

# 2. Tạo mạng private-net và subnet tương ứng
openstack network create private-net
openstack subnet create --network private-net --subnet-range 10.0.2.0/24 --dns-nameserver 8.8.8.8 private-subnet

# 3. Tạo Router kết nối mạng ngoài (mặc định trong Devstack là public)
openstack router create project-router
openstack router set --external-gateway public project-router

# 4. Gắn các subnet vào Router để định tuyến liên mạng
openstack router add subnet project-router dmz-subnet
openstack router add subnet project-router private-subnet
```

> [!TIP]
> Hãy vào Horizon truy cập *Network -> Network Topology* để xem sơ đồ mạng trực quan. Chụp lại màn hình sơ đồ này cho báo cáo (`module2_diagram.png`).

---

### Module 3: Triển Khai Dịch Vụ Đầu Tiên (Hello World Server)
**Mục tiêu**: Tạo máy ảo có 2 card mạng (NICs), cấu hình định tuyến và chạy Web Server.

1. **Chuẩn bị Ubuntu Cloud Image (vì máy ảo Cirros quá tối giản, khó chạy Web và Swift client)**:
   Tải và import Ubuntu 22.04 LTS cloud image vào Glance:
   ```bash
   # Tải ảnh đĩa cloud
   wget https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img
   
   # Import vào OpenStack
   openstack image create --disk-format qcow2 --container-format bare --public --file jammy-server-cloudimg-amd64.img ubuntu-22.04
   ```

2. **Tạo Keypair để SSH từ macOS**:
   ```bash
   openstack keypair create --public-key ~/.ssh/id_rsa.pub mykey
   ```

3. **Cấu hình Security Group (Cho phép SSH, HTTP và Ping)**:
   ```bash
   # Tạo security group mới
   openstack security group create web-sec-group
   
   # Cho phép ping (ICMP)
   openstack security group rule create --proto icmp web-sec-group
   
   # Cho phép SSH (port 22)
   openstack security group rule create --proto tcp --dst-port 22 web-sec-group
   
   # Cho phép HTTP (port 80)
   openstack security group rule create --proto tcp --dst-port 80 web-sec-group
   ```

4. **Khởi chạy máy ảo dual-homed (2 NICs)**:
   Lưu ý thứ tự: card mạng thứ nhất kết nối vào `dmz-net`, card thứ hai kết nối vào `private-net`.
   ```bash
   openstack server create \
     --image ubuntu-22.04 \
     --flavor m1.small \
     --key-name mykey \
     --security-group web-sec-group \
     --nic net-id=dmz-net \
     --nic net-id=private-net \
     hello-web-vm
   ```

5. **Gán Floating IP cho card mạng DMZ**:
   ```bash
   # Tạo một Floating IP từ mạng ngoài
   FIP_IP=$(openstack floating ip create public -f value -c floating_ip_address)
   echo "Floating IP vừa tạo: $FIP_IP"
   
   # Gán Floating IP vào port kết nối dmz-net của máy ảo hello-web-vm
   # Lấy port ID của máy ảo nằm trên dmz-net
   PORT_ID=$(openstack port list --device-id $(openstack server show hello-web-vm -f value -c id) | grep "10.0.1." | awk '{print $1}')
   
   openstack floating ip set --port $PORT_ID $FIP_IP
   ```

6. **Fix định tuyến bất đối xứng (Asymmetric Routing) bên trong máy ảo**:
   Do máy ảo có 2 card mạng, hệ điều hành Ubuntu trong VM có thể bị bối rối và trả gói tin phản hồi ra sai cổng mạng.
   *   SSH vào VM qua Floating IP từ macOS:
       ```bash
       ssh ubuntu@$FIP_IP
       ```
   *   Xem bảng định tuyến hiện tại bằng lệnh: `ip route` hoặc `route -n`.
   *   Bạn sẽ thấy có hai gateway mặc định (qua `10.0.1.1` và `10.0.2.1`). Hãy xóa gateway của mạng Private để mọi phản hồi HTTP đi ra ngoài Internet đều qua cổng DMZ (`10.0.1.1`):
       ```bash
       sudo ip route del default via 10.0.2.1
       ```

7. **Cài đặt Web Server và kiểm tra**:
   Trong máy ảo Ubuntu, khởi chạy một server Python đơn giản:
   ```bash
   echo "<h1>Hello World from OpenStack MiniPC/Ubuntu26 Cluster!</h1>" > index.html
   sudo python3 -m http.server 80
   ```
   Từ trình duyệt macOS, truy cập địa chỉ `http://<FLOATING_IP>` để xác nhận trang web tải thành công. Chụp ảnh màn hình làm minh chứng.

---

### Module 4: Tích Hợp Lưu Trữ (Swift Object Storage & Cinder Block Storage)
**Mục tiêu**: Lưu mã nguồn trên Swift, gắn ổ đĩa Cinder để lưu trữ bền vững, chụp snapshot.

1. **Làm việc với Swift (Object Storage)**:
   *   Tạo container chứa mã nguồn:
       ```bash
       openstack container create tetris-src
       ```
   *   Tạo file source code demo trên macOS và upload lên Swift container:
       ```bash
       echo "<h1>Tetris Game Code Placeholder</h1>" > index.html
       openstack object create tetris-src index.html
       ```
   *   Tạo link tải tạm thời (TempURL) hoặc cấu hình container public để VM có thể tải file này:
       ```bash
       # Phân quyền container sang chế độ cho phép đọc public (đọc trực tiếp không cần token)
       openstack container set --read-acl ".r:*" tetris-src
       # Địa chỉ tải file từ máy ảo sẽ có dạng (sử dụng Tailscale IP của Control Node):
       # http://100.99.10.20:8080/v1/AUTH_<PROJECT_ID>/tetris-src/index.html
       ```

2. **Làm việc với Cinder (Block Storage)**:
   *   Tạo một Volume dung lượng 1 GB:
       ```bash
       openstack volume create --size 1 my-data-volume
       ```
   *   Gắn Volume này vào máy ảo:
       ```bash
       openstack server add volume hello-web-vm my-data-volume --device /dev/vdb
       ```
   *   SSH vào máy ảo, định dạng phân vùng và mount ổ đĩa:
       ```bash
       ssh ubuntu@$FIP_IP
       
       # Định dạng ổ đĩa vừa gắn (/dev/vdb)
       sudo mkfs.ext4 /dev/vdb
       
       # Mount vào thư mục chứa web
       sudo mkdir -p /var/www/html
       sudo mount /dev/vdb /var/www/html
       
       # Tải code từ Swift về ổ đĩa này
       # Thay đổi URL bằng thông tin Swift của bạn
       PROJECT_ID=$(openstack project show demo -f value -c id) # Hoặc project admin
       sudo wget -O /var/www/html/index.html http://100.99.10.20:8080/v1/AUTH_${PROJECT_ID}/tetris-src/index.html
       
       # Chạy lại HTTP Server trên thư mục mới này
       cd /var/www/html
       sudo python3 -m http.server 80 &
       ```

3. **Chụp Volume Snapshot & Khôi Phục**:
   *   Tắt web server trong VM và unmount ổ đĩa để đảm bảo toàn vẹn dữ liệu:
       ```bash
       sudo umount /var/www/html
       ```
   *   Gỡ Volume ra khỏi máy ảo qua CLI trên macOS:
       ```bash
       openstack server remove volume hello-web-vm my-data-volume
       ```
   *   Tạo snapshot cho Volume:
       ```bash
       openstack volume snapshot create --volume my-data-volume my-volume-snapshot
       ```
   *   Tạo một Volume mới từ snapshot vừa tạo:
       ```bash
       openstack volume create --snapshot my-volume-snapshot --size 1 my-restored-volume
       ```
   *   Tạo một máy ảo mới (`new-web-vm`), gắn `my-restored-volume` vào máy ảo mới này, mount ổ đĩa và kiểm tra xem dữ liệu file `index.html` tải từ Swift vẫn tồn tại bền vững hay không.

---

### Module 5: Tự Động Hóa Với Orchestration (Heat)
**Mục tiêu**: Sử dụng file template Heat để tự động hóa toàn bộ quá trình thiết lập mạng, máy ảo, volume và khởi chạy web server.

1. **Chuẩn bị Template**:
   Chúng ta sử dụng file [`module5_service.yaml`](file:///Users/caolegiaphu/Documents/cloud_aws/final_project/module5_service.yaml) đã được cấu hình tối ưu. Template này tự động:
   *   Tạo mạng `dmz-net`, `private-net`, subnet và kết nối Router.
   *   Tạo Security Group cho phép SSH, HTTP, Ping.
   *   Khởi tạo Volume 1 GB từ Cinder.
   *   Tạo máy ảo chạy Ubuntu, gắn 2 card mạng.
   *   Cấp phát và gán Floating IP vào máy ảo.
   *   Chạy script cloud-init để tự động: format ổ đĩa Cinder, mount ổ đĩa, fix lỗi asymmetric routing, và tải code từ Swift (nếu được truyền qua tham số `code_url`), sau đó khởi chạy web server Python.

2. **Triển khai Stack bằng CLI trên macOS**:
   Thực thi lệnh sau:
   ```bash
   # Lấy ID hoặc tên của Keypair đã tạo ở Module 3 (ví dụ: mykey)
   # Truyền link code từ Swift (hoặc bỏ trống để dùng trang Hello World mặc định)
   PROJECT_ID=$(openstack project show admin -f value -c id) # hoặc demo
   CODE_LINK="http://100.99.10.20:8080/v1/AUTH_${PROJECT_ID}/tetris-src/index.html"
   
   openstack stack create -t module5_service.yaml \
     --parameter key_name=mykey \
     --parameter code_url=$CODE_LINK \
     my-web-stack
   ```

3. **Kiểm tra trạng thái triển khai**:
   ```bash
   # Xem danh sách stack đang chạy
   openstack stack list
   
   # Xem chi tiết tài nguyên được tạo ra
   openstack stack resource list my-web-stack
   
   # Lấy kết quả đầu ra (IP Floating để truy cập web)
   openstack stack output show --all my-web-stack
   ```
   Chờ đến khi stack có trạng thái `CREATE_COMPLETE`. Lấy IP Floating từ output của stack và truy cập trên trình duyệt macOS để nghiệm thu kết quả dự án.

4. **Thu dọn tài nguyên (Dành cho việc dọn dẹp sau khi chấm điểm)**:
   ```bash
   openstack stack delete my-web-stack
   ```

---

## 5. Danh Sách Các Lỗi Thường Gặp & Cách Khắc Phục (Troubleshooting)

1.  **Lỗi kết nối MySQL / RabbitMQ từ Compute Node đến Control Node**:
    *   *Nguyên nhân*: UFW chặn cổng hoặc cấu hình IP trong `local.conf` bị sai.
    *   *Khắc phục*: Chạy lệnh `sudo ufw disable` trên cả hai node. Kiểm tra xem Compute Node có ping được IP của Control Node hay không.
2.  **Lỗi "No valid host was found" khi tạo Instance**:
    *   *Nguyên nhân*: Compute Node chưa được đăng ký thành công vào Control Node, hoặc Mini PC không đủ tài nguyên nhưng Compute Node chưa chạy.
    *   *Khắc phục*: Trên Control Node, chạy lệnh `openstack hypervisor list`. Đảm bảo thấy dòng thông tin của Compute Node (Ubuntu 26) ở trạng thái `State: up` và `Status: enabled`.
3.  **Lỗi không thể ping/SSH máy ảo từ macOS mặc dù đã gán Floating IP**:
    *   *Nguyên nhân*: Thiếu cấu hình route tĩnh trên macOS hoặc chưa mở Security Group.
    *   *Khắc phục*: Kiểm tra xem tính năng Subnet Router trên Tailscale đã được bật và phê duyệt chưa. Nếu cấu hình thủ công, hãy chạy lại lệnh thêm route tĩnh `sudo route -n add 172.24.4.0/24 100.99.10.20` trên macOS và đảm bảo IP Forwarding đã được bật trên Control Node. Đảm bảo Security Group của máy ảo đã mở cổng 22 và giao thức ICMP.
