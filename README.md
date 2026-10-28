# Lab8_AT-BMHTTT_1150080116_HoangMinhThang
# Lab: Local DNS Attack (DNS Spoofing & Cache Poisoning) trên pfSense
https://youtu.be/mz2eRGfFM1M
##  Tổng quan dự án
Dự án mô phỏng và thực nghiệm các kỹ thuật tấn công DNS cục bộ trong môi trường mạng cô lập được định tuyến và kiểm soát bởi tường lửa **pfSense**. Bài thực hành dựa trên kiến trúc của SEED Labs kết hợp với hệ điều hành hiện đại (**Ubuntu 24.04 LTS**, **BIND9 9.18+** và **pfSense 2.7+**).

###  Mục tiêu
1. **Thiết lập phân đoạn mạng an toàn**: Cấu hình mạng con `DNS-LAB` độc lập trên pfSense với bộ quy tắc tường lửa (Firewall Ruleset) cô lập, chặn kết nối ra WAN/Internet.
2. **Triển khai máy chủ DNS**: Cài đặt và cấu hình dịch vụ BIND9 đóng vai trò Recursive & Caching Resolver cho mạng nội bộ.
3. **Thực nghiệm 3 kỹ thuật tấn công DNS**:
   - **Task 1: HOSTS File Redirection** (Chuyển hướng cục bộ bằng tập tin cấu hình hệ điều hành).
   - **Task 2: Direct DNS Spoofing** (Đua gói tin và giả mạo phản hồi DNS trực tiếp tới Client).
   - **Task 3: DNS Cache Poisoning** (Đầu độc bộ nhớ đệm của BIND9 Resolver).
4. **Phân tích cơ chế và đề xuất giải pháp phòng thủ** (DNSSEC, DNS COOKIE, Layer 2 Hardening).

---

## 🏗️ Kiến trúc mô hình mạng (Topology)

| Thành phần | Hệ điều hành / Dịch vụ | Địa chỉ IP | Phân đoạn mạng (Interface) | Vai trò |
|---|---|---|---|---|
| **pfSense** | FreeBSD / pfSense Firewall | `10.9.0.1/24`<br>`172.16.6.1/24` | `em4` (DNSLAB)<br>`em3` (DMZ) | Gateway định tuyến và thực thi chính sách bảo mật cô lập |
| **DNS-SERVER** | Ubuntu Server 24.04 (BIND9) | `10.9.0.10/24` | Internal (`dns-lab`) | Recursive / Caching DNS Resolver nội bộ |
| **VICTIM** | Ubuntu 24.04 | `10.9.0.100/24` | Internal (`dns-lab`) | Máy trạm nạn nhân |
| **ATTACKER** | Ubuntu / Kali Linux | `10.9.0.200/24` | Internal (`dns-lab`) | Máy kẻ tấn công (Promiscuous Mode = Allow VMs) |
| **APP-TEST (AUTH)** | Ubuntu Server 24.04 | `172.16.6.20/24` | Internal (`lab6-dmz`) | Authoritative DNS mô phỏng (phản hồi trễ 1.5s) |
| **DMZ-WEB** | Nginx / Web Server | `172.16.6.10/24` | Internal (`lab6-dmz`) | Website hợp lệ trong vùng DMZ |

---

##  Chính sách Tường lửa pfSense (DNSLAB Rules)

pfSense áp dụng 5 luật theo thứ tự ưu tiên từ trên xuống dưới (First-Match) trên interface `DNSLAB`:

1. **Pass**: Cho phép `DNS-SERVER` (`10.9.0.10`) gửi truy vấn `UDP/TCP 53` sang máy chủ DMZ (`172.16.6.20`).
2. **Pass**: Cho phép mạng `DNSLAB` (`10.9.0.0/24`) truy cập máy chủ Web hợp lệ (`172.16.6.10:80`).
3. **Block + Log**: Chặn toàn bộ lưu lượng sang mạng nội bộ khác (`LAN: 10.6.0.0/24`).
4. **Block + Log**: Chặn toàn bộ lưu lượng sang mạng quản trị (`MGMT: 192.168.56.0/24`).
5. **Block + Log (Default Deny)**: Chặn toàn bộ lưu lượng còn lại, cô lập hoàn toàn mạng thử nghiệm không cho ra ngoài Internet (WAN).

---

## ⚙️ Các bước triển khai thực nghiệm

### 1. Baseline (Xác nhận trạng thái phân giải gốc)
- Kiểm tra truy vấn từ `VICTIM` đến `DNS-SERVER`:
  ```bash
  dig @10.9.0.10 www.external.lab9.test
  ```
- **Kết quả mong đợi**: Trả về `172.16.6.10` với độ trễ phản hồi hợp lệ khoảng 1.5 - 2.5 giây.

---

### 2. Task 1: Chuyển hướng bằng tập tin `/etc/hosts`
- **Mục tiêu**: Đánh giá độ ưu tiên của Name Service Switch (NSS).
- **Thực hiện trên VICTIM**:
  ```bash
  echo "10.9.0.200 www.lab9.test" | sudo tee -a /etc/hosts
  curl http://www.lab9.test
  dig @10.9.0.10 www.lab9.test
  ```
- **Hiện tượng**: `curl` chuyển hướng sang trang web giả mạo (`10.9.0.200`), nhưng `dig` vẫn trả về IP gốc (`172.16.6.10`) do `dig` bỏ qua NSS và truy vấn thẳng DNS Server.

---

### 3. Task 2: Direct DNS Spoofing (Giả mạo phản hồi trực tiếp tới Victim)
- **Cơ chế**: Attacker bật chế độ lắng nghe toàn mạng (Promiscuous Mode), bắt gói truy vấn DNS từ `VICTIM` và phát sinh gói tin phản hồi DNS giả mạo với IP nguồn giả của `DNS-SERVER`, khớp đúng `Transaction ID` và `UDP Port`.
- **Thực hiện trên ATTACKER**:
  ```bash
  sudo ip link set enp0s3 promisc on
  sudo python3 spoof.py
  ```
- **Thực hiện trên VICTIM**:
  ```bash
  dig @10.9.0.10 www.external.lab9.test
  ```
- **Kết quả**: Gói tin giả mạo thắng cuộc đua mạng (Query time: ~2 msec), Client nhận IP giả `10.9.0.200`.

---

### 4. Task 3: DNS Cache Poisoning (Đầu độc bộ nhớ đệm BIND9)
- **Cơ chế**: Attacker giả mạo máy chủ Authoritative (`172.16.6.20`), gửi phản hồi giả tới `DNS-SERVER` (`10.9.0.10`) trước khi gói phản hồi hợp lệ kịp đến. BIND9 tiếp nhận dữ liệu và lưu vào bộ nhớ đệm (Cache).
- **Thực hiện trên ATTACKER**: Chạy script đầu độc Resolver (`p3.py`).
- **Kích hoạt từ VICTIM**:
  ```bash
  dig @10.9.0.10 www.external.lab9.test
  ```
- **Kiểm tra cache trên DNS-SERVER**:
  ```bash
  sudo rndc dumpdb -cache
  sudo grep external /var/cache/bind/named_dump.db
  ```
- **Kết quả**: Bản ghi độc `www.external.lab9.test. IN A 10.9.0.200` được lưu giữ trong cache BIND9 với thời gian sống (TTL = 120s). Mọi client truy vấn sau đó đều bị điều hướng sai mà không phụ thuộc vào script tấn công.

---

### 5. Khôi phục môi trường (Restoration)
- Dừng toàn bộ script tấn công trên máy Attacker (`Ctrl + C`).
- Xóa sạch cache BIND9 trên máy chủ:
  ```bash
  sudo rndc flush
  ```
- Đặt lại chế độ card mạng VirtualBox của Attacker về **`Promiscuous Mode = Deny`**.
- Xác nhận trạng thái phân giải đã trở về IP thật `172.16.6.10`.

---

## Giải pháp Phòng thủ (Hardening Recommendations)

1. **DNSSEC (DNS Security Extensions)**: Triển khai ký số mật mã công khai (RRSIG, DNSKEY, DS) cho các zone. Resolver bắt buộc xác thực chữ ký trước khi chấp nhận vào cache.
2. **DNS COOKIE (RFC 7873)**: Sử dụng mã định danh 64-bit giữa client và server để chống giả mạo địa chỉ IP nguồn (IP Spoofing) trên giao thức UDP.
3. **Bảo mật Tầng 2 (Layer 2 Hardening)**: Tắt chế độ Promiscuous Mode trên hạ tầng ảo hóa; cấu hình **DHCP Snooping**, **Dynamic ARP Inspection (DAI)** và **Port Security** trên switch vật lý/ảo để ngăn nghe lén.
4. **Source Port & TXID Randomization**: Sử dụng bộ sinh số ngẫu nhiên có độ entropy cao cho cổng nguồn UDP và Transaction ID nhằm mở rộng không gian tìm kiếm lên $2^{32} \approx 4.3 \times 10^9$, triệt tiêu khả năng đoán trúng gói tin.
