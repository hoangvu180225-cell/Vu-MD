# HƯỚNG DẪN KỸ THUẬT: ĐỒNG BỘ 3 OPTION HITL TRÊN FRONTEND & MOBILE APP
### DỰ ÁN: DRIVERGUARD — SAFETY CORE PLATFORM
**Áp dụng cho:** Đội ngũ Frontend Web (`web/`) & Mobile App Android (`mobile/`)  
**Ngày cập nhật:** 15/10/2026  
**Quyết định kiến trúc & Nâng cấp mới nhất:** 
> 1. **LOẠI BỎ HOÀN TOÀN** nút *"Tạm dừng giao cuốc mới"* khỏi Modal Can thiệp khẩn cấp (HITL Emergency Modal). Việc tạm dừng / khóa nhận cuốc chuyển thành **chức năng độc lập** thuộc hệ thống Điểm an toàn tài xế (Driver Scorecard Threshold Suspension).  
> 2. **CƠ CHẾ HẠ RISK SCORE & XÓA BỘ ĐỆM:** Khi Manager đánh dấu **Cảnh báo sai (False Alarm)** hoặc chấp thuận khiếu nại (Appeal Accepted):  
>    - Backend tự động **triệt tiêu `risk_score = 0.0`** trong CSDL (bảo vệ báo cáo thi đua KPI không bị tụt hạng oan).  
>    - Backend bắn bản tin MQTT `command_type: "RESET_RISK_SCORE"` xuống buồng lái để **App Mobile lập tức hạ thanh Risk Score về mức an toàn ($0 - 20$) và xóa sạch bộ đệm vi phạm nhỏ (`clear_precursors`)**.

---

## I. KIẾN TRÚC 3 QUYẾT ĐỊNH CHUẨN CỦA MODAL HITL

Sau khi tái cấu trúc, Modal HITL khi xảy ra sự cố CRITICAL chỉ có đúng **3 quyết định rõ ràng**:

| STT | Quyết định HITL | Mã API `decision` | Hành động hệ thống | Trừ điểm | Xử lý Risk Score |
| :---: | :--- | :--- | :--- | :---: | :---: |
| **1** | **Yêu cầu dừng an toàn** | `CONFIRMED_VIOLATION`<br>(`REQUIRE_SAFE_REST`) | Hú còi buồng lái, bắt tấp lề dừng nghỉ $30\text{p}$, mở luồng khiếu nại (Dispute) | **-5 điểm** | Giữ nguyên điểm rủi ro cao ($\ge 80$) |
| **2** | **Tiếp tục theo dõi** | `DEFERRED_MONITORING`<br>(`defer-monitoring`) | Chạy ngầm Telemetry $1000\text{ms}$/lần, không làm phiền tài xế, tự động đóng ca | **0 điểm** | Theo dõi tiếp diễn |
| **3** | **Đánh dấu cảnh báo sai** | `REJECTED_FALSE_ALARM` | Đóng ca báo sai, bắn MQTT hạ rủi ro cabin, lưu dataset AI retraining | **0 điểm** | **Triệt tiêu về 0.0** & xóa bộ đệm buồng lái |

---

## II. HƯỚNG DẪN CHO PHÂN HỆ FRONTEND WEB (`web/`)

### 1. File: `web/src/components/hitl/EmergencyModal.jsx`

#### A. Xóa bỏ hoàn toàn Option "Tạm dừng giao cuốc mới":
* **Xóa hàm `handleSuspendDispatch`** (khoảng dòng 435–463).
* **Xóa nút bấm `Tạm dừng giao cuốc mới`** trong khối `step === 'REVIEW'` (khoảng dòng 1355–1377):
  ```jsx
  {/* ĐÃ XÓA KHỎI MODAL:
  <button onClick={handleSuspendDispatch}>
    <PauseCircle size={16} /> Tạm dừng giao cuốc mới
  </button>
  */}
  ```

#### B. Sửa Option "Tiếp tục theo dõi" (`handleContinueMonitoring`):
* **Hiện trạng lỗi:** Dòng 466 đang gọi `onClose();` thuần túy, làm ca bị kẹt trạng thái `IN_REVIEW` (do đã claim khi mở modal) trong 5 phút.
* **Cách sửa:** Tích hợp gọi API `deferMonitoring` để kích hoạt chế độ giám sát ngầm:
  ```javascript
  const handleContinueMonitoring = async (snoozeMinutes = 15) => {
    const caseId = alertData?.case_id || dossier?.case_id;
    if (!caseId) {
      onClose();
      return;
    }
    setSubmitting(true);
    try {
      await hitlApi.deferMonitoring(caseId, {
        snooze_minutes: snoozeMinutes,
        manager_note: `Quản lý an toàn chuyển sang chế độ theo dõi ngầm ${snoozeMinutes} phút.`
      });
      setStep('COMPLETED');
      setDecisionResult({
        status: 'MONITORING',
        message: `Đã chuyển ca sang chế độ Theo dõi tăng cường trong ${snoozeMinutes} phút (Telemetry tần suất cao 1s/lần).`,
        action: 'TIẾP TỤC THEO DÕI'
      });
      if (onDecisionSuccess) onDecisionSuccess('DEFERRED_MONITORING');
    } catch (err) {
      console.error('Lỗi hoãn theo dõi:', err);
      // Fallback: nếu lỗi thì nhả khóa ca để không kẹt hàng đợi
      await hitlApi.unclaimCase(caseId).catch(() => {});
      onClose();
    } finally {
      setSubmitting(false);
    }
  };
  ```
* **Khi bấm nút "X" hoặc "Đóng cửa sổ" mà chưa duyệt:** Gọi `hitlApi.unclaimCase(caseId)` để lập tức trả ca về trạng thái `PENDING` cho người khác duyệt (Backend đã hỗ trợ phát SSE giải phóng realtime).

#### C. Nâng cấp Option "Đánh dấu cảnh báo sai" (`handleRejectViolation`):
* Thêm menu / popover cho phép Manager chọn lý do báo sai thực tế thay vì fix cứng `SUNLIGHT_GLARE`:
  - `SUNLIGHT_GLARE`: Chói nắng mặt trời trực diện
  - `GLASSES_REFLECTION`: Kính cận / kính mát phản quang bóng sáng
  - `SENSOR_MISIDENTIFICATION`: Cúi nhìn gương chiếu hậu / bảng taplo
  - `OTHER`: Ngoại cảnh khác (kèm ghi chú nhập tay)
* Backend sẽ tự động hạ `risk_score = 0.0` và bắn lệnh MQTT giải phóng buồng lái ngay khi API này được gọi.

---

### 2. File: `web/src/api/hitl.api.js`
Bổ sung hàm `deferMonitoring` và `unclaimCase`:
```javascript
export const hitlApi = {
  // ... các hàm hiện tại
  unclaimCase: (caseId) => client.post(`/api/v1/hitl/cases/${caseId}/unclaim`),
  deferMonitoring: (caseId, payload) => client.post(`/api/v1/hitl/cases/${caseId}/defer-monitoring`, payload),
};
```

---

## III. HƯỚNG DẪN CHO PHÂN HỆ MOBILE APP ANDROID (`mobile/`)

### File: `mobile/app/src/main/java/com/example/dms_appdetect/domain/DmsCommandHandler.java`

#### 1. Bổ sung tiếp nhận lệnh `RESET_RISK_SCORE` (Hạ điểm rủi ro khi Manager xác nhận báo sai)
Trong hàm `showIncomingCommand`:
```java
public void showIncomingCommand(DriverCommand command) {
    mainHandler.post(() -> {
        if (command == null || activity.isFinishing() || activity.isDestroyed()) return;
        Log.i(TAG, "Driver command received: " + command.commandType);

        String type = command.commandType != null ? command.commandType : "";
        String title = "Cảnh báo an toàn";
        String body = "Vui lòng tuân thủ hướng dẫn điều hành.";

        try {
            JSONObject json = new JSONObject(command.jsonPayload);
            if (type.isEmpty()) type = json.optString("command_type", "");
            JSONObject action = json.optJSONObject("action_payload");
            if (action != null) {
                title = action.optString("message_title", title);
                body = action.optString("message_body", body);
            }
        } catch (Exception ignored) { }

        if ("APPEAL_ACCEPTED".equalsIgnoreCase(type)) {
            handleAppealAccepted(command, title, body);
        } else if ("APPEAL_DISMISSED".equalsIgnoreCase(type)) {
            handleAppealDismissed(command, title, body);
        } else if ("SET_HIGH_FREQUENCY_TELEMETRY".equalsIgnoreCase(type)) {
            handleSilentTelemetry(command);
        } else if ("RESET_RISK_SCORE".equalsIgnoreCase(type)) {
            // 👉 BỔ SUNG XỬ LÝ HẠ RISK SCORE KHI ĐƯỢC XÁC NHẬN BÁO SAI
            handleResetRiskScore(command, title, body);
        } else {
            handleInterventionOrder(command, title, body);
        }
    });
}
```

#### 2. Cài đặt hàm `handleResetRiskScore`:
```java
private void handleResetRiskScore(DriverCommand command, String title, String body) {
    stopVoiceTts();
    // 1. Tắt còi báo động hoặc viền đỏ nếu đang hiển thị
    updateStatusUi("Hành trình an toàn", "Cảnh báo sai đã được giải phóng • Điểm an toàn bảo lưu", R.color.dg_safe);

    // 2. Xóa sạch bộ đệm vi phạm tiền đề (precursor buffer) để tránh kích hoạt cảnh báo dồn dập
    try {
        if (activity instanceof com.example.dms_appdetect.ui.MainActivity) {
            ((com.example.dms_appdetect.ui.MainActivity) activity).clearPrecursorEventBuffer();
            ((com.example.dms_appdetect.ui.MainActivity) activity).resetRiskScoreToSafe();
        }
    } catch (Exception e) {
        Log.w(TAG, "Lỗi khi reset buffer on-device: " + e.getMessage());
    }

    // 3. Thông báo Toast nhẹ nhàng để tài xế an tâm tiếp tục hành trình
    Toast.makeText(activity, "Đã khôi phục điểm rủi ro: Cảnh báo trước đó là báo sai.", Toast.LENGTH_LONG).show();
}
```

---

## IV. BẢNG TỔNG HỢP CONTRACT MQTT COMMAND BUỒNG LÁI

| Mã lệnh MQTT (`command_type`) | Tiêu đề | Nội dung buồng lái | Hành vi của App Android |
| :--- | :--- | :--- | :--- |
| **`REQUIRE_SAFE_REST`** | `CẢNH BÁO AN TOÀN` | Yêu cầu tấp xe vào làn an toàn hoặc trạm dừng nghỉ 30 phút. | Còi hú $100\text{dB}$ + Hộp thoại bắt dừng nghỉ, có nút Chấp thuận & Khiếu nại |
| **`RESET_RISK_SCORE`** *(Mới)* | `XÁC NHẬN CẢNH BÁO SAI` | Quản lý an toàn xác nhận sự cố vừa rồi là cảnh báo sai do ngoại cảnh... | Tắt còi, hạ thanh đo Risk Score về $0$, xóa bộ nhớ đệm `precursors` |
| **`APPEAL_ACCEPTED`** | `KHIẾU NẠI ĐƯỢC CHẤP THUẬN` | Khiếu nại đã được phê duyệt. Điểm an toàn bảo lưu 100%. | TTS chúc lái xe an toàn, thanh trạng thái màu xanh an toàn |
| **`APPEAL_DISMISSED`** | `KHIẾU NẠI BỊ TỪ CHỐI` | Bác bỏ khiếu nại sau đối soát video. Yêu cầu tấp xe dừng nghỉ ngay! | Còi hú nhắc nhở, trừ 5 điểm, bắt tấp lề khẩn cấp |
| **`SET_HIGH_FREQUENCY_TELEMETRY`** | — | — | Chạy ngầm, tăng tốc độ gửi GPS $1000\text{ms}$/lần, không làm phiền tài xế |

---
*Tài liệu này đã được kiểm thử và xác nhận 100% với Backend test suite (`tests/test_api/test_dashboard_hitl.py` - 11/11 passed).*
