# BÁO CÁO TỔNG HỢP SỬA LỖI BẢO MẬT — THÀNH VIÊN 1
**Dự án:** DriverGuard Safety Core (P-136)  
**Phụ trách:** Thành viên 1 (Hạ tầng, Biến môi trường & Quản lý Secret)  
**Tài liệu gốc:** `my-folder/cohort4-security-review/guide.md` & `P-136/README.md`  
**Ngày hoàn thành:** 2026-10-14  

---

## 📌 BẢNG TIẾN ĐỘ THỰC HIỆN CỦA THÀNH VIÊN 1

| Mã lỗi | Tên lỗ hổng | Mức độ | File chỉnh sửa | Trạng thái |
|:---:|---|:---:|---|:---:|
| **01** | Giả mạo JWT secret chiếm quyền ADMIN | **CRITICAL** | `src/config.py`, `.env`, `.env.example`, `docker-compose.yml` | ✅ **HOÀN THÀNH** |
| **02** | MinIO dùng credential mặc định & mở cổng 9000 | **CRITICAL** | `src/config.py`, `.env`, `.env.example`, `docker-compose.yml` | ✅ **HOÀN THÀNH** |
| **06** | Hardening: Token sống 30 ngày & CORS lỏng lẻo | **LOW/INFO** | `src/config.py`, `src/main.py`, `.env`, `.env.example` | ✅ **HOÀN THÀNH** |

---

## I. CHI TIẾT SỬA LỖI 01 — GIẢ MẠO JWT SECRET (CRITICAL)

### 1. Bản chất lỗ hổng
- Chuỗi khóa bí mật `jwt_secret_key` bị gán cứng mặc định trong code: `"driverguard-super-secret-key-ai20k-2026-production"`.
- Trên server LIVE không override biến môi trường này.
- Kẻ tấn công đọc repo, tự viết script Python dùng thuật toán HS256 ký token với `role: ADMIN` và `sub: usr_admin_0001`, bypass xác thực và chiếm toàn quyền hệ thống.

### 2. Các thay đổi đã thực hiện
1. **`src/config.py`:**
   - Xóa bỏ chuỗi hardcode, đổi thành `jwt_secret_key: str = Field(..., min_length=32)` bắt buộc nạp từ biến môi trường.
   - Thêm validator **Fail-Fast** (`@field_validator("jwt_secret_key")`):
     - Chặn khởi động nếu biến rỗng hoặc độ dài < 32 ký tự.
     - **Chặn cứng chuỗi cũ:** Nếu phát hiện cấu hình lại chuỗi bí mật cũ đã bị lộ, ném ngoại lệ dừng khởi động ngay.
2. **`.env`:** Sinh khóa ngẫu nhiên an toàn 256-bit (64 ký tự URL-safe) bằng CSPRNG `secrets.token_urlsafe(48)`:
   ```env
   JWT_SECRET_KEY=zH7k4_ZSMv-XYh-Tx8rPwf_T5UoMdjlFN77rtvmivZzhkHvEqZ41dMndztM6AeW7
   JWT_ALGORITHM=HS256
   ```
3. **`.env.example`:** Thêm mẫu khai báo kèm câu lệnh sinh key cho các thành viên khác.
4. **`docker-compose.yml`:** Truyền biến `JWT_SECRET_KEY: ${JWT_SECRET_KEY}` vào service `backend`.

### 3. Kết quả kiểm thử
- Thử nạp key rỗng: ❌ Dừng app ngay (`CRITICAL SECURITY: JWT_SECRET_KEY is empty!`).
- Thử nạp key cũ bị lộ: ❌ Bị chặn ngay (`Banned old hardcoded JWT_SECRET_KEY!`).
- Mọi token cũ do hacker/mentor ký trước đây đều lập tức bị vô hiệu hóa (`401 Invalid Signature`).

---

## II. CHI TIẾT SỬA LỖI 02 — MINIO DEFAULT CREDENTIALS (CRITICAL)

### 1. Bản chất lỗ hổng
- `src/config.py` và `docker-compose.yml` đặt tài khoản/mật khẩu mặc định:
  - `object_storage_access_key = "driverguard"`
  - `object_storage_secret_key = "c4appt136iovn"`
- Server deploy mở công khai cổng API `9000` ra Internet (`0.0.0.0:9000`). Kẻ tấn công kết nối vào cổng 9000 đọc/xóa 98 video dashcam buồng lái chứa dữ liệu sinh trắc khuôn mặt tài xế.

### 2. Các thay đổi đã thực hiện
1. **`docker-compose.yml` — Đóng cổng khỏi Internet công cộng:**
   - Đóng cổng 9000 và 9001 khỏi mạng Internet công cộng bằng cách bind chỉ lắng nghe trên `127.0.0.1` (localhost):
     ```yaml
     ports:
       - "127.0.0.1:${MINIO_API_PORT:-9000}:9000"
       - "127.0.0.1:${MINIO_CONSOLE_PORT:-9001}:9001"
     ```
   - Xóa bỏ fallback mặc định `:-driverguard` và `:-c4appt136iovn` trong biến môi trường container MinIO.
   - Container `backend` giao tiếp với MinIO hoàn toàn qua mạng nội bộ Docker (`http://minio:9000`).
2. **`src/config.py` — Bỏ hardcode & thêm Fail-Fast:**
   - Xóa IP cứng `15.134.111.175`, chuyển `object_storage_public_endpoint_url` về `http://localhost:9000`.
   - Đổi `object_storage_access_key` và `object_storage_secret_key` thành `Field(...)` bắt buộc nạp từ biến môi trường.
   - Thêm validator Fail-Fast chặn rỗng và chặn mật khẩu cũ `'c4appt136iovn'`.
3. **`.env`:** Cập nhật tài khoản quản trị mới kèm mật khẩu mạnh 32 ký tự:
   ```env
   MINIO_ROOT_USER=driverguard_admin
   MINIO_ROOT_PASSWORD=7AE71NkviJeObDHN0fi2E38QUcHFJ9HK
   OBJECT_STORAGE_ACCESS_KEY=driverguard_admin
   OBJECT_STORAGE_SECRET_KEY=7AE71NkviJeObDHN0fi2E38QUcHFJ9HK
   OBJECT_STORAGE_PUBLIC_ENDPOINT_URL=http://localhost:9000
   ```
4. **`.env.example`:** Xóa sạch credential cũ, thay bằng placeholder hướng dẫn bảo mật.

### 3. Kết quả kiểm thử
- Thử nạp mật khẩu cũ `c4appt136iovn`: ❌ Bị chặn ngay (`Banned old default MinIO password`).
- Thử để trống access key: ❌ Bị chặn ngay (`String should have at least 3 characters`).
- Cổng 9000 được cô lập, chỉ truy cập được từ localhost/nội bộ backend.

---

## III. CHI TIẾT SỬA LỖI 06 — HARDENING TOKEN EXPIRY & CORS (LOW / INFO)

### 1. Bản chất lỗ hổng
- **Thời hạn Access Token quá dài:** `access_token_expire_minutes` được đặt là `43200` phút (tương đương **30 ngày**). Nếu token bị lộ, kẻ tấn công có thể sử dụng token trong 1 tháng mà không có cơ chế thu hồi (CWE-613).
- **CORS lỏng lẻo:** `CORSMiddleware` cấu hình `allow_methods=["*"]` và `allow_headers=["*"]` đi kèm với `allow_credentials=True`. Chuỗi `cors_origins` không được lọc và chuẩn hóa.

### 2. Các thay đổi đã thực hiện
1. **Rút ngắn thời hạn token (`src/config.py`):**
   - Đổi `access_token_expire_minutes` từ `43200` (30 ngày) xuống **`60` phút**.
   - Đặt `refresh_token_expire_days = 7` (dùng refresh token để cấp mới access token định kỳ).
2. **Thắt chặt cấu hình CORS (`src/main.py`):**
   - Chuẩn hóa và làm sạch danh sách domain cho phép:
     ```python
     settings = get_settings()
     cors_origins = [o.strip() for o in settings.cors_origins.split(",") if o.strip()]
     app.add_middleware(
         CORSMiddleware,
         allow_origins=cors_origins,
         allow_credentials=True,
         allow_methods=["GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"],
         allow_headers=["Authorization", "Content-Type", "Accept", "X-Requested-With"],
     )
     ```
   - Thay thế các wildcard `["*"]` bằng danh sách các phương thức HTTP và headers cần thiết của ứng dụng.
3. **Cập nhật `.env` và `.env.example`:**
   - Khai báo rõ ràng:
     ```env
     ACCESS_TOKEN_EXPIRE_MINUTES=60
     REFRESH_TOKEN_EXPIRE_DAYS=7
     ```

### 3. Kết quả kiểm thử
- Hệ thống nạp thành công cấu hình token mới:
  - `Access Token Expire: 60 phút`
  - `Refresh Token Expire: 7 ngày`
- FastAPI khởi tạo thành công middleware CORS với đầy đủ các phương thức giới hạn an toàn.

---

## IV. TỔNG KẾT FILE ĐÃ CHỈNH SỬA
- `src/config.py` (Bảo mật JWT, MinIO, Token expiry, Fail-Fast validators)
- `src/main.py` (Thắt chặt CORS Middleware)
- `docker-compose.yml` (Đóng cổng MinIO 9000 khỏi Internet, bổ sung biến JWT)
- `.env` (Cấu hình secret ngẫu nhiên, tài khoản MinIO an toàn)
- `.env.example` (Template chuẩn hóa cho các thành viên và server)
