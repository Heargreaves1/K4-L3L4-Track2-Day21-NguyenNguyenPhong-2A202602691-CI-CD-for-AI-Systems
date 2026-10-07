# Báo Cáo Lab Day 21 - CI/CD cho AI Systems

| | |
|---|---|
| Họ và tên | Nguyễn Nguyên Phong |
| MSSV | 2A202602691 |
| Lớp / Khóa | K4 |
| Repo GitHub | https://github.com/Heargreaves1/K4-L3L4-Track2-Day21-NguyenNguyenPhong-2A202602691-CI-CD-for-AI-Systems |
| Ngày nộp | 07/10/2026 |

---

## 1. Bộ Siêu Tham Số Đã Chọn và Lý Do

| Lần chạy | n_estimators | learning_rate | max_depth | f1_score | accuracy |
|---|---|---|---|---|---|
| 1 | 100 | 0.1 | 3 | 0.7109 | 0.8780 |
| 2 | 50 | 0.05 | 2 | 0.6051 | 0.8460 |
| 3 | 200 | 0.1 | 5 | 0.7149 | 0.8740 |

**Bộ siêu tham số đã chọn:** `n_estimators=200`, `learning_rate=0.1`, `max_depth=5`.

**Lý do:** Bộ siêu tham số này đạt điểm F1 cao nhất (0.7149) trên tập holdout, vượt qua ngưỡng chất lượng 0.65 để đủ điều kiện triển khai vào môi trường sản xuất. Đáng chú ý, lần chạy có accuracy cao nhất là Lần 1 (0.8780), nhưng Lần 3 mới là lần đạt F1 cao nhất (0.7149 so với 0.7109). Điều này chỉ ra rằng accuracy bị kéo lên bởi lớp đa số (thu nhập thấp), trong khi F1 phản ánh chuẩn xác năng lực phân loại của lớp thiểu số (thu nhập cao). Đồng thời, ta quan sát thấy sự đánh đổi: khi giảm learning_rate từ 0.1 xuống 0.05 và giảm n_estimators từ 100 xuống 50 (Lần 2), mô hình bị underfitting nghiêm trọng, khiến F1 tụt xuống chỉ còn 0.6051 và vi phạm ngưỡng chặn của Quality Gate.

---

## 2. Vì Sao Ngưỡng Chất Lượng Đặt Trên F1 Chứ Không Phải Accuracy

Tập dữ liệu Adult Census có phân bố mất cân bằng nghiêm trọng khi chỉ có 24.8% số mẫu thuộc lớp thu nhập cao (>50K) và tới 75.2% thuộc lớp thu nhập thấp (<=50K). Một mô hình ngây thơ luôn dự đoán mọi cá nhân đều thuộc diện "thu nhập thấp" vẫn sẽ đạt độ chính xác (accuracy) lên tới 75.2%, tạo cảm giác sai lệch về một mô hình tối ưu nhưng thực chất hoàn toàn vô dụng khi không nhận diện được bất kỳ trường hợp thu nhập cao nào (F1 = 0.0).

Chỉ số F1-score của lớp dương (target = 1) là trung bình điều hòa giữa Precision và Recall của nhóm thu nhập cao, đo lường toàn diện mức độ chính xác và độ bao phủ của lớp mục tiêu. Khi tính toán, ta không sử dụng `average="weighted"` hay `average="macro"` vì trọng số áp đảo của lớp đa số (75.2%) sẽ thổi phồng điểm số, làm vô hiệu hóa ý nghĩa của Quality Gate trong các bài toán mất cân bằng lớp.

---

## 3. Khó Khăn Gặp Phải và Cách Giải Quyết

| Khó khăn | Nguyên nhân | Cách giải quyết |
|---|---|---|
| Lỗi import `FallbackAsyncAdaptedQueuePool` khi MLflow kết nối SQLite | MLflow 2.13.0 không tương thích với SQLAlchemy 2.1.x mới nhất | Cố định ràng buộc `sqlalchemy<2.0.35` trong `requirements.txt` và hạ về SQLAlchemy 2.0.34. |
| Pipeline CI/CD có thể thất bại khi tải dữ liệu từ DVC remote | CI runner bắt đầu chạy ngay khi push git trong khi file dữ liệu chưa đẩy lên storage | Luôn thực hiện `dvc push` dữ liệu lên cloud storage hoàn tất trước khi tiến hành `git push`. |
| Nguy cơ so sánh sai ngưỡng chất lượng tại Quality Gate | GitHub Actions outputs luôn trả về giá trị ở dạng chuỗi ký tự (string) | Ép kiểu dữ liệu `float()` tường minh trong script kiểm tra của job Quality Gate trước khi so sánh. |

---

## 4. So Sánh Bước 2 và Bước 3 (bắt buộc, 2 - 3 câu)

| | f1_score | accuracy |
|---|---|---|
| Bước 2 (chỉ `train_batch1`) | 0.7149 | 0.8740 |
| Bước 3 (thêm `train_batch2`) | 0.7188 | 0.8760 |

**Nhận xét:** Khi bổ sung thêm 22.361 mẫu dữ liệu mới từ `train_batch2`, điểm F1 tăng nhẹ từ 0.7149 lên 0.7188 và accuracy giữ mức ổn định. Sự thay đổi không quá đột biến vì cả hai tập dữ liệu đều được lấy mẫu ngẫu nhiên từ cùng một phân phối dân số ban đầu, qua đó khẳng định giá trị cốt lõi của Bước 3 là chứng minh quy trình CI/CD tự động kích hoạt và tái triển khai trơn tru mà không cần bất kỳ can thiệp thủ công nào.
