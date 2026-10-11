# KẾ HOẠCH HOÀN THIỆN TÍNH NĂNG "TIẾP TỤC THEO DÕI" (ACTIVE MONITORING)
### DRIVERGUARD — PHÂN HỆ CAN THIỆP AN TOÀN HITL

> **Mục tiêu:** Chuyển đổi tùy chọn *"Tiếp tục theo dõi"* từ nút đóng popup thuần túy thành quy trình **Giám sát chủ động có lưu vết (Watchlist & Audit-backed)** xuyên suốt giữa **Backend $\leftrightarrow$ Frontend $\leftrightarrow$ Mobile App**.

---

## 1. Backend (BE) — Xử lý Nghiệp vụ & Vòng đời Ca

- [ ] **Mở rộng Model & Schema:**
  - Bổ sung trạng thái ca: `HitlStatus.MONITORING` (hoặc `DEFERRED_MONITORING`).
  - Thêm trường vào bảng `hitl_cases`: `monitoring_expires_at` (thời điểm hết hạn theo dõi), `escalation_count` (số lần vi phạm lặp lại).

- [ ] **API Xử lý Hoãn can thiệp & Giám sát (`POST /api/v1/hitl/cases/{case_id}/defer-monitoring`):**
  - Nhận body: `{ "snooze_minutes": 10, "manager_note": "Tài xế có dấu hiệu mỏi mắt nhẹ, theo dõi thêm 10 phút" }`.
  - Cập nhật ca: `status = MONITORING`, lưu `adjudicated_by_id`, `adjudicated_at`.

- [ ] **Ghi Sổ cái kiểm toán bất biến WORM (`safety_audit_ledgers`):**
  - Ghi nhận `action_type = "ADJUDICATION_DEFERRED_MONITORING"`.
  - Ký chữ ký HMAC-SHA256, lưu snapshot thời điểm hoãn và lý do của Manager để bảo vệ trách nhiệm pháp lý nếu xe xảy ra sự cố sau đó.

- [ ] **Phát sự kiện Realtime & Điều khiển:**
  - **SSE:** Phát `hitl_case_monitored` đến tất cả Web Dashboard để cập nhật trạng thái xe.
  - **MQTT:** Publish lệnh `SET_HIGH_FREQUENCY_TELEMETRY` xuống topic `command/driver/{driver_id}` để tăng tần suất gửi dữ liệu buồng lái.

- [ ] **Bộ đếm thời gian tự động (Background Worker / Celery / APScheduler):**
  - **Kịch bản A (An toàn):** Sau $X$ phút không có vi phạm mới $\rightarrow$ Tự động đóng ca (`AUTO_RESOLVED_SAFE`), giải phóng xe khỏi diện theo dõi.
  - **Kịch bản B (Tái phạm):** Nếu trong thời gian $X$ phút phát hiện vi phạm mới $\rightarrow$ Gom ca, nâng cấp thành **Cảnh báo khẩn cấp Cấp 2 (Escalated Alert)** và đẩy ngược lại Web Dashboard.

---

## 2. Frontend Web (FE) — Trực quan hóa & Thao tác Quản lý

- [ ] **Cập nhật Modal Can thiệp (`EmergencyModal.jsx`):**
  - Khi bấm *"Tiếp tục theo dõi"*, hiển thị popover nhanh:
    - Chọn thời gian theo dõi (5 phút / 10 phút / 15 phút).
    - Nhập ghi chú nhanh (mặc định sẵn gợi ý).
  - Gọi API `hitlApi.deferMonitoring(caseId, payload)` thay vì chỉ gọi `onClose()`.

- [ ] **Cập nhật Bản đồ Số & Thẻ Xe (`DashboardPage.jsx`):**
  - Đổi màu trạng thái của xe sang **Màu vàng / Cam nhấp nháy** (`MONITORED`).
  - Hiển thị badge: *"Đang giám sát đặc biệt (còn 08:45)"*.

- [ ] **Hàng đợi HITL (Queue Management):**
  - Tách hoặc gắn tab riêng: **"Đang theo dõi" (Watchlist)**.
  - Đếm ngược thời gian Snooze của từng ca; khi hết hạn hoặc khi có cảnh báo leo thang thì tự động kích hoạt âm báo nhắc nhở Manager.

- [ ] **Lắng nghe sự kiện SSE:**
  - Bắt sự kiện `hitl_case_monitored` và `hitl_case_escalated` để làm mới danh sách xe và bản đồ theo thời gian thực mà không cần F5.

---

## 3. Mobile App (Android DMS) — Giám sát buồng lái Tăng cường

- [ ] **Lắng nghe Lệnh Giám sát qua MQTT:**
  - Subscribe và xử lý payload từ topic `command/driver/{driver_id}` với lệnh `SET_MONITORING_MODE`:
    - Nhận tham số: `duration_seconds` (thời gian giám sát), `gps_interval_ms` (chu kỳ gửi GPS).

- [ ] **Chế độ Giám sát ngầm (Silent High-Sensitivity Mode):**
  - **Không làm phiền tài xế:** Không hiện popup hay hú còi cảnh báo gây căng thẳng không cần thiết trong thời gian đang theo dõi.
  - **Tăng tần suất Telemetry:** Rút ngắn chu kỳ gửi GPS và trạng thái từ `5s` xuống `1s - 2s` để Server vẽ vết di chuyển chi tiết hơn.
  - **Tăng độ nhạy AI Edge (Tạm thời):**
    - Hạ ngưỡng phát hiện nhắm mắt/gục đầu (ví dụ: EAR $\le 0.20$ thay vì $\le 0.18$, hoặc thời gian nhắm mắt giảm từ $2.0s \rightarrow 1.2s$).
    - Nếu vi phạm tái diễn $\rightarrow$ Gắn cờ vi phạm `is_escalated = true` trong bản tin MQTT gửi lên server để kích hoạt can thiệp cưỡng chế ngay lập tức.
