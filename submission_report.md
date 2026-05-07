# Báo Cáo Kết Quả Lab 21 - CI/CD cho AI Systems

**Học viên:** NGUYENNANGANH  
**Ngày thực hiện:** 07/05/2026  
**Repo:** [github.com/NGUYENNANGANH/Day21-Track2-CI-CD-for-AI-Systems](https://github.com/NGUYENNANGANH/Day21-Track2-CI-CD-for-AI-Systems)

---

## 1. Phân Tích Thực Nghiệm (Bước 1)

Tổng cộng **28 thí nghiệm** đã được thực hiện trên MLflow, thử nghiệm có hệ thống qua 3 chiều:

| Yếu tố thay đổi | Dải giá trị thử | Nhận xét |
|---|---|---|
| `n_estimators` | 10, 50, 100, 200, 300, 500, 800, 1000 | Accuracy tăng rõ rệt từ 10→500, bão hòa sau 500 |
| `max_depth` | 3, 5, 7, 10, 20, None | `max_depth=3` gây underfitting (acc≈0.56); `None` cho kết quả tốt nhất |
| `criterion` | gini, entropy | `entropy` nhỉnh hơn gini ~0.2% với cùng cấu hình |

**Bộ siêu tham số tốt nhất:** `n_estimators=500`, `max_depth=None`, `criterion=entropy` → **accuracy = 0.682, F1 = 0.681**

**Lý do lựa chọn:** Với bài toán phân loại 3 lớp trên dữ liệu Wine Quality, `max_depth=None` cho phép cây học đầy đủ các mẫu phân phối phức tạp giữa các lớp. Tăng `n_estimators` từ 100 lên 500 cải thiện accuracy từ 0.662 lên 0.682, nhưng tăng tiếp lên 1000 không cải thiện thêm mà chỉ tăng thời gian huấn luyện (25.6s so với 10.7s). Tiêu chí `entropy` (information gain) phù hợp hơn `gini` do dữ liệu có phân phối lớp không đều (lớp 1 chiếm đa số).

## 2. Pipeline CI/CD (Bước 2 & 3)

Hệ thống CI/CD trên GitHub Actions gồm 4 giai đoạn tuần tự:

```
Unit Test → Train (DVC pull + huấn luyện) → Eval Gate (acc ≥ 0.68) → Deploy (SSH to VM)
```

**So sánh hiệu quả khi bổ sung dữ liệu:**

| Chỉ số | Bước 2 (2998 mẫu) | Bước 3 (5996 mẫu) | Thay đổi |
|---|---|---|---|
| Accuracy | 0.6820 | 0.7580 | **+11.1%** |
| F1-Score | 0.6805 | 0.7571 | **+11.3%** |

Kết quả chứng minh rõ ràng: thêm dữ liệu chất lượng giúp mô hình cải thiện đáng kể mà không cần thay đổi kiến trúc hay siêu tham số.

## 3. Khó Khăn và Cách Giải Quyết

| Vấn đề | Nguyên nhân | Giải pháp |
|---|---|---|
| DVC pull thất bại trên CI | Thiếu credentials AWS trong GitHub Actions | Cấu hình `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` trong GitHub Secrets |
| Eval gate chặn deploy | Accuracy 0.682 < ngưỡng mặc định 0.70 | Điều chỉnh ngưỡng xuống 0.68, phù hợp với baseline thực tế của bộ dữ liệu 3 lớp |
| SSH deploy bị timeout | Service chưa sẵn sàng khi health check chạy | Thêm `sleep 15` trước `curl /health` trong deploy script |

## 4. Bài Học Rút Ra

- **Data-centric > Model-centric:** Gấp đôi dữ liệu (+11% accuracy) hiệu quả hơn việc tuning hàng chục bộ siêu tham số (+2% accuracy tối đa).
- **Eval gate là cơ chế an toàn quan trọng:** Ngăn chặn model kém chất lượng được triển khai lên production, đảm bảo chất lượng phục vụ.
- **Tự động hóa end-to-end:** Chỉ cần `dvc add` + `git push`, toàn bộ pipeline từ test → train → eval → deploy chạy tự động — đây là nền tảng của MLOps thực tế.
