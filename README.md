# LAB 3 – Anomaly Detection & Event Intelligence cho chuỗi thời gian IoT

> **Học phần:** Triển khai, phát triển ứng dụng AI và IoT – Buổi 3  
> **Dataset:** Numenta Anomaly Benchmark (NAB) – `ambient_temperature_system_failure.csv`  
> **Model:** Isolation Forest (`iforest_v2`) + MLPRegressor Autoencoder (demo)  
> **API:** FastAPI `/detect-anomaly` → `anomaly_score`, `severity`, `decision`

---

## Kết quả chạy thực tế

| Chỉ số | Isolation Forest | Autoencoder demo |
|--------|:---:|:---:|
| Precision | **0.2401** | 0.1645 |
| Recall | **0.5675** | 0.1763 |
| F1-score | **0.3374** | 0.1702 |
| Threshold | 0.6091 | MSE 0.0466 |
| Train rows | 4 723 | – |
| Test rows | 2 544 | 2 521 |

**Confusion Matrix – Isolation Forest (test set):**

```
              Predicted Normal  Predicted Anomaly
Actual Normal       1529              652  (FP)
Actual Anomaly       157  (FN)        206  (TP)
```

**API `/detect-anomaly` – mẫu response:**

```json
{
  "model_output": {
    "anomaly_score": 0.87,
    "threshold_used": 0.6091,
    "is_anomaly": true,
    "model_version": "iforest_v2"
  },
  "event": {
    "event_type": "TEMPERATURE_PATTERN_DEVIATION",
    "severity": "HIGH",
    "decision": "CREATE_ALERT_AND_REQUIRE_HUMAN_CHECK",
    "explanation": "value deviates strongly from recent pattern",
    "safety_note": "Không tự động điều khiển thiết bị khi anomaly cao; cần xác nhận hoặc rule an toàn."
  }
}
```

---

## 1. Lab này khác Lab 2 ở đâu?

- **Lab 2 = Data Pipeline**: làm cho telemetry đủ sạch, đủ đúng schema, đủ feature để AI dùng được.
- **Lab 3 = Event Pipeline**: dùng dữ liệu đã sẵn sàng để phát hiện bất thường và biến kết quả model thành `event_type`, `severity`, `decision`, `anomaly_event_log.csv` và API `/detect-anomaly`.

Luồng chính của project:

```text
Clean telemetry
→ feature window
→ anomaly model
→ anomaly_score
→ threshold
→ event_type
→ severity
→ alert / safety decision
→ anomaly_event_log.csv
→ API /detect-anomaly
```

## 2. Dataset

Bài mẫu dùng dataset public từ Numenta Anomaly Benchmark (NAB):

- Dataset: `ambient_temperature_system_failure.csv`
- Nguồn: https://github.com/numenta/NAB/tree/master/data/realKnownCause
- File raw: https://raw.githubusercontent.com/numenta/NAB/master/data/realKnownCause/ambient_temperature_system_failure.csv
- Label windows: https://raw.githubusercontent.com/numenta/NAB/master/labels/combined_windows.json

Nếu máy không có Internet, project dùng file sample kèm theo trong `data/`.

## 3. Cấu trúc project

```text
lab3_aiot_anomaly_event_intelligence_v3/
├─ data/                         # dữ liệu public hoặc sample fallback
├─ notebooks/                    # notebook hướng dẫn từng bước
├─ src/
│  ├─ download_data.py            # tải dữ liệu public NAB
│  ├─ train_anomaly.py            # train/test Isolation Forest + autoencoder demo
│  ├─ app.py                      # FastAPI deploy model
│  ├─ test_api.py                 # test API sau khi deploy bằng uvicorn
│  ├─ test_api_local.py           # test API logic không cần mở port
│  ├─ plot_results.py             # vẽ biểu đồ kết quả
│  └─ utils.py                    # hàm dùng chung
├─ models/                       # model .joblib sau khi train
├─ outputs/                      # metrics, prediction, anomaly_event_log, api_test_result
├─ figures/                      # biểu đồ kết quả
├─ diagrams/                     # hình minh họa trong tài liệu
└─ requirements.txt
```

## 4. Cài môi trường

```bash
cd lab3_aiot_anomaly_event_intelligence_v3
python -m venv .venv
```

Windows:

```bash
.venv\Scripts\activate
```

macOS/Linux:

```bash
source .venv/bin/activate
```

Cài thư viện:

```bash
pip install -r requirements.txt
```

## 5. Chạy bài mẫu bằng script

Tải dataset public hoặc dùng fallback sample:

```bash
python src/download_data.py
```

Train, test, đánh giá model:

```bash
python src/train_anomaly.py
```

Vẽ biểu đồ:

```bash
python src/plot_results.py
```

Kết quả cần thấy:

```text
models/anomaly_model_bundle_iforest_v2.joblib
models/isolation_forest_iforest_v1.joblib
models/mlp_autoencoder_demo.joblib
outputs/iforest_metrics.json
outputs/autoencoder_metrics.json
outputs/iforest_test_predictions.csv
outputs/anomaly_event_log.csv
figures/anomaly_detection_result.png
figures/anomaly_score_over_time.png
```
<img width="600" height="492" alt="image" src="https://github.com/user-attachments/assets/945945a6-debb-4c03-8211-2029a15a6a40" />
<img width="456" height="219" alt="image" src="https://github.com/user-attachments/assets/ae027d7a-3cbe-4b58-8b43-c6b38ea85938" />
<img width="504" height="561" alt="image" src="https://github.com/user-attachments/assets/5cd8e26d-94ab-4d29-ab05-3a9820cf6692" />


## 6. Chạy notebook

```bash
jupyter notebook notebooks/01_anomaly_detection_event_intelligence.ipynb
```
<img width="703" height="393" alt="image" src="https://github.com/user-attachments/assets/ca321b5b-f984-4766-821a-703dd2c2dab4" />
<img width="832" height="548" alt="image" src="https://github.com/user-attachments/assets/0d74360e-76ae-48b6-9ac0-5af7275164b5" />
<img width="966" height="464" alt="image" src="https://github.com/user-attachments/assets/99284a47-fcef-4430-8e42-42387e1d7ebe" />
<img width="967" height="298" alt="image" src="https://github.com/user-attachments/assets/c59d3bec-05bb-45c0-9c87-df59010e48a4" />
<img width="972" height="509" alt="image" src="https://github.com/user-attachments/assets/c8382a2c-5b1f-48c5-9c06-951f694487d6" />
<img width="953" height="570" alt="image" src="https://github.com/user-attachments/assets/d49b0485-afc6-4280-bccb-00a72e40c5b6" />
<img width="968" height="442" alt="image" src="https://github.com/user-attachments/assets/c8391259-432e-4d63-9fb5-dd1a8fdf23e8" />
<img width="967" height="577" alt="image" src="https://github.com/user-attachments/assets/0cec7526-1625-432f-b73e-8237ba153656" />
<img width="964" height="588" alt="image" src="https://github.com/user-attachments/assets/71d111ba-ce54-46b5-adab-b35eb6d78ec4" />
<img width="956" height="563" alt="image" src="https://github.com/user-attachments/assets/e136fb3b-61e1-49cd-916d-3b459499540a" />

Chạy từng cell từ trên xuống. Sau mỗi phần, đọc kỹ mục **Cần quan sát gì** và **Cần phân tích gì**.

## 7. Deploy model bằng FastAPI

Sau khi train model xong:

```bash
uvicorn src.app:app --reload
```

Mở trình duyệt:

```text
http://127.0.0.1:8000/docs
```

Test API bằng script ở terminal khác:

```bash
python src/test_api.py
```

Nếu máy không mở được local port, test logic API bằng:

```bash
python src/test_api_local.py
```

## 8. Deploy model bằng FastAPI – cách đúng

> Chạy uvicorn **từ thư mục `src/`** để Python tìm được `utils.py`:

```bash
cd src
python -m uvicorn app:app --reload --port 8000
```

Mở Swagger UI: http://127.0.0.1:8000/docs  
Test từ terminal khác (quay về thư mục gốc):

```bash
cd ..
python src/test_api.py
```

Nếu máy không mở được local port:

```bash
python src/test_api_local.py
```

---
<img width="975" height="408" alt="image" src="https://github.com/user-attachments/assets/ea1e4d01-434e-47f2-88c4-1dd6d467c42e" />
<img width="996" height="375" alt="image" src="https://github.com/user-attachments/assets/067bf81c-e189-41bf-bf89-7aa609052012" />
<img width="976" height="293" alt="image" src="https://github.com/user-attachments/assets/3e63a822-c29f-4127-af15-f6000ca68e85" />
<img width="979" height="474" alt="image" src="https://github.com/user-attachments/assets/03ebac4f-cf4e-4de8-8538-76387f0a8254" />

## 9. Phân tích kết quả

### Tại sao chia train/test theo thời gian?
Dữ liệu IoT có tính tuần tự. Random split gây **data leakage** – model nhìn thấy tương lai khi train. Project chia: train = trước 2014-02-02, test = sau ngày đó.

### Precision thấp (0.24) có nghĩa gì?
76% cảnh báo là false alert (FP = 652). Người vận hành dễ bị **alert fatigue** và bỏ qua cảnh báo thật.

### Recall thấp (0.57) gây rủi ro gì?
43% anomaly thật bị bỏ sót (FN = 157). Với kho lạnh hoặc phòng máy, bỏ sót lỗi nhiệt độ có thể hỏng thiết bị.

### anomaly_score khác decision như thế nào?

| | `anomaly_score` | `decision` |
|--|--|--|
| Bản chất | Số thực [0, 1] từ model | Chuỗi hành động nghiệp vụ |
| Ví dụ | 0.87 | `CREATE_ALERT_AND_REQUIRE_HUMAN_CHECK` |
| Ai tạo | Isolation Forest | Decision rule layer |

### Vì sao anomaly cao không được tự động điều khiển thiết bị?
Model có thể sai (652 FP trong test). Điều khiển tự động cần thêm: cooldown, cross-sensor validation, xác nhận người vận hành, safety interlock.

### Giảm alert fatigue bằng cách nào?
- **Cooldown**: sau alert không tạo thêm trong T phút cho cùng device.
- **Aggregation**: gom anomaly liên tiếp thành 1 event.
- **Severity filter**: chỉ push ngay khi HIGH; MEDIUM ghi log báo cáo định kỳ.

---

## 10. Biểu đồ kết quả

| Biểu đồ | Mô tả |
|---------|-------|
| `figures/anomaly_detection_result.png` | Chuỗi thời gian + điểm anomaly được phát hiện |
| `figures/anomaly_score_over_time.png` | anomaly_score theo thời gian và đường threshold |

---

## 11. Checklist hoàn thành

- [x] Notebook chạy hết không lỗi (10 bước)
- [x] `outputs/iforest_metrics.json` – Precision=0.24, Recall=0.57, F1=0.34
- [x] `outputs/anomaly_event_log.csv` – 858 event với severity và decision
- [x] `outputs/api_test_result.json` – PASS
- [x] `outputs/autoencoder_metrics.json` – F1=0.17
- [x] `models/anomaly_model_bundle_iforest_v2.joblib`
- [x] `figures/anomaly_detection_result.png` + `anomaly_score_over_time.png`
- [x] API `/health` → `{"status": "ok", "model_loaded": true}`
- [x] API `/detect-anomaly` → JSON đủ `anomaly_score`, `threshold_used`, `severity`, `decision`
- [x] Giải thích được test model ≠ deploy model
- [x] Giải thích được anomaly_score ≠ decision cuối cùng

---

## Tài liệu tham khảo

- [Numenta Anomaly Benchmark (NAB)](https://github.com/numenta/NAB)
- [scikit-learn Isolation Forest](https://scikit-learn.org/stable/modules/generated/sklearn.ensemble.IsolationForest.html)
- [FastAPI documentation](https://fastapi.tiangolo.com/)
