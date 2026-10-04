# Bài 1: Thay đổi cổng kết nối SSH (SSH Port Hardening)


## 1. Mục tiêu & Môi trường

### Mục tiêu
- Thay đổi cổng dịch vụ SSH mặc định từ `22` sang `2222` trên máy chủ ảo nhằm hạn chế tối đa các cuộc quét cổng tự động của botnet trên mạng Internet.
- Cấu hình tường lửa UFW trước khi khởi động lại dịch vụ SSH daemon để tránh nguy cơ bị khóa quyền truy cập (Lockout).

### Môi trường & Ràng buộc
- **Tài khoản thực hiện:** `devops` (có quyền `sudo`, không thao tác bằng `root` trực tiếp).
- **Hệ điều hành:** Ubuntu Linux.
- **Tường lửa:** UFW (Uncomplicated Firewall).

---

## 2. Các bước thực hiện

### Bước 1: Cấu hình tường lửa UFW mở cổng 2222/tcp

Trước khi thay đổi cấu hình SSH daemon, tiến hành mở cổng `2222/tcp` trên tường lửa UFW để đảm bảo không bị ngắt kết nối sau khi áp dụng cấu hình mới.

```bash
devops@server:~$ sudo ufw allow 2222/tcp
Rules updated
Rules updated (v6)

devops@server:~$ sudo ufw enable
Command may disrupt existing ssh connections. Proceed with operation (y|n)? y
Firewall is active and enabled on system startup

devops@server:~$ sudo ufw status verbose
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)
To                         Action      From
--                         ------      ----
2222/tcp                   ALLOW IN    Anywhere                  
2222/tcp (v6)              ALLOW IN    Anywhere (v6)             
```

---

### Bước 2: Chỉnh sửa cấu hình SSH Daemon

Mở tệp cấu hình `/etc/ssh/sshd_config` bằng trình soạn thảo `nano`:

```bash
devops@server:~$ sudo nano /etc/ssh/sshd_config
```

Tìm chỉ thị `Port` (mặc định `#Port 22` hoặc `Port 22`) và sửa thành `Port 2222`:

```text
# Package-provided defaults for OpenSSH server
# see sshd_config(5) for details

Port 2222
#AddressFamily any
#ListenAddress 0.0.0.0
#ListenAddress ::
```

---

### Bước 3: Kiểm tra cú pháp và Khởi động lại Dịch vụ SSH

Kiểm tra cú pháp tệp cấu hình SSH bằng lệnh `sshd -t` trước khi khởi động lại dịch vụ:

```bash
devops@server:~$ sudo sshd -t
# Không hiển thị lỗi, cấu hình hợp lệ
```

Khởi động lại dịch vụ SSH daemon để áp dụng cổng mới:

```bash
devops@server:~$ sudo systemctl restart ssh
devops@server:~$ sudo systemctl status ssh
● ssh.service - OpenBSD Secure Shell server
     Loaded: loaded (/lib/systemd/system/ssh.service; enabled; vendor preset: enabled)
     Active: active (running) since Sun 2026-10-04 22:15:00 +07; 10s ago
   Main PID: 1845 (sshd)
      Tasks: 1 (limit: 1114)
     Memory: 2.3M
        CPU: 15ms
     CGroup: /system.slice/ssh.service
             └─1845 "sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 slots"
```

---

### Bước 4: Chặn cổng 22 cũ trên tường lửa UFW

Sau khi xác nhận dịch vụ SSH đã chuyển sang cổng `2222`, xóa quy tắc cho phép cổng 22 (nếu có) trên UFW:

```bash
devops@server:~$ sudo ufw delete allow 22/tcp
Rule deleted
Rule deleted (v6)
```

---

## 3. Kiểm tra & Xác nhận kết quả

*Lưu ý: Giữ nguyên phiên làm việc hiện tại, mở một cửa sổ Terminal mới trên máy cá nhân để kiểm tra kết nối.*

### 3.1. Kiểm tra kết nối qua cổng mới (Port 2222)

Thực hiện kết nối SSH tới máy chủ thông qua cổng `2222`:

```bash
user@local-machine:~$ ssh -p 2222 devops@103.179.x.x

Welcome to Ubuntu 22.04.4 LTS (GNU/Linux 5.15.0-101-generic x86_64)
Last login: Sun Oct  4 22:10:00 2026 from 113.190.x.x
devops@server:~$ 
```

**Kết quả:** Kết nối thành công tới máy chủ qua cổng `2222` bằng SSH Key.

---

### 3.2. Kiểm tra kết nối qua cổng cũ (Port 22)

Thực hiện kết nối SSH tới máy chủ thông qua cổng `22`:

```bash
user@local-machine:~$ ssh -p 22 devops@103.179.x.x

ssh: connect to host 103.179.x.x port 22: Connection refused
```

**Kết quả:** Kết nối qua cổng `22` bị từ chối hoàn toàn (`Connection refused` / `Timeout`).

---

## 4. Kết luận

- Đã thay đổi thành công cổng truy cập SSH từ `22` sang `2222`.
- Cấu hình UFW mở cổng mới trước khi khởi động lại dịch vụ SSH giúp tránh sự cố bị khóa khỏi hệ thống (lockout).
- Hệ thống đã sẵn sàng giảm thiểu tần suất bị quét cổng tự động từ botnet trên Internet.