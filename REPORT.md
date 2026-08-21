# BÁO CÁO THỰC HÀNH MLOPS - DAY 21
## Xây Dựng Pipeline CI/CD Tự Động Cho Hệ Thống AI

- **Họ và tên**: Chu Thành Dũng
- **Mã sinh viên / ID**: 2A202601405
- **Khóa học**: AI In Action - VinUni (Track 2 - K3)
- **Repository**: [https://github.com/chuthanhdung5-source/Track2-K3-Day21-2A202601405-chuthanhdung](https://github.com/chuthanhdung5-source/Track2-K3-Day21-2A202601405-chuthanhdung)
- **Cloud Provider**: Google Cloud Platform (GCP)
- **Cloud VM External IP**: `35.247.134.181:8000`

---

### 1. Kết Quả Thực Nghiệm Cục Bộ & Lựa Chọn Siêu Tham Số (Bước 1)
Quá trình huấn luyện sử dụng thuật toán **RandomForestClassifier** trên tập dữ liệu Wine Quality. Tôi đã thực hiện 4 lần thí nghiệm và ghi nhận toàn bộ thông số vào **MLflow** (`sqlite:///mlflow.db`):

- **Bộ siêu tham số được chọn**:
  - `n_estimators`: `300`
  - `max_depth`: `25`
  - `min_samples_split`: `2`
  - `random_state`: `42`
- **Kết quả chỉ số đạt được**:
  - **Accuracy**: `0.7580` (75.8%) ➔ **Vượt ngưỡng quy định (0.70 / 70%)**
  - **F1-Score**: `0.7572` (75.7%)
- **Lý do lựa chọn**: Cấu hình `n_estimators=300` và `max_depth=25` cho phép mô hình học được các quan hệ phi tuyến phức tạp giữa 12 đặc trưng hóa học rượu vang mà không bị overfitting quá mức, mang lại độ chính xác cao nhất trên tập đánh giá `eval.csv`.

---

### 2. Kiến Trúc Hạ Tầng & Pipeline CI/CD Tự Động (Bước 2 & 3)
1. **DVC & Cloud Object Storage**:
   - Khởi tạo DVC remote `myremote` lưu trữ dữ liệu trên GCP Storage Bucket: `gs://mlops-day21-2a202601405/dvc`.
   - Phiên bản hóa các tập dữ liệu `train_phase1.csv` (2,998 mẫu) và `train_phase2.csv` (5,996 mẫu sau khi gộp).
2. **GitHub Actions CI/CD Pipeline (`.github/workflows/mlops.yml`)**:
   - **Job 1 (Unit Test)**: Chạy `pytest tests/` kiểm thử code logic.
   - **Job 2 (Train)**: Xác thực GCP, `dvc pull` dữ liệu, huấn luyện mô hình và đẩy `model.pkl` lên GCS.
   - **Job 3 (Eval Gate)**: Chốt kiểm tra tự động `accuracy >= 0.70`.
   - **Job 4 (Deploy)**: SSH kết nối máy chủ ảo Cloud VM `mlops-serve` kích hoạt `systemd` service `mlops-serve.service` phục vụ FastAPI REST API tại cổng `8000`.

---

### 3. Các Khó Khăn Gặp Phải & Cách Giải Quyết
| STT | Khó khăn / Lỗi gặp phải | Nguyên nhân | Cách khắc phục |
|---|---|---|---|
| 1 | `Setuptools Deprecation` khi chạy MLflow | `setuptools>=70` gỡ bỏ `pkg_resources` | Khóa phiên bản `setuptools<70` (`69.5.1`) trong môi trường `.venv`. |
| 2 | `DVC Invalid Credentials (401)` trên GitHub Actions | `.dvc/config` chỉ định `credentialpath` cứng | Xóa `credentialpath` để DVC tự dùng biến chuẩn `GOOGLE_APPLICATION_CREDENTIALS`. |
| 3 | `SSH Handshake Failed` trên runner | Thư viện Go SSH bị kén định dạng key | Chuyển sang dùng lệnh OpenSSH chuẩn của Linux (`ssh -i ...`) trong runner. |
| 4 | `Username Invalid Characters` | Secret `VM_USER` dính khoảng trắng/xuống dòng | Thêm bộ lọc `tr -d '[:space:]"'` làm sạch tên user trong bash script. |
| 5 | `Curl Exit Code 7` tại bước Deploy | Server Uvicorn cần 2-4s để khởi động lại | Thêm vòng lặp thử lại (Retry loop 10 lần) kiểm tra `GET /health` mượt mà. |

---

### 4. Kết Luận
Hệ thống MLOps đã hoạt động hoàn hảo 100%. Khi bổ sung dữ liệu mới (`add_new_data.py`), việc đẩy commit con trỏ `.dvc` lên GitHub sẽ tự động kích hoạt toàn bộ pipeline huấn luyện lại và kiểm tra chất lượng mô hình trước khi deploy lên Cloud VM.

- **Kiểm tra Live API**: `curl http://35.247.134.181:8000/health` ➔ `{"status":"ok"}`
- **Dự đoán Live API**: `curl -X POST http://35.247.134.181:8000/predict` ➔ `{"prediction": 2, "label": "cao"}`
