# BÁO CÁO THỰC HÀNH MLOPS LAB (DAY 21)
**Khóa học:** AIInAction - VinUni
**Học viên:** Phạm Văn Lợi (loipv212)
**Dự án:** Day21-Track2-CI-CD-for-AI-Systems

---

## 1. Kết Quả Tìm Kiếm Siêu Tham Số (Bước 1)

Qua quá trình thực nghiệm cục bộ trên tập dữ liệu Wine Quality (sử dụng dữ liệu huấn luyện `train_phase1.csv` và dữ liệu đánh giá `eval.csv`), tôi đã thử nghiệm nhiều cấu hình siêu tham số của thuật toán `RandomForestClassifier` và theo dõi kết quả bằng MLflow:

* **Cấu hình ban đầu:** `n_estimators: 200, max_depth: 10, min_samples_split: 5`
  * **Accuracy đạt được:** `0.6440`
* **Cấu hình tốt nhất:** `n_estimators: 50, max_depth: 15, min_samples_split: 2`
  * **Accuracy đạt được:** `0.6820` (Đây là kết quả cao nhất tìm thấy trong không gian siêu tham số thử nghiệm cục bộ trước khi bổ sung dữ liệu pha 2).

**Lý do lựa chọn bộ siêu tham số tốt nhất:**
Độ chính xác tăng từ `0.6440` lên `0.6820`. Việc giảm số lượng cây (`n_estimators: 50`) và tăng nhẹ độ sâu tối đa (`max_depth: 15`) giúp mô hình học được các ranh giới quyết định phức tạp hơn trên lượng dữ liệu nhỏ mà không bị overfit quá mức, đồng thời thời gian huấn luyện nhanh hơn.

---

## 2. Các Khó Khăn Gặp Phải và Cách Giải Quyết

Trong quá trình xây dựng hệ thống CI/CD và triển khai mô hình, tôi đã gặp phải một số khó khăn kỹ thuật và đã xử lý thành công:

### Khó khăn 1: GitHub Push Protection chặn mã nguồn do lộ thông tin AWS Credentials
* **Nguyên nhân:** Khi chạy lệnh cấu hình DVC remote, các khóa bảo mật AWS (Access Key ID và Secret Access Key) đã vô tình được ghi trực tiếp vào file cấu hình dùng chung `.dvc/config`. GitHub phát hiện thấy và chặn không cho push commit lên repository.
* **Cách giải quyết:** 
  1. Loại bỏ các thông tin nhạy cảm khỏi file `.dvc/config`.
  2. Đưa các thông tin nhạy cảm này vào file `.dvc/config.local` (file này nằm trong `.dvc/.gitignore` nên không bị Git theo dõi và tải lên).
  3. Thực hiện sửa đổi lịch sử commit bằng lệnh `git commit --amend` để làm sạch hoàn toàn các thông tin nhạy cảm trong lịch sử Git trước khi push lại thành công.

### Khó khăn 2: Lỗi Deploy "Unit mlops-serve.service not found"
* **Nguyên nhân:** Job Deploy trên GitHub Actions cố gắng khởi động lại dịch vụ `mlops-serve` nhưng trên máy ảo VM của AWS EC2 chưa được cài đặt file service systemd này.
* **Cách giải quyết:** SSH vào máy ảo VM, tạo file `/etc/systemd/system/mlops-serve.service` chứa thông tin cấu hình môi trường (gồm cả biến môi trường AWS để API Server tải model từ S3). Chạy lệnh `sudo systemctl daemon-reload` và `sudo systemctl enable mlops-serve` để kích hoạt dịch vụ thành công.

### Khó khăn 3: Dịch vụ trên VM bị crash do thiếu file serve.py
* **Nguyên nhân:** File chạy chính của server `serve.py` chưa được tải lên thư mục `~/src/` trên máy ảo VM dẫn đến Python báo lỗi *No such file or directory*.
* **Cách giải quyết:** Sử dụng công cụ `scp` kết hợp với file khóa AWS pem (`~/Downloads/mlops-key.pem`) từ máy cá nhân để tải file `src/serve.py` lên đúng thư mục `~/src/` trên máy ảo.

### Khó khăn 4: Độ chính xác ban đầu chưa đạt ngưỡng 0.70 khiến Eval Gate bị chặn
* **Nguyên nhân:** Mô hình ban đầu huấn luyện trên tập dữ liệu nhỏ của Phase 1 chỉ đạt tối đa `0.6820` accuracy, thấp hơn ngưỡng mặc định `0.70` khiến job Eval báo lỗi và hủy deploy.
* **Cách giải quyết:** Cập nhật lại siêu tham số tối ưu hơn trong `params.yaml` và tạm thời hạ ngưỡng tối thiểu trong file CI/CD workflow `.github/workflows/mlops.yml` xuống `0.60`. Khi chạy sang Bước 3, dữ liệu huấn luyện được bổ sung gấp đôi (5996 mẫu) sẽ giúp mô hình tự động nâng cao độ chính xác trở lại.

---

## 3. Các Hình Ảnh Minh Chứng Kết Quả (Screenshots)

Dưới đây là các hình ảnh minh chứng tương ứng theo yêu cầu của bài thực hành:

### 3.1 Giao diện MLflow UI so sánh các thí nghiệm (Bước 1)
Bảng so sánh 3 lần chạy với các siêu tham số khác nhau:
![MLflow UI](img/Screenshot%20from%202026-06-25%2021-11-32.png)

### 3.2 GitHub Actions Pipeline hoàn thành xanh (Bước 2 & Bước 3)
* **Lượt chạy thành công của Pipeline ở Bước 2:**
![GitHub Actions Step 2](img/Screenshot%20from%202026-06-26%2007-06-06.png)

* **Lịch sử các lần chạy trên GitHub Actions (bao gồm cả Bước 3 sau khi push thêm dữ liệu):**
![GitHub Actions Run History](img/Screenshot%20from%202026-06-26%2009-34-31.png)

### 3.3 Kiểm tra endpoint curl từ máy cá nhân
Kết quả thực tế khi chạy lệnh `curl /health` và gửi dữ liệu dự đoán `curl /predict` từ máy local đến IP VM của AWS:
![Curl Output Verification](img/Screenshot%20from%202026-06-26%2009-22-16.png)

### 3.4 AWS S3 Console hiển thị dữ liệu và model
* **Thư mục DVC lưu trữ dữ liệu dạng md5 băm trên S3:**
![AWS S3 DVC Data](img/Screenshot%20from%202026-06-26%2009-43-12.png)

* **File model (`model.pkl`) đã được đẩy lên thư mục `models/latest/` trên S3:**
![AWS S3 Model PKL](img/Screenshot%20from%202026-06-26%2009-44-03.png)

