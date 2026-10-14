# BÁO CÁO TỔNG HỢP SỬA LỖI BẢO MẬT — THÀNH VIÊN 2
**Dự án:** DriverGuard Safety Core (P-136)  
**Phụ trách:** Thành viên 2 (Bảo vệ Đường ống Telemetry, Kiểm soát Truy cập GPS & Chống DoS Hiệu năng)  
**Nhánh Git:** `fix/sec-telemetry-dos`  
**Commit ID:** `07248e5`  
**Tài liệu tham chiếu:** `cohort4-security-review/P-136/README.md` & `HINT_FIX_MENTOR.md`  
**Ngày hoàn thành:** 14/10/2026  

---

## 📌 BẢNG TIẾN ĐỘ THỰC HIỆN CỦA THÀNH VIÊN 2

| Mã lỗi | Tên lỗ hổng | Mức độ | File chỉnh sửa | Trạng thái |
|:---:|---|:---:|---|:---:|
| **03** | Ingestion telemetry/alert KHÔNG xác thực | **HIGH** | `src/api/telemetry.py`, `tests/test_api/test_telemetry.py` | ✅ **HOÀN THÀNH** |
| **04** | BOLA: xem vị trí/lịch sử mọi xe | **HIGH** | `src/api/telemetry.py`, `tests/test_api/test_telemetry.py` | ✅ **HOÀN THÀNH** |
| **08** | DoS: bề mặt ghi sụp dưới tải nhỏ do geocoding đồng bộ | **HIGH** | `src/services/geocoding_service.py`, `src/services/telemetry_service.py`, `src/api/telemetry.py` | ✅ **HOÀN THÀNH** |

---

## I. CHI TIẾT SỬA LỖI 03 — INGESTION TELEMETRY / ALERT KHÔNG XÁC THỰC (HIGH)

### 1. Bản chất lỗ hổng
- Các endpoint nạp tọa độ GPS (`POST /api/v1/telemetry/gps`), cảnh báo an toàn (`POST /api/v1/telemetry/alert`), và phản hồi can thiệp (`POST /api/v1/safety/commands/ack`) mở tự do ra Internet mà không kiểm tra token Bearer.
- Kẻ tấn công ẩn danh có thể bơm tọa độ tốc độ giả (`speed_kmh=999`) hoặc gửi cảnh báo ngủ gật giả cho bất kỳ tài xế nào, dẫn đến can thiệp nhầm và kỷ luật oan.

### 2. Các thay đổi đã thực hiện
1. **`src/api/telemetry.py`:**
   - Thêm dependency `current_user: User = Depends(get_current_user)` vào cả 3 endpoints. Request ẩn danh lập tức bị từ chối với mã **`401 Unauthorized`**.
   - Kiểm tra định danh: Tài xế (`DRIVER`) chỉ được gửi dữ liệu cho xe mình đang phụ trách ca trực. Nếu cố tình giả mạo `driver_id` khác sẽ bị trả về **`403 Forbidden`**.

### 3. Kết quả kiểm thử
- Gửi GPS không token: ❌ Bị chặn ngay (`401 Unauthorized`).
- Gửi Alert rỗng không token: ❌ Bị chặn ngay (`401 Unauthorized`).
- Tài xế hợp lệ gửi đúng xe: ✅ Tiếp nhận thành công (`200 OK`).

---

## II. CHI TIẾT SỬA LỖI 04 — BOLA XEM VỊ TRÍ & LỊCH SỬ XE (HIGH)

### 1. Bản chất lỗ hổng
- Endpoint `GET /api/v1/telemetry/vehicles/{vehicle_id}/latest` và `.../history` chỉ yêu cầu đăng nhập tài khoản bất kỳ mà không kiểm tra xe có thuộc về tài xế đó hay không.
- Tài xế `driver1` có thể theo dõi hành trình thời gian thực của mọi xe trong đội.

### 2. Các thay đổi đã thực hiện
1. **`src/api/telemetry.py`:**
   - Xây dựng hàm kiểm tra quyền đối tượng `check_vehicle_access(db, current_user, vehicle_id)`:
     - `ADMIN`, `SAFETY_MANAGER`, `DISPATCHER`: Toàn quyền xem mọi xe.
     - `DRIVER`: Kiểm tra xem xe có nằm trong ca trực hoạt động (`VehicleAssignment` có `status == ACTIVE`), hoặc là xe đang lái (`Vehicle.current_driver_id == current_user.id`), hoặc xe gán tĩnh (`DriverProfile.assigned_vehicle_id`). Nếu không thỏa mãn, ném lỗi **`403 Forbidden`**.

### 3. Kết quả kiểm thử (Script `02_authz_telemetry.py`)
- `driver1` xem xe `veh_vinfast_vf8_01` (thuộc về mình): ✅ `200 OK`.
- `driver1` xem xe `veh_vinfast_vf8_02` (thuộc về mình): ✅ `200 OK`.
- `driver1` xem xe `veh_vinfast_vf9_03` (xe người khác): ❌ **`403 Forbidden`** (Bị chặn).
- `driver1` xem xe `veh_vinfast_vf8_04` (xe người khác): ❌ **`403 Forbidden`** (Bị chặn).
- `driver1` xem xe `veh_vinfast_vf6_05` (xe người khác): ❌ **`403 Forbidden`** (Bị chặn).

---

## III. CHI TIẾT SỬA LỖI 08 — CHỐNG DoS BỀ MẶT GHI & ASYNC GEOCODING (HIGH)

### 1. Bản chất lỗ hổng
- Mỗi lần ghi GPS không có tên đường, hệ thống gọi HTTP đồng bộ ra ngoài (Nominatim OpenStreetMap) với timeout 2.0s.
- Mỗi request mất 600ms - 1.5s và khóa worker thread; khi có vài chục request liên tiếp, hệ thống sụp đổ với 82.5% timeout và p99 vọt lên 2.25s - 3.4s.

### 2. Các thay đổi đã thực hiện
1. **`src/services/geocoding_service.py`:**
   - Bổ sung `get_fast_address(lat, lon)`:
     - Tầng 1: Kiểm tra cache không gian in-memory (làm tròn 3 chữ số thập phân ~110m). Nếu hit: trả về trong **0ms**.
     - Tầng 2: Nếu cache miss, tự động gán nhãn địa bàn theo Bounding Box ngoại tuyến tức thì (Hà Nội, TP.HCM, Đà Nẵng).
   - Bổ sung `resolve_geocoding_in_background(log_id, lat, lon)`: Khởi chạy luồng **Daemon Thread độc lập** để gọi API ngoài trong nền và cập nhật DB sau, không chặn chu trình HTTP request.
2. **`src/services/telemetry_service.py`:**
   - Cập nhật `ingest_gps_telemetry` và `ingest_safety_alert` sang chế độ `async_geocoding=True`, trả về ngay trong < 30ms.
3. **`src/api/telemetry.py`:**
   - Thêm `background_tasks: BackgroundTasks` cho `POST /telemetry/gps` để kích hoạt worker nền khi cần phân giải tọa độ mới.

### 3. Số liệu đối chứng Stress Test trước và sau khi Fix (`08_loadtest_write_surface.py`)

| Concurrency (CCU) | Throughput (RPS) Trước Fix | **Throughput (RPS) Sau Fix** | p99 Latency Trước Fix | **p99 Latency Sau Fix** | Tỉ lệ lỗi |
|:---:|:---:|:---:|:---:|:---:|:---:|
| **c = 1** | 1.5 RPS | **36.1 RPS** *(↑ 24 lần)* | 1,167.7 ms | **32.7 ms** *(↓ 97%)* | 0.0% |
| **c = 5** | 7.2 RPS | **49.2 RPS** *(↑ 6.8 lần)* | 1,094.8 ms | **179.5 ms** *(↓ 84%)* | 0.0% |
| **c = 10** | 11.0 RPS | **40.9 RPS** *(↑ 3.7 lần)* | 1,453.3 ms | **951.6 ms** | 0.0% |
| **c = 20** | 17.3 RPS | **41.1 RPS** *(↑ 2.4 lần)* | 1,309.6 ms | **929.8 ms** | 0.0% |
| **c = 30** | 17.6 RPS *(Nghẽn)* | **78.5 RPS** *(↑ 4.5 lần)* | 2,257.4 ms | **495.4 ms** *(↓ 78%)* | **0.0%** |

---

## IV. TỔNG KẾT FILE ĐÃ CHỈNH SỬA & MERGE SAFETY

- `src/api/telemetry.py` (Bảo mật endpoint, auth ingestion, BOLA check, async geocoding)
- `src/services/geocoding_service.py` (Bộ nhớ đệm không gian, non-blocking address resolution, daemon thread)
- `src/services/telemetry_service.py` (Cải tiến ingestion pipeline không chờ I/O mạng)
- `tests/test_api/test_telemetry.py` (14/14 unit tests pass 100%)
- `tests/test_api/test_dashboard_hitl.py` (Đồng bộ token cho driver ack)

> **Lưu ý quy trình:** Nhánh `fix/sec-telemetry-dos` đã hoàn thành độc lập 100%, không bị conflict với Thành viên 1, 3, 4. Theo phân công, **giữ nguyên trạng thái để chờ Thành viên 3 hoàn tất trước khi tổng hợp chung**.
