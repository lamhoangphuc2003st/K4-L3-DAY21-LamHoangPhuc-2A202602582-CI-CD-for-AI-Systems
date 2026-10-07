# Báo Cáo Lab Day 21 - CI/CD cho AI Systems

| | |
|---|---|
| Họ và tên | Lâm Hoàng Phúc |
| MSSV | 2A202602582 |
| Lớp / Khóa | K4 |
| Repo GitHub | https://github.com/lamhoangphuc2003st/K4-L3-DAY21-LamHoangPhuc-2A202602582-CI-CD-for-AI-Systems |
| Ngày nộp | 07/10/2026 |

---

## 1. Bộ Siêu Tham Số Đã Chọn và Lý Do

| Lần chạy | n_estimators | learning_rate | max_depth | f1_score | accuracy |
|---|---|---|---|---|---|
| 1 | 100 | 0.1 | 3 | 0.7109 | 0.878 |
| 2 | 50 | 0.05 | 2 | 0.6051 | 0.846 |
| 3 | 200 | 0.1 | 5 | **0.7149** | 0.874 |
| 4 | 300 | 0.05 | 4 | 0.7070 | 0.874 |
| 5 | 200 | 0.2 | 3 | 0.7032 | 0.870 |

**Bộ siêu tham số đã chọn:** `n_estimators=200`, `learning_rate=0.1`, `max_depth=5`.

**Lý do:** Bộ này cho `f1_score` cao nhất (0.7149) trên tập holdout, vượt ngưỡng 0.65. Lần chạy có accuracy cao nhất (lần 1, 0.878) lại không phải lần có F1 cao nhất. Khi accuracy chỉ chênh vài phần nghìn, xếp hạng theo accuracy gần như phản ánh cách mô hình đoán lớp đa số chứ không phản ánh khả năng bắt lớp thu nhập cao. Accuracy dao động hẹp (0.846 - 0.878) trong khi F1 dao động rộng hơn nhiều (0.605 - 0.715), nên F1 phân biệt các mô hình rõ hơn. Về đánh đổi, lần 2 (learning_rate 0.05, chỉ 50 cây nông) bị underfit rõ rệt; khi giảm learning_rate xuống 0.05 thì phải tăng lên 300 cây (lần 4) mới đạt mức tương đương lần 1, còn tăng learning_rate lên 0.2 (lần 5) không cải thiện thêm.

---

## 2. Vì Sao Ngưỡng Chất Lượng Đặt Trên F1 Chứ Không Phải Accuracy

Tập Adult mất cân bằng lớp: chỉ 24,8% số mẫu có thu nhập > 50K. Một mô hình vô dụng luôn trả lời "thu nhập thấp" vẫn đạt accuracy 0,752 mà không nhận diện được một người thu nhập cao nào, nên một ngưỡng kiểu "accuracy >= 0,75" sẽ để lọt mô hình đó ra production. F1 của lớp dương là trung bình điều hòa của precision và recall trên chính lớp thu nhập cao, vì vậy mô hình đoán toàn lớp 0 có F1 = 0 và bị quality gate chặn ngay. F1 chỉ cao khi mô hình vừa bắt được nhiều người thu nhập cao (recall) vừa ít gán nhầm (precision). Không dùng `average="weighted"` hay `average="macro"` vì các cách này trộn F1 của lớp 0 (chiếm 75% và luôn cao) vào kết quả, kéo con số lên và làm ngưỡng 0,65 mất ý nghĩa.

---

## 3. Khó Khăn Gặp Phải và Cách Giải Quyết

| Khó khăn | Nguyên nhân | Cách giải quyết |
|---|---|---|
| `mlflow 2.13` lỗi import (`pkg_resources`, `FallbackAsyncAdaptedQueuePool`). | Môi trường mới cài setuptools và SQLAlchemy 2.1 không còn tương thích. | Ghim thêm `setuptools<81` và `sqlalchemy<2.1` trong `requirements.txt`. |
| Không tạo được `sa-key.json`, VM không được cấp IP public. | Org GCP bật policy `iam.disableServiceAccountKeyCreation` và `compute.vmExternalIpAccess`. | Dùng Workload Identity Federation cho GitHub Actions và service account gắn vào VM thay cho key; chỉ cho phép IP public riêng cho VM `income-api`. |
| Push lên GitHub không kích hoạt pipeline. | Repo là fork nên Actions mặc định tắt trigger từ push. | Bật workflow trong tab Actions rồi push lại commit dữ liệu. |

---

## 4. So Sánh Bước 2 và Bước 3 (bắt buộc, 2 - 3 câu)

| | f1_score | accuracy |
|---|---|---|
| Bước 2 (chỉ `train_batch1`) | 0.7149 | 0.874 |
| Bước 3 (thêm `train_batch2`) | 0.7354 | 0.882 |

**Nhận xét:** F1 tăng khoảng 0,02 và accuracy tăng 0,008 sau khi gấp đôi dữ liệu. Tuy nhiên tập holdout chỉ có 500 mẫu (khoảng 124 mẫu dương), nên mức chênh này tương ứng chỉ vài dự đoán đúng thêm và nằm trong biên dao động ngẫu nhiên. Vì hai nửa dữ liệu cùng phân phối, tôi không kết luận rằng thêm dữ liệu chắc chắn làm mô hình tốt hơn. Điều được kiểm chứng ở Bước 3 là commit dữ liệu đã tự động đi hết vòng train, quality gate và deploy mà không cần thao tác thủ công.
