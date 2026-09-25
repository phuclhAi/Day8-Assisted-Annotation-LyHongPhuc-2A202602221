# Vì sao chọn lô này?

Trong 50 dòng đứng đầu `outputs/selection_round1.csv`, nếu chỉ đủ công rà 5 ảnh, tôi ưu tiên:

| Ưu tiên | Frame | Score | t (s) | Rank | Lý do |
| --- | --- | ---: | ---: | ---: | --- |
| 1 | frame_0182.jpg | 0.9591 | 72.8 | 1 | Score cao nhất lô: U=0.918 (nhiều box ở vùng bất định 0.15–0.5 conf), A=1.0 (tỉ lệ box mơ hồ tối đa trong pool), D=1.0 (ảnh đầu tiên nên khoảng cách thời gian tới ảnh đã gán tối đa). |
| 2 | frame_0369.jpg | 0.9324 | 147.6 | 2 | U cao nhất round1 (0.9315); sau khi sửa, frame này có tới 25 box bị model bỏ sót (added) — nhiều nhất trong 12 ảnh, đúng như điểm bất định dự đoán. |
| 3 | frame_0331.jpg | 0.9154 | 132.4 | 5 | A=1.0 nhưng U thấp hơn (0.8308) — mô hình tự tin sai: sau khi sửa, frame này có 6 box bị xoá (deleted) vì trùng/box giả, cao nhất trong 12 ảnh, cho thấy A bắt đúng vùng model tạo nhiều box nhiễu. |
| 4 | frame_0372.jpg (không được chọn) | 0.9101 | 148.8 | 6 | Score cao thứ 6 nhưng bị loại vì chỉ cách frame_0369 (đã chọn, t=147.6) đúng 1.2 giây — dưới `MIN_GAP_S=2.0`. Cùng một quãng xe gần như trùng cảnh, chọn thêm sẽ tốn công rà mà không thêm nhiều thông tin mới. |
| 5 | frame_0187.jpg | 0.8995 | 74.8 | 10 | Score thấp hơn 4 ảnh trên nhưng vẫn nằm trong ngân sách 5 ảnh; đây cũng là ảnh tôi dùng cho `BLIND_SCAN.md` — quan sát độc lập trước khi xem nhãn AI khớp với vùng bất định mà score dự đoán. |

Ba frame thuộc lô 12 ảnh model chọn (bằng chứng CSV + `selection_round1.jpg`):

- **frame_0182.jpg** (rank 1, score 0.9591) — ảnh được chọn đầu tiên, contact sheet ghi `t=72.8s ss=0.96`, đúng điểm tổng hợp cao nhất.
- **frame_0369.jpg** (rank 2, score 0.9324) — `outputs/round1_diff.md` cho thấy model chỉ đề xuất 14 box nhưng sau khi rà có 38 box (25 box `added`), xác nhận U cao là có căn cứ: model thật sự bỏ sót rất nhiều xe ở ảnh này.
- **frame_0331.jpg** (rank 5, score 0.9154) — model đề xuất 20 box nhưng có tới 6 box bị xoá vì trùng/giả (deleted), khớp với A=1.0 (nhiều box ở vùng conf mơ hồ 0.15–0.5, dấu hiệu model không chắc).

Một frame có điểm cao nhưng không được chọn: **frame_0372.jpg** (score 0.9101, hạng 6) — bị loại vì cách frame_0369 (đã chọn) chỉ 1.2 giây, vi phạm `MIN_GAP_S=2.0`. Vì camera đứng yên, các xe trong hai ảnh gần như là cùng một nhóm xe, rà thêm ảnh này tốn công nhưng ít thông tin mới — đúng vai trò chống trùng lặp của tham số `D`/`MIN_GAP_S`.

Điều phép chọn này **chưa chứng minh** về chất lượng mô hình: điểm bất định (uncertainty) chỉ là một phép đo dựa trên độ tự tin của model **trước khi** huấn luyện lại — nó cho biết model "không chắc" ở đâu, không đảm bảo rằng gán nhãn xong các ảnh này rồi fine-tune sẽ làm model tốt hơn. Bằng chứng trực tiếp: vòng 1 chọn đúng những ảnh có U/A cao nhất để sửa, nhưng sau khi fine-tune AP50 lại **giảm** từ 0.771 xuống 0.401 (xem `reports/REPORT.md` mục 4–5) — chọn mẫu tốt không tự động đảm bảo huấn luyện tốt, đặc biệt với lô quá nhỏ (12 ảnh) và không có tập validation.
