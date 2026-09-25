# Báo cáo Lab Ngày 08: Học chủ động cho bộ phát hiện xe

Họ và tên: Lý Hồng Phúc

Công cụ gán nhãn đã dùng: CVAT (Docker chạy local)

## 1. Dữ liệu và cách chia tập

Camera trong video đứng yên và video là một cảnh quay liên tục (không có scene cut nào vượt 0.08,
trong khi một lần cắt cảnh thật thường trên 0.3 — xem `data/DATA.md`). Vì vậy hai ảnh cách nhau
0.4 giây gần như giống hệt nhau, và mỗi chiếc xe ở lại trong khung hình liên tục vài giây. Nếu chia
tập pool/test ngẫu nhiên, cùng một chiếc xe rất dễ xuất hiện ở cả hai tập cùng lúc (ảnh liền kề nhau
theo thời gian). Khi đó mô hình được chấm một phần trên chính những chiếc xe nó đã "nhìn thấy" lúc
huấn luyện — đây là rò rỉ dữ liệu (data leakage), và số đo AP50/recall sẽ **bị đẩy lên cao hơn** khả
năng tổng quát hoá thật của mô hình, khiến kết luận về chất lượng model trở nên lạc quan giả tạo.

Để tránh việc này, `data/DATA.md` chia dữ liệu theo trục thời gian: 20 ảnh test lấy từ 4 đoạn cách
đều (tâm giây 20/60/100/140), loại bỏ vùng đệm 112 ảnh quanh mỗi đoạn test, phần còn lại (268 ảnh)
mới là pool. Ảnh pool gần test nhất vẫn cách 4.4 giây — đủ xa để một chiếc xe không thể xuất hiện ở
cả hai tập.

## 2. Mô hình khởi đầu lạnh (cold start)

Từ `reports/rounds_table.md`, vòng 0:

| vòng | model | AP50 | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | yolov8n cold start (COCO car+bus+truck) | 0.771 | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |

Recall theo kích thước xe chênh lệch rất rõ: xe nhỏ (small) chỉ đạt recall 0.182, trong khi xe vừa
và lớn đạt 0.547–0.561 — mô hình COCO gốc gần như không nhận ra xe nhỏ/ở xa trong ảnh đêm, đúng như
`GUIDELINE_LABEL.md` cảnh báo (xe xa chỉ còn hai chấm đèn rất dễ bị bỏ sót). Xem `outputs/compare_round0.jpg`:
ở `frame_0050` và `frame_0350`, các box vàng (FN — bỏ sót) tập trung ở nhóm xe nhỏ phía xa/giữa ảnh,
trong khi các box xanh lá (TP) chủ yếu là xe lớn, gần camera, có đèn pha rõ. `frame_0350` còn có vài
box đỏ (FP) nằm giữa một cụm xe nhỏ sát nhau — nhiều khả năng model chỉ lệch/nhân đôi box trong vùng
đông xe chứ không hẳn "bịa" ra xe không tồn tại.

Trường hợp nên rà lại nhãn tham chiếu trước khi kết luận model sai: các box đỏ (FP) trong cụm xe nhỏ
ở `frame_0350` — vì nhãn tham chiếu của tập test cũng do một mô hình khác tạo ra và **chưa được
người rà** (`data/DATA.md`), rất có thể đó là xe thật mà nhãn tham chiếu bỏ sót, không phải model
"đoán nhầm".

## 3. Chiến lược chọn mẫu

Công thức `score = W_U·U + W_A·A + W_D·D` (W_U=0.5, W_A=0.3, W_D=0.2) kết hợp ba tín hiệu: `U` là độ
bất định trung bình của 5 box khó nhất trong ảnh (cao nhất khi conf ≈ 0.5); `A` là số box "mơ hồ"
(0.15 ≤ conf < 0.5), chuẩn hoá theo giá trị lớn nhất trong pool — bắt các vùng model tạo nhiều nhiễu;
`D` là khoảng cách thời gian tới ảnh đã gán gần nhất (tối đa 10s) — khuyến khích chọn ảnh đa dạng,
không trùng cảnh. `MIN_GAP_S=2.0` chặn việc chọn hai ảnh trong cùng lô cách nhau dưới 2 giây, vì
camera đứng yên nên hai ảnh quá gần gần như trùng lặp hoàn toàn, tốn công rà mà ít thêm thông tin.

Ba frame trong lô + một frame khác (chi tiết và bằng chứng ở `reports/SELECTION.md`):

- `frame_0182.jpg` (rank 1, score 0.9591) — score tổng hợp cao nhất lô.
- `frame_0369.jpg` (rank 2, score 0.9324, U cao nhất) — sau khi sửa có 25/38 box là `added`, xác
  nhận U cao đúng là do model bỏ sót nhiều xe thật.
- `frame_0331.jpg` (rank 5, score 0.9154, A=1.0) — sau khi sửa có 6 box bị `deleted` (trùng/giả),
  nhiều nhất lô, khớp với A cao (nhiều box ở vùng conf mơ hồ).
- `frame_0372.jpg` (rank 6, score 0.9101, KHÔNG được chọn) — bị loại vì chỉ cách `frame_0369` đã
  chọn 1.2 giây (< `MIN_GAP_S`), minh hoạ vai trò chống ảnh gần trùng.

Điểm bất định **không chứng minh** ảnh đó sẽ cải thiện mô hình. Đây là tín hiệu đo trên chính model
hiện tại, phản ánh model "không chắc" ở đâu tại thời điểm đó, không phải phép đo nhân quả về hiệu quả
huấn luyện sau này. Bằng chứng trực tiếp trong chính lab này: vòng 1 chọn đúng 12 ảnh có U/A cao nhất
để sửa, nhưng sau khi fine-tune AP50 lại giảm mạnh (0.771 → 0.401, xem mục 4).

## 4. Các vòng học chủ động (active learning)

Từ `reports/rounds_table.md`:

| vòng | model | ảnh train | box train | AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | cold start | 0 | 0 | 0.771 | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |
| 1 | fine-tune vòng 1 | 12 | 333 | 0.401 | **-0.370** | 1.000 | 0.104 | 0.189 | 0.000 | 0.101 | 0.293 |

**Mức độ sửa nhãn gợi ý** (`outputs/round1_diff.md`): model đề xuất 169 box cho 12 ảnh, sau khi rà
còn 333 box — **accepted 133 (79%)**, **edited 19**, **deleted 17** (box giả/trùng), **added 181**
(xe bị bỏ sót). Tức hơn một nửa số box cuối cùng (181/333 ≈ 54%) là do người thêm hoàn toàn mới —
pre-label ở ngưỡng conf 0.25 bỏ sót rất nhiều xe, đặc biệt xe nhỏ/tối.

**AP50 sau fine-tune giảm mạnh** so với cold start (-0.370), và recall theo mọi kích thước xe đều
xấu đi, xe nhỏ rớt xuống **0 tuyệt đối** (0.182 → 0.000). Precision lại đạt 1.000 (tuyệt đối), nghĩa
là model sau fine-tune trở nên rất "nhát" — chỉ dự đoán khi cực kỳ chắc chắn nên gần như không còn
false positive, nhưng bỏ sót phần lớn xe thật.

Ca cụ thể trong `outputs/compare_round1.jpg`: ở `frame_0050`, cold start bắt được TP 11/18 xe, còn
model vòng 1 chỉ còn **TP 0/18** — toàn bộ 18 box tham chiếu đều thành FN (vàng). Đây là ca xấu đi rõ
rệt, có thể kiểm chứng trực tiếp trên ảnh: cột "round 1" gần như trống box so với cột "cold start".

Nguyên nhân nhiều khả năng là **overfit/quên catastrophic**: chỉ 12 ảnh (333 box) cho một class,
huấn luyện 50 epoch, không tách tập validation (`val=False`, dùng trọng số `last.pt`), và mỗi vòng
huấn luyện lại từ `yolov8n.pt` gốc — dữ liệu quá ít khiến model học lệch theo đặc điểm riêng của 12
ảnh này, đánh mất khả năng tổng quát mà model COCO gốc vốn có.

Phân biệt ba nguồn thông tin:
- **Quan sát độc lập** (`reports/BLIND_SCAN.md`): trên `frame_0187`, tôi đếm được 26 xe bằng mắt
  trước khi xem nhãn AI, và ghi trước hai vị trí dễ bị bỏ sót/vẽ sai (một xe nhỏ/tối chỉ còn đốm đèn
  mờ, một vệt sáng nhoè do chuyển động dễ bị vẽ lệch box).
- **Lỗi pre-label đã sửa** (`reports/REVIEW_LOG.csv`, dẫn chứng thực tế từ `frame_0227.jpg`, đối
  chiếu trực tiếp CVAT DB với `prelabels/`): một ca `added` (xe nhỏ/tối AI bỏ sót hoàn toàn), một ca
  `deleted` (box AI trùng lên cùng một xe tải đã có box khác), một ca `edited` (box AI lệch, không
  ôm sát nóc/đuôi xe tải).
- **Kết quả model sau train** (`outputs/round1_diff.md` + `metrics_round1.json`): dù đã sửa đúng
  79% box AI và thêm 181 box mới, model sau fine-tune vẫn kém hơn cold start — cho thấy chất lượng
  nhãn sửa tay không phải là yếu tố duy nhất quyết định, cấu hình huấn luyện (số ảnh, epoch, thiếu
  validation) cũng ảnh hưởng lớn.

Ca khó theo guideline: xe nhỏ/tối gần giữa `frame_0227.jpg`, chỉ còn thấy cụm đèn pha mờ, thân xe
chìm vào nền tối. Theo `GUIDELINE_LABEL.md` ("chỉ thấy đèn, thân xe tối nhưng vẫn đoán được đường
viền → vẽ box theo phần thân đoán được quanh cụm đèn, không chỉ khoanh hai chấm đèn"), tôi vẽ box
ôm theo vùng thân xe suy đoán được thay vì chỉ khoanh hai chấm sáng.

## 5. Kết luận và giới hạn

Vòng 1 **kém hơn** cold start trên mọi chỉ số ngoài precision (AP50 -0.370, recall -0.385, F1
-0.451, recall xe nhỏ về 0). Tôi **chưa kết luận nên dừng hay tiếp tục** ngay: kết quả xấu đi quá
lớn và quá đồng loạt (mọi nhóm kích thước xe đều tệ hơn, recall xe nhỏ về đúng 0) để chỉ giải thích
bằng "lô nhỏ, nhiễu ngẫu nhiên" — nhiều khả năng là overfit do 12 ảnh/50 epoch/không validation, nên
trước khi train thêm ở vòng 2, việc cần kiểm trước là: (1) xác nhận không có lỗi ánh xạ toạ độ/class
khi ghép `labels/round1/` (vì recall xe nhỏ về đúng 0 là dấu hiệu bất thường, không chỉ là "giảm
nhẹ"), và (2) thử fine-tune với ít epoch hơn hoặc học từ trọng số vòng trước thay vì luôn train lại
từ `yolov8n.pt` gốc.

Hai ca còn yếu/bất định đề xuất cho vòng sau, lấy từ `outputs/selection_round2.csv` (tính trên chính
model vòng 1 — vốn đã kém, nên các số U/score ở đây thấp hơn vòng 1 vì model hiện tại tự tin nhưng
sai nhiều hơn là "biết mình không chắc"):

- `frame_0074.jpg` (rank 1, score 0.7427, U=0.6568, model vòng 1 chỉ còn đề xuất 7 box) — chi phí
  rà thấp (ít box), nhưng đúng vùng model hiện tại yếu nhất nên cần ưu tiên kiểm.
- `frame_0291.jpg` (rank 2, score 0.6936) — `frame_0075.jpg` (rank 3, score 0.69, cách `frame_0074`
  chỉ 0.4 giây) bị loại vì vi phạm `MIN_GAP_S`, nên `frame_0291.jpg` là lựa chọn tiếp theo không
  trùng cảnh.

Giới hạn của kết luận: tập test chỉ 20 ảnh/403 box tham chiếu hợp lệ, nên theo `data/DATA.md`,
chênh lệch dưới khoảng 0.01 AP50 giữa hai vòng vốn đã không đủ ý nghĩa — ở đây chênh lệch 0.370 là
rất lớn nên tôi tin đây là tín hiệu thật, nhưng cỡ mẫu nhỏ vẫn khiến ước lượng theo từng nhóm kích
thước xe (đặc biệt "small", chỉ 66 box) có phương sai cao. Ngoài ra, nhãn tham chiếu của tập test do
model khác tạo và chưa được người rà, nên một phần "sai" đo được có thể là do nhãn tham chiếu sai chứ
không phải model — mục 2 đã nêu một ca cụ thể cần rà lại. Luật bỏ qua xe cao dưới 16px cũng làm số đo
không phản ánh đầy đủ khả năng nhận diện xe rất nhỏ/rất xa của model.
