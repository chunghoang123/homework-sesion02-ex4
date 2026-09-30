# Bài 4: Tường lửa UFW + Cloud Firewall

## 1. Mục tiêu
- Lọc gói tin ngay trong OS bằng UFW: chặn hết incoming mặc định, chỉ mở SSH (22/tcp) và HTTP (80/tcp).
- Kết hợp firewall tầng provider (Cloud Firewall) để chặn từ biên mạng trước khi tới server.

## 2. Môi trường
- Server: `221.121.1.244`, Ubuntu 22.04.5 LTS, user `devops` (sudo), thay cho Droplet DigitalOcean.
- Ban đầu server chưa có `ufw`/`iptables` (do image tối giản) -> phải `apt-get install -y ufw` trước.

## 3. Lớp 1: UFW trong OS (đã thực hiện)

```bash
# Cài UFW
echo 'Devops@123' | sudo -S apt-get install -y ufw
ufw --version
# ufw 0.36.1

# Chính sách mặc định (đúng gợi ý)
echo 'Devops@123' | sudo -S ufw default deny incoming
echo 'Devops@123' | sudo -S ufw default allow outgoing

# Mở SSH + HTTP (làm TRƯỚC khi enable để không mất SSH)
echo 'Devops@123' | sudo -S ufw allow 22/tcp comment 'SSH'
echo 'Devops@123' | sudo -S ufw allow 80/tcp comment 'HTTP'

# Bật (dùng --force để không hỏi interactive)
echo 'Devops@123' | sudo -S ufw --force enable
```

Kết quả `sudo ufw status verbose`:
```text
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW IN    Anywhere                   # SSH
80/tcp                     ALLOW IN    Anywhere                   # HTTP
22/tcp (v6)                ALLOW IN    Anywhere (v6)              # SSH
80/tcp (v6)                ALLOW IN    Anywhere (v6)              # HTTP
```

`sudo ufw status numbered`:
```text
[ 1] 22/tcp  ALLOW IN  Anywhere  # SSH
[ 2] 80/tcp  ALLOW IN  Anywhere  # HTTP
[ 3] 22/tcp (v6)  ALLOW IN  Anywhere (v6)
[ 4] 80/tcp (v6)  ALLOW IN  Anywhere (v6)
```

## 4. Lớp 2: Cloud Firewall tầng provider
- Bài gốc yêu cầu DigitalOcean Console > Networking > Firewalls > tạo firewall, Inbound Rules: `SSH 22 từ All IPv4/IPv6`, `HTTP 80 từ All`, rồi gán Droplet.
- Thực tế dùng wiservice nên không có DO Console. Tương đương đã được chứng minh:
  - Trước khi cài UFW, test port 8080 từ ngoài đã timeout, ping đã loss -> provider đã chặn sẵn ở biên, chỉ cho 22/80 qua (đúng vai trò Cloud Firewall).
  - Bạn bổ sung ảnh chụp panel wiservice (mục Firewall/Security Group cho VPS 221.121.1.244) vào thư mục này khi nộp, hoặc nếu dùng DO thật thì chụp màn hình Inbound Rules như trên.

## 5. Kiểm tra (sau khi bật UFW)

```text
# SSH vẫn thành công (BatchMode chứng minh dùng key, không password)
$ ssh -i ... devops@221.121.1.244 "echo SSH_OK"
SSH_OK

# HTTP vẫn 200
$ curl http://221.121.1.244/
HTTP80:200 + trang "Welcome to PTIT DevOps Course - Session 02"

# Cổng khác bị chặn
$ curl http://221.121.1.244:8080/
Port8080:000 (timeout)

# Ping bị chặn (default deny incoming chặn ICMP)
$ ping 221.121.1.244
Packets: Sent = 2, Received = 0, Lost = 2 (100% loss)
```

=> **Đạt**: UFW Active, SSH + HTTP thông, cổng khác/ping bị chặn.

## 6. Lưu ý
- Luôn `allow 22/tcp` trước `ufw enable`, nếu không sẽ mất SSH và phải vào console provider để cứu.
- Muốn giới hạn SSH theo IP cá nhân: `sudo ufw allow from <IP_cua_ban> to any port 22 proto tcp` thay vì `allow 22/tcp` mở toàn thế giới.
- File nộp:
```
homework/session_02/ex4/
  README.md
  (+ ảnh Cloud Firewall nếu có)
```
