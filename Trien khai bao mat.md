## Tóm tắt

Biên mạng: Đi qua Cloudflare Proxy trên domain c4-app-136.io.vn để ẩn 100% IP thật của máy chủ EC2, tự động cấp chứng chỉ HTTPS/SSL và chống DDoS.
Máy chủ: Cấu hình Nginx Reverse Proxy nhận kết nối từ Cloudflare, ép header HTTPS, hỗ trợ upload video lớn (100MB) và tối ưu truyền dữ liệu Realtime SSE.
Tường lửa (AWS Security Group): Đóng toàn bộ các cổng nhạy cảm (8000, 9000, 9001, 18083), không mở cổng Database ra Internet (dùng Supabase Cloud). Chỉ mở đúng 80, 443, 1883 (MQTT xe) và SSH Key 22. Thành viên muốn quản trị nội bộ thì dùng SSH Tunneling an toàn.*"

## 📌 1. Bức Tranh Tổng Thể (Architecture Flow)

Hệ thống DriverGuard được thiết kế theo nguyên tắc bảo mật **"Defense in Depth" (Phòng thủ đa tầng)**:

```
[ Người dùng / Mobile Driver / Web Frontend ]
                     │
                     ▼ (HTTPS - Port 443 - Ổ khóa xanh an toàn)
┌─────────────────────────────────────────────────────────────────────────┐
│ 1. TẦNG BIÊN MẠNG: CLOUDFLARE PROXY & WAF                               │
│    • Domain: c4-app-136.io.vn                                           │
│    • Ẩn giấu 100% IP máy chủ EC2 (15.134.111.175)                       │
│    • Cấp chứng chỉ Universal SSL tự động, chống DDoS L3/L4/L7           │
│    • Chế độ SSL/TLS: Flexible + Always Use HTTPS                        │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │ (HTTP Port 80 an toàn từ Cloudflare)
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 2. TẦNG HẠ TẦNG CLOUD: AWS EC2 SECURITY GROUP (Tối Giản & An Toàn)      │
│    • Port 80, 443: Mở công khai cho Web & API traffic từ Cloudflare     │
│    • Port 1883: Mở cho thiết bị IoT trên xe gửi dữ liệu Telemetry       │
│    • Port 22 (SSH): Quản trị từ xa bằng SSH Key                         │
│    • Port 8000, 9000, 9001, 18083, 5432: ĐÃ ĐÓNG HOÀN TOÀN KHỎI INTERNET │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 3. TẦNG HOST OS: NGINX REVERSE PROXY (Port 80)                          │
│    • Nhận request từ Cloudflare, gắn X-Forwarded-Proto https            │
│    • Mở client_max_body_size 100M để upload video/bằng chứng sự kiện   │
│    • Tắt proxy_buffering để stream SSE realtime mượt mà                 │
│    • Chuyển tiếp an toàn vào Frontend (Port 8080) và Backend API        │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ 4. TẦNG DOCKER & DATABASE CÔ LẬP NỘI BỘ (Zero Trust Network)            │
│    • driverguard_frontend (Port 8080): Phục vụ React SPA & Web          │
│    • driverguard_backend  (Port 8000): FastAPI Core Service             │
│    • driverguard_emqx     (Port 1883): MQTT Broker kết nối đội xe       │
│    • driverguard_minio    (Port 9000, 9001): Bind nội bộ 127.0.0.1      │
│    • Database: Supabase Cloud Postgres (Port 6543 ngoại vi, không mở EC2)│
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 🛠️ 2. Chi Tiết Các Tầng Bảo Vệ Đã Hoàn Thành

### Tầng 1: Cloudflare Edge (DNS, Proxy & WAF)
* **Ẩn IP gốc (Origin IP Masking):** Người dùng khi truy cập hoặc ping `c4-app-136.io.vn` chỉ thấy dải IP proxy của Cloudflare (`104.21.x.x`), hoàn toàn che giấu IP máy chủ EC2 (`15.134.111.175`).
* **Universal SSL:** Toàn bộ kết nối giữa trình duyệt/app di động và Cloudflare đều được mã hóa bằng HTTPS có ổ khóa xanh chuẩn quốc tế.
* **Xử lý triệt để lỗi chuyển hướng:** Đã dọn dẹp các bản ghi DNS rác cũ (`103.x.x.x`), chỉ giữ duy nhất bản ghi trỏ về máy chủ EC2, giải quyết dứt điểm lỗi `ERR_TOO_MANY_REDIRECTS`.

---

### Tầng 2: AWS Security Group (Tường Lửa Cloud)
Áp dụng nghiêm ngặt nguyên tắc **"Least Privilege" (Quyền tối thiểu)**, bảng Inbound Rules chỉ mở đúng **4 cổng cần thiết**:
1. `80` (HTTP): Nhận kết nối sạch từ Cloudflare.
2. `443` (HTTPS): Nhận kết nối web bảo mật.
3. `1883` (Custom TCP): Cho thiết bị xe ô tô / Edge Mobile gửi dữ liệu telemetry MQTT.
4. `22` (SSH): Cho phép nhóm quản trị kết nối terminal bằng SSH Key.
* **Đã đóng hoàn toàn:** Cổng `8000` (FastAPI), `9000` (MinIO API), `9001` (MinIO Web), `18083` (EMQX Web), và cổng Database `5432/5433`.

---

### Tầng 3: Host Nginx Reverse Proxy
Cấu hình tại `/etc/nginx/sites-available/default`:
* **Ép Header HTTPS:** `proxy_set_header X-Forwarded-Proto https;` giúp Backend nhận biết phiên làm việc đã được mã hóa, không bị lỗi Mixed Content.
* **Hỗ trợ tải file/video lớn:** `client_max_body_size 100M;` tránh lỗi HTTP 413 Payload Too Large khi tài xế gửi clip vi phạm.
* **Tối ưu truyền dữ liệu Realtime SSE:** `proxy_buffering off;` và `proxy_read_timeout 86400s;` đảm bảo kênh SSE đẩy cảnh báo an toàn tức thì về màn hình giám sát mà không bị nghẽn đệm.

---

### Tầng 4: Docker & Database Cô Lập Nội Bộ
* **Database trên Cloud Supabase (Cổng 6543):** Hệ thống dùng Supabase Cloud Database (AWS Tokyo). Máy chủ EC2 đóng vai trò client gọi ra ngoài, do đó **không cần mở bất kỳ cổng Database nào trên EC2**.
* **MinIO bind nội bộ `127.0.0.1`:** Cổng 9000 và 9001 của MinIO chỉ lắng nghe trên localhost máy chủ, ngăn chặn tuyệt đối các đợt quét cổng từ bên ngoài Internet.
* **Toàn bộ Secret chuyển vào `.env`:** JWT secret, mật khẩu root MinIO, database URL đều được bảo mật trong file môi trường, không bị commit lên GitHub.

---

## 🔑 3. HƯỚNG DẪN DÀNH CHO TEAM TRUY CẬP CÁC CỔNG ĐÃ ĐÓNG (8000, 9001, 18083)

Vì lý do bảo mật, các cổng 8000, 9000, 9001, 18083 đã được đóng khỏi Internet công cộng. Dưới đây là cách các thành viên trong nhóm truy cập bình thường:

### Cách 1: Gọi API Backend (Dành cho Test API, Postman, Mobile App)
- **Không dùng:** `http://15.134.111.175:8000/api/v1/...` (cổng 8000 đã đóng).
- **Dùng link chính thức qua HTTPS:**  
  👉 **`https://c4-app-136.io.vn/api/v1/...`**  
  *(Đã có HTTPS ổ khóa xanh, đi qua Cloudflare và Nginx tự động chuyển tiếp vào Backend).*

---

### Cách 2: Vào MinIO Console (9001) & EMQX Dashboard (18083) bằng SSH Tunnel
Khi cần vào Web quản trị MinIO xem video hoặc vào EMQX xem MQTT broker, bạn chỉ cần tạo 1 "đường hầm bí mật" (SSH Tunnel) từ máy mình vào server:

#### 👉 Dành cho ai dùng Terminal (Mac, Linux, Powershell / Git Bash trên Windows):
# ae nào cần thì t gửi file pem cho
Mở terminal trên máy tính cá nhân (tại thư mục chứa file key `.pem`), gõ lệnh:
```bash
# Lệnh mở đường hầm cho MinIO Console (9001) và EMQX Dashboard (18083):
ssh -i ec2_key.pem -L 9001:localhost:9001 -L 18083:localhost:18083 ubuntu@15.134.111.175
```
*Sau khi gõ lệnh trên và giữ cửa sổ terminal mở:*
- **Vào MinIO Console:** Mở trình duyệt gõ 👉 `http://localhost:9001` (đăng nhập bằng user/pass admin).
- **Vào EMQX Dashboard:** Mở trình duyệt gõ 👉 `http://localhost:18083` (tài khoản `admin` / `12345678@`).

*(Nếu cần test trực tiếp backend port 8000 mà không qua domain, chỉ cần thêm `-L 8000:localhost:8000` vào lệnh trên rồi vào `http://localhost:8000/docs`).*

---

