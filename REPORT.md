# LAB 3 – Anomaly Detection & Event Intelligence
## Báo cáo phân tích kết quả

**Học phần:** Triển khai, phát triển ứng dụng AI và IoT  
**Dataset:** Numenta Anomaly Benchmark – `ambient_temperature_system_failure.csv`  
**Mô hình chính:** Isolation Forest (`iforest_v2`)  
**Mô hình phụ:** MLPRegressor Autoencoder (demo)

---

## 1. Kết quả model

| Chỉ số | Isolation Forest | Autoencoder demo |
|--------|-----------------|-----------------|
| Precision | **0.2401** | 0.1645 |
| Recall | **0.5675** | 0.1763 |
| F1-score | **0.3374** | 0.1702 |
| Threshold | 0.6091 (từ train) | MSE 0.0466 (từ train) |

**Confusion Matrix – Isolation Forest (tập test, 2544 dòng):**

|  | Predicted Normal | Predicted Anomaly |
|--|--|--|
| **Actual Normal** | TN = 1529 | FP = 652 |
| **Actual Anomaly** | FN = 157 | TP = 206 |

- Train: 4723 dòng (2013-07-04 → 2014-02-02)  
- Test: 2544 dòng (2014-02-02 → 2014-05-28)  
- Anomaly trong tập test: 363 điểm (thuộc 2 window NAB)

---

## 2. Câu hỏi phân tích

### Q1. Lab 3 đang phát hiện bất thường hay dự báo tương lai? Giải thích.

**Phát hiện bất thường (anomaly detection), không phải dự báo.**  
Model nhận đầu vào là chuỗi telemetry hiện tại và lịch sử gần, rồi tính `anomaly_score` cho từng điểm dữ liệu đã có. Output là `is_anomaly`, `severity`, `decision` – tất cả đều mô tả trạng thái *hiện tại hoặc quá khứ gần*.  
Forecasting (Lab 4) mới là bài toán dự báo giá trị tương lai chưa xảy ra.

---

### Q2. Vì sao dữ liệu IoT cần chia train/test theo thời gian?

Dữ liệu IoT có tính **tuần tự thời gian** (temporal ordering). Nếu random split, model có thể học được thông tin từ tương lai để dự đoán quá khứ – đây là **data leakage**.  
Trong thực tế triển khai, model chỉ được phép biết dữ liệu quá khứ. Chia theo thời gian (`past → train`, `future → test`) mô phỏng đúng điều kiện vận hành thật.  
Ví dụ trong project: train dùng dữ liệu trước 2014-02-02, test dùng dữ liệu sau ngày đó.

---

### Q3. Precision thấp gây rủi ro gì cho người vận hành?

Precision = 0.24 nghĩa là **76% cảnh báo là false alert** (FP = 652 trong tổng số 858 cảnh báo).  
Rủi ro:
- Người vận hành nhận quá nhiều cảnh báo sai → **mất niềm tin vào hệ thống**.
- Dẫn đến hiện tượng **alert fatigue**: người vận hành bắt đầu bỏ qua cảnh báo hoặc tắt alert.
- Khi anomaly thật xảy ra, cảnh báo có thể bị bỏ qua vì đã quen với cảnh báo sai.

---

### Q4. Recall thấp gây rủi ro gì cho hệ thống?

Recall = 0.57 nghĩa là **43% anomaly thật bị bỏ sót** (FN = 157).  
Rủi ro:
- Lỗi hệ thống thật không được phát hiện → hỏng thiết bị, mất dữ liệu, nguy hiểm an toàn.
- Với môi trường kho lạnh hoặc phòng máy chủ, bỏ sót một lỗi nhiệt độ có thể gây hư hại nghiêm trọng.
- Recall thấp nguy hiểm hơn Precision thấp trong các use-case có chi phí bỏ sót cao.

---

### Q5. MSE trong Autoencoder có ý nghĩa gì?

Autoencoder học tái tạo (reconstruct) các **window dữ liệu bình thường**. Khi nhận window bất thường, model không tái tạo tốt → **reconstruction MSE cao**.  
Ý nghĩa:
- MSE thấp: window giống với pattern bình thường đã học → khả năng không bất thường.
- MSE cao: window khác với pattern bình thường → có thể bất thường.
- Threshold MSE = 0.0466 lấy từ phân bố MSE trên tập train (quantile 96%), không lấy từ test.

---

### Q6. anomaly_score khác decision như thế nào?

| | anomaly_score | decision |
|--|--|--|
| **Bản chất** | Số thực [0,1] từ model | Chuỗi hành động nghiệp vụ |
| **Ví dụ** | 0.87 | `CREATE_ALERT_AND_REQUIRE_HUMAN_CHECK` |
| **Ai tạo ra** | Model (Isolation Forest) | Decision layer (business rule) |
| **Dùng để** | So sánh với threshold | Kích hoạt workflow, ghi log, gửi alert |

`anomaly_score` chỉ nói "bao nhiêu bất thường", còn `decision` nói "cần làm gì tiếp theo". Một điểm có `anomaly_score = 0.65` có thể chỉ được ghi log (MEDIUM), trong khi `anomaly_score = 0.92` mới tạo alert và yêu cầu con người kiểm tra (HIGH).

---

### Q7. Vì sao anomaly cao chưa chắc được phép điều khiển thiết bị?

**An toàn hệ thống (safety)** là nguyên tắc tối thượng trong AIoT công nghiệp.  
- Model có thể **sai** (FP = 652 trên tập test). Nếu tự động điều khiển khi anomaly cao → có thể tắt thiết bị đang hoạt động bình thường.
- Điều khiển tự động dựa trên anomaly cần thêm: xác nhận từ nhiều nguồn, cooldown period, safety interlock, xác nhận người vận hành.
- API trong Lab 3 luôn trả `safety_note: "Không tự động điều khiển thiết bị khi anomaly cao; cần xác nhận hoặc rule an toàn."` để nhắc nhở điều này.

---

### Q8. Nếu triển khai thật, cần thêm safety rule nào?

1. **Cooldown / debounce**: chỉ tạo alert nếu anomaly kéo dài ≥ N điểm liên tiếp (tránh spike đơn lẻ).
2. **Ngưỡng kép (dual threshold)**: cảnh báo khi score > 0.60, chỉ tự động hành động khi score > 0.90 và kéo dài.
3. **Cross-sensor validation**: anomaly chỉ được xác nhận khi ≥ 2 sensor độc lập cùng báo bất thường.
4. **Human-in-the-loop**: mọi quyết định điều khiển thiết bị cần xác nhận người vận hành trong vòng X phút.
5. **Audit log**: ghi lại toàn bộ anomaly event, decision, và ai/cái gì đã phê duyệt.
6. **Fallback mode**: nếu model không trả về kết quả trong timeout, hệ thống chuyển về chế độ an toàn.

---

### Q9. Nếu cảnh báo quá nhiều, nhóm sẽ giảm alert fatigue bằng cách nào?

1. **Alert aggregation**: gom các anomaly liên tiếp trong cùng 1 window thành 1 event thay vì N cảnh báo.
2. **Cooldown period**: sau khi tạo alert, không tạo alert mới trong vòng T phút cho cùng device.
3. **Severity escalation**: chỉ thông báo ngay lập tức khi severity = HIGH; MEDIUM ghi log và báo cáo định kỳ.
4. **Adaptive threshold**: tự động tăng threshold khi FP rate cao, giảm khi FN rate cao.
5. **Tăng Precision**: điều chỉnh contamination parameter, thêm feature tốt hơn, hoặc dùng ensemble model.
6. **Dashboard summary**: thay vì push từng alert, hiển thị trend và summary theo giờ/ngày.

---

### Q10. Nhóm sẽ hiển thị gì trên dashboard ngoài giá trị cảm biến?

1. **Anomaly score timeline**: biểu đồ `anomaly_score` theo thời gian với đường threshold.
2. **Severity heatmap**: màu sắc theo severity (xanh = normal, vàng = MEDIUM, đỏ = HIGH) trên timeline.
3. **Event log table**: bảng các anomaly event gần nhất với `event_type`, `decision`, `explanation`.
4. **Model health indicator**: Precision/Recall hiện tại, số FP/FN trong 24h.
5. **Sensor status**: trạng thái từng sensor (NORMAL / STUCK_CANDIDATE / ANOMALY).
6. **Rolling stats**: `rolling_mean`, `rolling_std`, `zscore` theo thời gian thực.
7. **Alert counter**: tổng số alert theo severity trong 1h / 24h / 7 ngày.

---

## 3. Tổng kết pipeline

```text
NAB Dataset (7267 dòng)
  └─► add_time_features()
        rolling_mean_12, rolling_std_12, rolling_mean_36
        delta_1, delta_3, zscore_rolling, is_stuck_candidate
  └─► time_split (65/35)
        Train: 4723 dòng  ─► Isolation Forest (fit trên train_normal)
        Test:  2544 dòng  ─► anomaly_score, is_anomaly
  └─► build_events()
        severity (LOW/MEDIUM/HIGH)
        event_type (TEMPERATURE_ANOMALY, SENSOR_STUCK, SUDDEN_JUMP, ...)
        decision (LOG / CREATE_WARNING / CREATE_ALERT_AND_REQUIRE_HUMAN_CHECK)
  └─► anomaly_event_log.csv
  └─► FastAPI /detect-anomaly
        Input:  history[] TelemetryPoint
        Output: model_output + event + api_check
```

---

## 4. Checklist nộp

- [x] Notebook `01_anomaly_detection_event_intelligence.ipynb` đã chạy đủ cell
- [x] `outputs/iforest_metrics.json` – Precision=0.2401, Recall=0.5675, F1=0.3374
- [x] `outputs/anomaly_event_log.csv` – 858 event với severity và decision
- [x] `outputs/api_test_result.json` – PASS
- [x] `outputs/autoencoder_metrics.json` – F1=0.1702
- [x] `models/anomaly_model_bundle_iforest_v2.joblib`
- [x] `figures/anomaly_detection_result.png`
- [x] `figures/anomaly_score_over_time.png`
- [x] `figures/api_health_response.json` – `model_loaded: true`
- [x] `figures/api_swagger_ui.png` – Swagger UI screenshot
- [x] API `/health` → `{"status": "ok", "model_loaded": true}`
- [x] API `/detect-anomaly` → JSON có `anomaly_score`, `threshold_used`, `severity`, `decision`
