# Báo Cáo Thực Hành: Cấu Hình Tường Lửa UFW Và Chẩn Đoán Cổng Mạng

- **Bài tập**: Bài 3 - Cấu hình tường lửa UFW và chẩn đoán cổng mạng (Session 06 / Linux Network & Firewall Management)
- **Đường dẫn nộp bài**: `SS06/Bai03/README.md` (tương đương `homework/session_06/ex3/`)
- **Tài khoản thực hiện**: `devops` (Sudoer)
- **Dịch vụ tường lửa**: UFW (`Uncomplicated Firewall`)

---

## 1. Bối cảnh & Mục tiêu

### Bối cảnh
Khi triển khai ứng dụng web chạy ở cổng tùy chỉnh (ví dụ: `8080/tcp`) trên máy chủ Cloud VPS công khai, việc mở toàn bộ giao tiếp mạng là cực kỳ nguy hiểm. Máy chủ cần được trang bị một bức tường lửa (Firewall) đóng vai trò lá chắn, mặc định chặn toàn bộ lưu lượng truy cập từ ngoài vào (Incoming traffic) ngoại trừ các cổng dịch vụ được phép rõ ràng.

### Mục tiêu
- Cấu hình chính sách mặc định của tường lửa UFW:
  - **Chặn toàn bộ lưu lượng đi vào hệ thống** (`deny incoming`).
  - **Cho phép toàn bộ lưu lượng đi ra khỏi hệ thống** (`allow outgoing`).
- Mở cổng **22/tcp** dành cho dịch vụ quản trị SSH để tránh nguy cơ bị khóa quyền truy cập máy chủ (Lockout).
- Mở cổng **8080/tcp** phục vụ cho ứng dụng web.
- Kích hoạt UFW và kiểm tra trạng thái quy tắc mạng chi tiết bằng `sudo ufw status verbose`.
- Sử dụng công cụ `ss` (Socket Statistics) để chẩn đoán các cổng đang lắng nghe (`LISTEN`) thực tế trên hệ thống.

---

## 2. Môi trường triển khai

- **Hệ điều hành**: Ubuntu 22.04 / 24.04 LTS (WSL2 / Linux Virtual Machine / Cloud VPS)
- **Tường lửa mặc định**: UFW (`Uncomplicated Firewall`)
- **Công cụ chẩn đoán mạng**: `ss` (Socket Statistics) / `curl`
- **Cổng dịch vụ cần mở**:
  - `22/tcp`: SSH Daemon (`sshd`)
  - `8080/tcp`: Custom Web Application Server

---

## 3. Các bước thực hiện chi tiết

### Bước 1: Thiết lập chính sách mặc định (Default Policies)

Trước khi kích hoạt tường lửa, thiết lập chính sách bảo mật mặc định cho hệ thống:

```bash
# 1. Chặn toàn bộ kết nối từ ngoài đi vào máy chủ
sudo ufw default deny incoming

# 2. Cho phép máy chủ chủ động tạo kết nối đi ra ngoài
sudo ufw default allow outgoing
```

---

### Bước 2: Thêm quy tắc mở cổng dịch vụ cần thiết

> ⚠️ **LƯU Ý QUAN TRỌNG**: Luôn thêm quy tắc cho phép kết nối cổng SSH (`22/tcp`) **trước khi bật UFW** để tránh bị ngắt kết nối SSH hiện tại và bị khóa khỏi server (SSH Lockout).

```bash
# 1. Cho phép kết nối SSH qua cổng 22/tcp
sudo ufw allow 22/tcp comment 'Allow SSH Management'

# 2. Cho phép kết nối Web App qua cổng 8080/tcp
sudo ufw allow 8080/tcp comment 'Allow Web App Port 8080'
```

---

### Bước 3: Kích hoạt tường lửa UFW

Sau khi đã định nghĩa xong các quy tắc an toàn, tiến hành bật tường lửa:

```bash
sudo ufw enable
```

*(Nhấn `y` để xác nhận kích hoạt khi UFW cảnh báo về kết nối SSH)*

---

### Bước 4: Chẩn đoán các cổng mạng đang lắng nghe thực tế (`ss`)

Sử dụng công cụ `ss` để kiểm tra tất cả các socket TCP đang ở trạng thái `LISTEN` trên server:

```bash
sudo ss -tlnp
```

Các cờ (flags) sử dụng trong lệnh:
- `-t`: Hiển thị kết nối TCP.
- `-l`: Chỉ lấy các socket đang lắng nghe (`LISTEN`).
- `-n`: Hiển thị dạng số (Port Number) thay vì hiển thị tên dịch vụ.
- `-p`: Hiển thị Process ID (PID) và tên tiến trình đang nắm giữ cổng.

---

## 4. Trích xuất Log Terminal Chứng Minh (Verification Logs)

### 1. Kiểm tra trạng thái tường lửa UFW chi tiết (`sudo ufw status verbose`)

```console
devops@ubuntu-server:~$ sudo ufw status verbose
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW IN    Anywhere                   # Allow SSH Management
8080/tcp                   ALLOW IN    Anywhere                   # Allow Web App Port 8080
22/tcp (v6)                ALLOW IN    Anywhere (v6)              # Allow SSH Management
8080/tcp (v6)              ALLOW IN    Anywhere (v6)              # Allow Web App Port 8080
```

---

### 2. Kiểm tra các cổng lắng nghe trên máy chủ (`sudo ss -tlnp`)

```console
devops@ubuntu-server:~$ sudo ss -tlnp
State      Recv-Q Send-Q Local Address:Port               Peer Address:Port              Process                                                                                                                                   
LISTEN     0      128          0.0.0.0:22                  0.0.0.0:*                  users:(("sshd",pid=845,fd=3))                                                                                                             
LISTEN     0      511          0.0.0.0:8080                0.0.0.0:*                  users:(("node",pid=1532,fd=18))                                                                                                           
LISTEN     0      128             [::]:22                     [::]:*                  users:(("sshd",pid=845,fd=4))                                                                                                             
LISTEN     0      511             [::]:8080                   [::]:*                  users:(("node",pid=1532,fd=19))                                                                                                           
```

*Kết quả phân tích*:
- Tường lửa báo trạng thái `Status: active`.
- Cổng `22/tcp` và `8080/tcp` được ghi nhận `ALLOW IN` từ mọi địa chỉ IP (`Anywhere`).
- Lệnh `ss -tlnp` xác nhận tiến trình SSH daemon (`sshd`) đang lắng nghe trên cổng `22` và ứng dụng Web (`node`) đang lắng nghe trên cổng `8080`.

---

## 5. Bảng Phân Tích Luật UFW (Firewall Rules Summary)

| Cổng (Port) | Giao thức | Hành động (Action) | Nguồn (From) | Mục đích sử dụng |
| :---: | :---: | :---: | :---: | :--- |
| `22/tcp` | TCP | `ALLOW IN` | Anywhere | Kết nối quản trị hệ thống qua SSH |
| `8080/tcp` | TCP | `ALLOW IN` | Anywhere | Truy cập ứng dụng Web công khai |
| *Tất cả cổng khác* | *Tất cả* | `DENY IN` | Anywhere | Chặn toàn bộ truy cập không hợp lệ từ bên ngoài |

---

## 6. Kết luận

- Đã thiết lập thành công tường lửa UFW với chính sách bảo mật nghiêm ngặt (`default deny incoming`).
- Cấu hình mở thành công cổng `22/tcp` (SSH) và `8080/tcp` (Web App), loại bỏ rủi ro bị lockout hệ thống.
- Sử dụng thành thạo công cụ chẩn đoán mạng `ss -tlnp` và `ufw status verbose` để kiểm tra hạ tầng mạng trước khi đưa ứng dụng vào vận hành sản phẩm.
