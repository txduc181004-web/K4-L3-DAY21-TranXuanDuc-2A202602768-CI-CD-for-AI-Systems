# Báo Cáo Lab Day 21 - CI/CD cho AI Systems

| | |
|---|---|
| Họ và tên | Trần Xuân Đức |
| MSSV | 2A202602768 |
| Lớp / Khóa | K4 |
| Repo GitHub | https://github.com/txduc181004-web/K4-L3-DAY21-TranXuanDuc-2A202602768-CI-CD-for-AI-Systems |
| Ngày nộp | 07/10/2026 |

---

## 1. Bộ Siêu Tham Số Đã Chọn và Lý Do

| Lần chạy | n_estimators | learning_rate | max_depth | f1_score | accuracy |
|---|---|---|---|---|---|
| 1 | 100 | 0.1 | 3 | 0.7109 | **0.8780** |
| 2 | 50 | 0.05 | 2 | 0.6051 | 0.8460 |
| 3 | 200 | 0.1 | 5 | **0.7149** | 0.8740 |
| 4 | 300 | 0.05 | 4 | 0.7070 | 0.8740 |
| 5 | 200 | 0.2 | 3 | 0.7032 | 0.8700 |

**Bộ siêu tham số đã chọn:** `n_estimators=200`, `learning_rate=0.1`, `max_depth=5`.

**Lý do:** Lần chạy 3 có `f1_score` cao nhất (0.7149), tức là bắt được nhiều người thu nhập cao nhất mà vẫn giữ precision ổn. Lần có accuracy cao nhất lại là lần 1 (0.878), không trùng với lần có F1 cao nhất. Điều này cho thấy accuracy chủ yếu phản ánh lớp đa số và không đủ để chọn mô hình. Accuracy của 5 lần chỉ dao động trong khoảng 0.846 - 0.878, còn F1 dao động từ 0.605 đến 0.715. Lần 2 vẫn có accuracy 0.846 nhưng F1 dưới ngưỡng 0.65. Về đánh đổi: giảm `learning_rate` xuống 0.05 thì phải tăng `n_estimators` lên 300 và tăng độ sâu (lần 4) mới đạt lại mức F1 ~0.71. Ngược lại, lần 2 vừa có learning rate thấp vừa ít cây và cây nông nên học thiếu rõ rệt.

---

## 2. Vì Sao Ngưỡng Chất Lượng Đặt Trên F1 Chứ Không Phải Accuracy

Tập Adult mất cân bằng: chỉ 24,8% mẫu có thu nhập > 50K. Một mô hình luôn trả lời "thu nhập thấp" đã đạt accuracy 0,752 dù không nhận ra được một người thu nhập cao nào. Vì vậy accuracy cao không chứng minh mô hình hữu ích, và ngưỡng đặt trên accuracy sẽ cho qua cả một mô hình vô dụng. F1 của lớp dương là trung bình điều hòa giữa precision (trong số người được dự đoán là thu nhập cao, bao nhiêu người đúng) và recall (trong số người thu nhập cao thật, mô hình bắt được bao nhiêu). Mô hình "luôn đoán thấp" có F1 = 0, nên ngưỡng F1 ≥ 0,65 loại được nó ngay. Khi gọi `f1_score` không được dùng `average="weighted"` hay `average="macro"`. Hai cách này trộn thêm F1 của lớp đa số (vốn rất cao, khoảng 0,9), nên con số bị kéo lên và có thể vượt ngưỡng dù lớp dương vẫn bị dự đoán kém.

---

## 3. Khó Khăn Gặp Phải và Cách Giải Quyết

| Khó khăn | Nguyên nhân | Cách giải quyết |
|---|---|---|
| `train.py` lỗi import khi dùng MLflow | mlflow 2.13 cần `pkg_resources` (bị loại khỏi venv mới) và không tương thích SQLAlchemy 2.1 | Ghim `setuptools<81`, `sqlalchemy<2.1`, `numpy==1.26.4` trong `requirements.txt` |
| Không tạo được S3 bucket | IAM user của lab chỉ có quyền EC2/VPC/IAM, không có quyền S3 | Tạo IAM user riêng `income-lab-ci` có quyền tối thiểu trên đúng một bucket; EC2 đọc model qua IAM Role, không lưu key trên VM |
| `git push` không kích hoạt pipeline | Repo là fork, GitHub tắt workflow chạy theo push trên fork mới | Bật workflow trong tab Actions; trước đó chạy thủ công bằng `workflow_dispatch` |

---

## 4. So Sánh Bước 2 và Bước 3 (bắt buộc, 2 - 3 câu)

| | f1_score | accuracy |
|---|---|---|
| Bước 2 (chỉ `train_batch1`) | 0.7149 | 0.874 |
| Bước 3 (thêm `train_batch2`) | 0.7354 | 0.882 |

**Nhận xét:** F1 tăng khoảng 0,02 nhưng mức tăng này rất nhỏ. Trên 500 mẫu holdout (124 mẫu dương), mô hình mới chỉ bắt thêm 3 người thu nhập cao và giảm 1 dự đoán nhầm. Vì `train_batch2` được chia ngẫu nhiên từ cùng nguồn nên có cùng phân phối với `train_batch1`. Do đó mức tăng này nằm trong biên dao động của tập holdout nhỏ, không chứng minh được rằng gấp đôi dữ liệu làm mô hình tốt hơn. Điều Bước 3 thực sự chứng minh là commit dữ liệu tự kích hoạt pipeline và đưa mô hình mới lên VM mà không cần thao tác thủ công.

---

## 5. Phần Bổ Sung Ngoài Yêu Cầu

- Quality gate chặn Release: chạy pipeline với bộ tham số yếu (lần 2, F1 = 0,6051) trên nhánh `demo/quality-gate-fail`. Kết quả là job Quality Gate thất bại và Release bị bỏ qua (ảnh `07-quality-gate-chan.png`).
- Job Train chỉ upload model vào `artifacts/staging/<commit>/`. Chỉ khi qua quality gate, job Release mới promote model sang `artifacts/current/`. Nhờ vậy một model không đạt ngưỡng không bao giờ ghi đè model đang phục vụ.
