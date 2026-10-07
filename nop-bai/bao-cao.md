# Báo Cáo Lab Day 21 - CI/CD cho AI Systems

| | |
|---|---|
| Họ và tên | Nguyễn Tuấn Anh |
| MSSV | 2A202602535 |
| Lớp / Khóa | K4 |
| Repo GitHub | https://github.com/Tuan2Anh/K4-L3-DAY21-NguyenTuanAnh-2A202602535-CI-CD-for-AI-Systems |
| Ngày nộp | 07/10/2026 |

---

## 1. Bộ Siêu Tham Số Đã Chọn và Lý Do

| Lần chạy | n_estimators | learning_rate | max_depth | f1_score | accuracy |
|---|---|---|---|---|---|
| 1 | 100 | 0.1 | 3 | 0.7109 | 0.8780 |
| 2 | 50 | 0.05 | 2 | 0.6051 | 0.8460 |
| 3 | 200 | 0.1 | 5 | 0.7149 | 0.8740 |

**Bộ siêu tham số đã chọn:** `n_estimators=200`, `learning_rate=0.1`, `max_depth=5`.

**Lý do:** Bộ siêu tham số ở lần 3 đạt `f1_score` cao nhất (0.7149), vượt qua ngưỡng quy định của Quality Gate (>= 0.65). Mặc dù lần chạy 1 có `accuracy` cao nhất (0.8780 so với 0.8740), nhưng lần 3 có F1 vượt trội hơn, phản ánh khả năng nhận diện lớp thiểu số (thu nhập > 50K) tốt hơn mà không bị sai lệch bởi lớp đa số. Khi giảm `learning_rate` xuống 0.05 và `max_depth` xuống 2 ở lần 2, mô hình quá nông và chưa kịp hội tụ dẫn đến `f1_score` giảm sâu xuống 0.6051 (dưới ngưỡng triển khai), cho thấy sự đánh đổi cần thiết giữa tốc độ học, số lượng cây và độ sâu để mô hình nắm bắt được ranh giới quyết định phi tuyến tính phức tạp.

---

## 2. Vì Sao Ngưỡng Chất Lượng Đặt Trên F1 Chứ Không Phải Accuracy

Tập dữ liệu Adult Census có sự mất cân bằng lớp nghiêm trọng với tỷ lệ người có thu nhập > 50K chỉ chiếm khoảng 24.8%, trong khi lớp <= 50K chiếm tới 75.2%. Do đó, một mô hình suy đoán ngây thơ luôn gán nhãn "thu nhập thấp" cho mọi cá nhân vẫn dễ dàng đạt `accuracy` xấp xỉ 75.2% mà hoàn toàn mất đi giá trị phân loại thực tế.

Chỉ số `accuracy` tính bình quân trên toàn bộ mẫu nên bị chi phối áp đảo bởi lớp đa số. Ngược lại, `f1_score` tính trên lớp dương (target = 1) là trung bình điều hòa giữa Precision (độ chuẩn xác khi dự đoán thu nhập cao) và Recall (độ bao phủ số người thu nhập cao thực tế). F1 buộc mô hình phải vừa hạn chế báo động giả vừa không bỏ sót nhóm mục tiêu. Tuyệt đối không sử dụng `average="weighted"` hay `average="macro"` vì các trọng số tính gộp này sẽ lại bị lớp đa số kéo tăng ảo, làm sai lệch ý nghĩa thực sự của ngưỡng kiểm soát chất lượng.

---

## 3. Khó Khăn Gặp Phải và Cách Giải Quyết

| Khó khăn | Nguyên nhân | Cách giải quyết |
|---|---|---|
| Lỗi cài đặt package `Invalid argument (os error 22)` | Dự án nằm trên phân vùng NTFS Windows (`/mnt/win_d`), hệ thống file NTFS cấm tên thư mục có dấu chấm ở cuối khi giải nén wheel. | Tạo môi trường ảo `.venv` trên phân vùng Linux gốc `btrfs` (`/home/tuananh/.venvs/day21`) và symlink vào thư mục dự án. |
| Xung đột thư viện `SQLAlchemy` với MLflow khi dùng SQLite | Phiên bản `sqlalchemy==2.1.3` mới nhất có breaking change làm lỗi import `FallbackAsyncAdaptedQueuePool` của `mlflow==2.13.0`. | Cố định phiên bản `sqlalchemy<2.1` và `setuptools<70` trong file `requirements.txt`. |
| Lỗi unpickle mô hình `AttributeError` khi khởi động API trên VM | VM cài mặc định phiên bản `scikit-learn` mới hơn (v1.7) trong khi mô hình được huấn luyện bằng `scikit-learn==1.4.2`. | Đồng bộ cài đặt chính xác `scikit-learn==1.4.2` và `joblib==1.4.2` trên VM và khởi động lại service. |

---

## 4. So Sánh Bước 2 và Bước 3 (bắt buộc, 2 - 3 câu)

| | f1_score | accuracy |
|---|---|---|
| Bước 2 (chỉ `train_batch1`) | 0.7149 | 0.8740 |
| Bước 3 (thêm `train_batch2`) | 0.7354 | 0.8820 |

**Nhận xét:** Khi bổ sung thêm 22.361 mẫu dữ liệu mới từ `train_batch2`, cả `f1_score` và `accuracy` đều tăng (F1 tăng từ 0.7149 lên 0.7354 và accuracy tăng từ 0.8740 lên 0.8820). Dữ liệu gia tăng với quy mô gấp đôi giúp mô hình học thêm được nhiều đặc trưng biên và tổng quát hóa tốt hơn, đồng thời toàn bộ quy trình CI/CD đã tự động kích hoạt, huấn luyện, kiểm thử và release thành công lên server mà không cần can thiệp thủ công.
