# HƯỚNG DẪN TÍCH HỢP HỆ THỐNG ĐIỂM AN TOÀN TÀI XẾ (DRIVER SAFETY SCORE & LEDGER)
> **Dự án:** DriverGuard — Core Safety Platform (AI20K Build Phase)  
> **Phiên bản:** v1.0 (Quy tắc tinh gọn: Trừ điểm theo cảnh báo, Hồi phục an toàn, Sổ cái minh bạch & Chế tài)  
> **Đối tượng áp dụng:** Backend API, Web Dashboard (React/Vite), Mobile App (Android Java)

---

## 1. TỔNG QUAN HỆ THỐNG ĐÃ TINH GỌN (CORE RULES)

Để phù hợp với thời gian phát triển và khả năng nhận diện hiện tại của Edge AI, hệ thống tính điểm an toàn tài xế (`safety_score` $\in [0, 100]$) được tinh gọn thành **4 quy tắc trọng tâm**:

| STT | Trụ cột | Quy tắc triển khai | Chi tiết kỹ thuật |
| :---: | :--- | :--- | :--- |
| **1** | **Trừ điểm theo cảnh báo** | • **Mức Cảnh báo (`WARNING`):** Trừ **`2.0 điểm`**<br>• **Mức Nguy cấp (`CRITICAL`):** Trừ **`5.0 điểm`** | Trừ điểm khi Điều hành viên xác nhận vi phạm (`CONFIRMED`) hoặc bác khiếu nại (`DISMISSED`). |
| **2** | **Cơ chế hồi phục (Recovery)** | • Thưởng lái xe an toàn: **`+1.0` đến `+3.0 điểm`**<br>• Hoàn điểm cảnh báo sai: **`+2.0` hoặc `+5.0 điểm`** | Điểm sau hồi phục tối đa đạt trần **`100.0 điểm`** (không vượt quá 100). |
| **3** | **Sổ cái biến động điểm (Ledger)** | Bảng dữ liệu `driver_score_ledgers` lưu vết audit trail minh bạch từng lần cộng/trừ điểm. | Ghi nhận: `delta_score`, `balance_after`, `reason`, `source`, `case_id`, `created_at`. |
| **4** | **Chế tài & Xếp hạng 5 bậc** | Dựa trên điểm hiện tại để phân loại Hạng **`S` / `A` / `B` / `C` / `D`** và khuyến nghị hành vi. | Trả kèm mã chế tài `sanction_code` và hướng dẫn `action_required`. |

---

## 2. BẢNG XẾP HẠNG & KHUNG CHẾ TÀI VẬN HÀNH

| Thang điểm | Phân hạng (`tier`) | Tên hạng (`tier_label`) | Mã chế tài (`sanction_code`) | Hành động vận hành (`action_required`) |
| :---: | :---: | :---: | :---: | :--- |
| **90 – 100** | **`S`** | **Xuất sắc** | `NORMAL` | Đạt chuẩn an toàn cao nhất, đủ điều kiện hưởng thưởng KPI an toàn đội xe. |
| **80 – 89** | **`A`** | **Tốt** | `NORMAL` | Vận hành đạt yêu cầu an toàn, duy trì vận hành bình thường. |
| **65 – 79** | **`B`** | **Cần chú ý** | `CAUTION` | Gửi cảnh báo nhắc nhở trên app buồng lái, nâng cao tập trung khi lái xe. |
| **50 – 64** | **`C`** | **Nguy cơ cao** | `INTENSIVE_MONITORING` | Kích hoạt giám sát telemetry ngầm tần suất cao, điều hành viên theo dõi sát. |
| **< 50** | **`D`** | **Báo động đỏ** | `SUSPENSION_RECOMMENDED` | Khuyến nghị tạm dừng phân ca xe, yêu cầu tài xế đào tạo lại an toàn. |

---

## 3. DANH SÁCH ENDPOINTS BACKEND MỚI

### 3.1. API Lấy thông tin cá nhân tài xế (`GET /api/v1/auth/me`)
* **Quyền hạn:** `DRIVER` (hoặc bất kỳ role nào đã đăng nhập).
* **Mở rộng trả về trong `driver_profile`:**
```json
{
  "success": true,
  "status_code": 200,
  "message": "Lấy thông tin người dùng thành công",
  "data": {
    "id": "usr_driver_1042",
    "username": "driver1",
    "role": "DRIVER",
    "driver_profile": {
      "driver_license_no": "GPLX-29-881923",
      "assigned_vehicle_id": "veh_vinfast_vf8_02",
      "vehicle_plate": "29A-888.99",
      "safety_score": 94.0,
      "shift_status": "ON_DUTY",
      "tier": "S",
      "tier_label": "Xuất sắc",
      "sanction_code": "NORMAL",
      "sanction_name": "Đạt chuẩn xuất sắc",
      "action_required": "Đạt chuẩn an toàn cao nhất, đủ điều kiện hưởng thưởng an toàn đội xe."
    }
  }
}
```

---

### 3.2. API Sổ cái biến động điểm cho Tài xế (`GET /api/v1/drivers/me/score-ledger`)
* **Quyền hạn:** Chỉ tài xế đang đăng nhập (`role == DRIVER`).
* **Query Params:** `limit=50&offset=0`
* **Response:**
```json
{
  "success": true,
  "status_code": 200,
  "message": "Lấy sổ cái điểm an toàn thành công",
  "data": {
    "driver_id": "usr_driver_1042",
    "current_safety_score": 92.0,
    "tier": "S",
    "tier_label": "Xuất sắc",
    "sanction_code": "NORMAL",
    "sanction_name": "Đạt chuẩn xuất sắc",
    "action_required": "Đạt chuẩn an toàn cao nhất, đủ điều kiện hưởng thưởng an toàn đội xe.",
    "total": 3,
    "items": [
      {
        "id": "dsl_20261016035012_a1b2c3",
        "driver_id": "usr_driver_1042",
        "delta_score": -2.0,
        "balance_after": 92.0,
        "reason": "Xác nhận vi phạm an toàn (Cảnh báo) - Ca case_hitl_20261016_01",
        "source": "HITL_CONFIRMED",
        "severity": "WARNING",
        "case_id": "case_hitl_20261016_01",
        "created_at": "2026-10-16T03:50:12Z"
      },
      {
        "id": "dsl_20261015142010_e4f5a6",
        "driver_id": "usr_driver_1042",
        "delta_score": 1.0,
        "balance_after": 94.0,
        "reason": "Hồi phục: Lái xe an toàn ca làm việc",
        "source": "RECOVERY_SAFE_DRIVE",
        "severity": null,
        "case_id": null,
        "created_at": "2026-10-15T14:20:10Z"
      }
    ]
  }
}
```

---

### 3.3. API Sổ cái tài xế cho Dashboard (`GET /api/v1/drivers/{driver_id}/score-ledger`)
* **Quyền hạn:** `ADMIN`, `SAFETY_MANAGER`, `DISPATCHER`.
* **Query Params:** `limit=50&offset=0`
* **Response:** Tương tự như endpoint 3.2, cho phép quản lý xem lịch sử của bất kỳ tài xế nào.

---

### 3.4. API Tra cứu Xếp hạng & Chế tài (`GET /api/v1/drivers/{driver_id}/safety-tier`)
* **Quyền hạn:** Tài xế tự xem thông tin của mình, Quản lý/Admin xem được tất cả.
* **Response:**
```json
{
  "success": true,
  "status_code": 200,
  "message": "Tra cứu phân hạng an toàn thành công",
  "data": {
    "driver_id": "usr_driver_1042",
    "safety_score": 92.0,
    "tier": "S",
    "tier_label": "Xuất sắc",
    "sanction_code": "NORMAL",
    "sanction_name": "Đạt chuẩn xuất sắc",
    "action_required": "Đạt chuẩn an toàn cao nhất, đủ điều kiện hưởng thưởng an toàn đội xe."
  }
}
```

---

### 3.5. API Quản lý kích hoạt Hồi phục điểm (`POST /api/v1/drivers/{driver_id}/recover-score`)
* **Quyền hạn:** `ADMIN`, `SAFETY_MANAGER`.
* **Request Body:**
```json
{
  "points": 2.0,
  "reason": "Thưởng lái xe an toàn tuần không vi phạm"
}
```
* **Response:**
```json
{
  "success": true,
  "status_code": 200,
  "message": "Hồi phục thành công +2.0 điểm an toàn",
  "data": {
    "driver_id": "usr_driver_1042",
    "recovered_points": 2.0,
    "new_safety_score": 94.0,
    "tier": "S",
    "tier_label": "Xuất sắc",
    "ledger_id": "dsl_20261016035240_f8d91a"
  }
}
```

---

## 4. HƯỚNG DẪN TÍCH HỢP CHO FRONTEND & MOBILE APP

### 4.1. Hướng dẫn dành cho Frontend (Web Dashboard)
1. **Bảng Quản lý Tài xế (`DriversManagementPanel.jsx`):**
   * Hiển thị thêm cột hoặc badge **Xếp hạng an toàn** (Hạng S / A / B / C / D với màu tương ứng: S xanh lá đậm, A xanh lam, B vàng cam, C cam đỏ, D đỏ).
   * Bổ sung nút bấm `"Xem sổ cái"` trên từng hàng tài xế để mở Modal hiển thị danh sách biến động điểm (`GET /api/v1/drivers/{driver_id}/score-ledger`).
2. **Nút Thưởng/Hồi phục điểm (`recover-score`):**
   * Trong modal chi tiết tài xế, quản lý có thể nhập số điểm muốn cộng thưởng (+1 đến +5đ) và bấm `"Cộng điểm an toàn"`.

### 4.2. Hướng dẫn dành cho Mobile App (Android)
1. **Màn hình Profile/Dashboard của Bác tài:**
   * Sau khi đăng nhập và gọi `GET /api/v1/auth/me`, đọc các trường:
     * `driver_profile.tier` (ví dụ: `"S"`)
     * `driver_profile.tier_label` (ví dụ: `"Xuất sắc"`)
     * `driver_profile.action_required` (thông điệp chế tài/nhắc nhở).
   * Hiển thị Huy hiệu Hạng bên cạnh điểm an toàn (VD: `An toàn: 94đ • Hạng S`).
2. **Màn hình "Lịch sử Điểm An Toàn" (Safety Score Ledger):**
   * Tạo một mục trong Menu: `"Lịch sử điểm an toàn"`.
   * Gọi `GET /api/v1/drivers/me/score-ledger` để hiển thị danh sách dạng Timeline:
     * Dòng màu xanh: `+2.0 điểm` (Lý do: Lái xe an toàn).
     * Dòng màu đỏ: `-5.0 điểm` (Lý do: Vi phạm an toàn mức Nguy cấp).

---

## 5. NẾU FE VÀ MOBILE APP CHƯA SỬA NGAY, CÓ ẢNH HƯỞNG GÌ KHÔNG?

> [!NOTE]
> **KẾT LUẬN: HOÀN TOÀN KHÔNG ẢNH HƯỞNG (TƯƠNG THÍCH NGƯỢC 100% - ZERO BREAKING CHANGES).**

Hệ thống đã được thiết kế tuân thủ nghiêm ngặt nguyên tắc **Backward Compatibility**:

1. **Về Schema dữ liệu cũ:**
   * Trường `safety_score` và `current_safety_score` trong `User`, `DriverProfile`, `DriverShiftProfile`, `HitlCase` **giữ nguyên 100% kiểu dữ liệu (float/int) và tên trường**.
   * Các màn hình hiện tại của FE và Mobile App đang đọc `safety_score` vẫn hiển thị số điểm bình thường (ví dụ: `94 / 100`), không bị null hay sai format.
2. **Về các trường mới:**
   * Các trường mới như `tier`, `sanction_code`, `action_required` là **trường mở rộng bổ sung (additive fields)** có giá trị mặc định. Nếu FE hoặc App chưa viết code đọc các trường này, parser JSON (như Gson trên Android hoặc `res.data` trên React) sẽ tự động bỏ qua mà không gây crash hay lỗi.
3. **Về luồng thẩm định HITL:**
   * Nút bấm xác nhận vi phạm trên FE (`confirm_case`) vẫn hoạt động như cũ. Điểm trừ được tính tự động ngầm dưới Backend (WARNING trừ 2, CRITICAL trừ 5) và tự động ghi log vào Sổ cái `driver_score_ledgers`.
   * Luồng giải tỏa cảnh báo sai (`reject_false_alarm` / `UPHELD`) vẫn trả về message cũ và tự động hoàn lại điểm ngầm cho tài xế.
4. **Kế hoạch triển khai:**
   * **Giai đoạn hiện tại:** Backend đã hoàn thiện, tự động tính toán, lưu trữ sổ cái và bảo toàn dữ liệu an toàn.
   * **Giai đoạn tiếp theo (khi có thời gian):** Đội ngũ FE và App chỉ cần tạo UI hiển thị sổ cái và huy hiệu xếp hạng mà không cần thay đổi bất kỳ logic cốt lõi nào.
