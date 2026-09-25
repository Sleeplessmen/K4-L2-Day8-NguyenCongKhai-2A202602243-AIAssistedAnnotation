# Báo cáo Lab Ngày 08: Học chủ động cho bộ phát hiện xe

Họ và tên: Nguyễn Công Khải

Công cụ gán nhãn đã dùng: CVAT

`outputs/round*_diff.md`. Không coi nhãn test do mô hình tạo là chân lý tuyệt đối.

## 1. Dữ liệu và cách chia tập

Tại sao tập chưa gán nhãn (pool) và tập kiểm thử (test set) được chia theo trục thời gian, có vùng
dệm ở giữa, thay vì chia ngẫu nhiên? Nếu chia ngẫu nhiên, số đo trên tập kiểm thử sẽ bị lệch theo
hướng nào, và vì sao?

Tập pool và tập test được chia theo trục thời gian với vùng đệm ở giữa nhằm tránh rò rỉ thông tin giữa tập train và test. Nếu chia ngẫu nhiên, sẽ có nguy cơ cao các frame liền kề (có cùng xe trong khung hình) rơi vào cả hai tập, khiến mô hình được train trên tập train có thể "nhớ" được các đặc trưng của xe trong tập test (do xe xuất hiện liên tục qua nhiều frame), dẫn đến đánh giá quá lạc quan (over-optimistic) trên tập test. Chia theo trục thời gian đảm bảo các frame trong tập test đều ở thời điểm sau tập train, loại bỏ hoàn toàn nguy cơ rò rỉ thông tin này.

## 2. Mô hình khởi đầu lạnh (cold start)

Chép dòng vòng 0 từ `rounds_table.md`. Dựa vào `outputs/compare_round0.jpg`, cho biết mô hình khởi
dầu lạnh không khớp nhãn tham chiếu ở những loại xe nào. Độ phủ (recall) theo kích thước xe cho
thấy điều gì? Một trường hợp nào cần người rà lại nhãn tham chiếu trước khi kết luận mô hình sai?

Dòng vòng 0 từ `rounds_table.md`:
- AP50: 0.771
- P@0.25: 0.925
- R@0.25: 0.489
- F1: 0.640
- R small: 0.182
- R medium: 0.547
- R large: 0.561

Mô hình khởi đầu lạnh (YOLOv8n pretrained trên COCO car+bus+truck) có precision rất cao (0.925) nhưng recall thấp (0.489), đặc biệt recall trên xe nhỏ chỉ 0.182. Điều này cho thấy model bỏ sót nhiều xe, nhất là xe nhỏ. Trên `compare_round0.jpg`, model không khớp nhãn tham chiếu ở các xe nhỏ ở xa camera hoặc bị che khuất một phần. Độ phủ theo kích thước cho thấy recall của xe nhỏ rất thấp (0.182) so với xe trung bình (0.547) và xe lớn (0.561), chứng tỏ model khó phát hiện các vật thể nhỏ.

Một trường hợp cần rà lại nhãn tham chiếu: các xe ở xa camera hoặc bị che khuất có thể có nhãn tham chiếu không đầy đủ (bỏ sót hoặc box không chính xác). Trước khi kết luận mô hình sai, cần kiểm tra rằng nhãn tham chiếu có đủ các xe trên ảnh hay không, đặc biệt là xe nhỏ.

## 3. Chiến lược chọn mẫu

Giải thích bằng lời công thức `score = W_U·U + W_A·A + W_D·D` và vai trò của `MIN_GAP_S`.
Dẫn ba frame trong `reports/SELECTION.md` và một frame khác để chứng minh cách bạn cân nhắc
độ bất định, ảnh gần trùng và công gán nhãn. Điểm bất định có chứng minh ảnh đó sẽ cải thiện
mô hình không? Vì sao?

Công thức `score = W_U·U + W_A·A + W_D·D` kết hợp ba yếu tố:
- **U (Uncertainty - độ bất định)**: Đo mức độ model không chắc chắn về nhãn. W_U là trọng số cho yếu tố này.
- **A (Ambiguity - độ mơ hồ)**: Đo số lượng box ambiguous (có thể bị sai) trong frame. W_A là trọng số.
- **D (Diversity - độ đa dạng)**: Đo mức độ frame khác biệt so với các frame đã chọn. W_D là trọng số.

`MIN_GAP_S` là khoảng thời gian tối thiểu giữa các frame được chọn, nhằm tránh chọn các frame gần trùng nhau (cùng xe trong khung hình).

Ba frame trong `reports/SELECTION.md` và lý do:
1. **frame_0182.jpg** (score 0.9591): U=0.9182, D=1.0, 28 box với 18 ambiguous. Sau khi rà (xem `round1_diff.md`), số box tăng từ 13 lên 21, thêm 8 box. Điều này chứng minh điểm bất định cao tương quan với việc model bỏ sót xe. Frame này cải thiện mô hình vì bổ sung các box bị bỏ sót.

2. **frame_0369.jpg** (score 0.9324): U=0.9315, D=0.8889, 43 box với 16 ambiguous. Sau rà, box tăng từ 14 lên 26, thêm 12 box. Điểm bất định cao chứng tỏ model rất không chắc chắn, và việc thêm nhiều box sau rà sẽ cải thiện recall.

3. **frame_0380.jpg** (score 0.9170): U=0.934, D=1.0, 40 box với 15 ambiguous. Điểm cao và độ đa dạng tối đa, đảm bảo bổ sung thông tin mới.

Frame **frame_0000.jpg** (score 0.787, thứ 92) có điểm cao nhưng không chọn vì nằm ở t_sec=0.0, gần đầu video. Minh họa cho việc: Mặc dù có U cao (0.9074), nhưng frame này có thể gần trùng với các frame train, nên chi phí rà 24 box có thể không xứng đáng so với lợi ích cho fine-tune.

Điểm bất định có chứng minh ảnh đó sẽ cải thiện mô hình: Có, nhưng không hoàn toàn. Các frame có điểm bất định cao thường chứa nhiều box bị bỏ sót (FN) hoặc box sai (FP), và sau khi sửa, mô hình có thêm dữ liệu chất lượng cao. Tuy nhiên, không phải mọi frame bất định đều hữu ích - cần cân nhắc chi phí rà nhãn và nguy cơ ảnh gần trùng.

## 4. Các vòng học chủ động (active learning)

Chép bảng từ `rounds_table.md`. Với mỗi vòng, trình bày:

- mức độ bạn đã sửa nhãn gợi ý (số box giữ nguyên, chỉnh sửa, xoá, thêm mới, lấy từ
  `outputs/round*_diff.md`);
- AP50 thay đổi bao nhiêu so với khởi đầu lạnh và so với vòng trước;
- nhóm xe nào tốt lên hoặc xấu đi theo số đo trên cùng tập test.

Dựa vào các ảnh `compare_round*.jpg`, chỉ ra một ca kết quả đổi sau fine-tune (tốt hơn hoặc xấu
di), cùng lý do có thể kiểm. Dùng `BLIND_SCAN.md`, `REVIEW_LOG.csv` và `round1_diff.md` phân biệt
quan sát độc lập, lỗi pre-label đã sửa và kết quả mô hình sau train. Mô tả một ca khó theo guideline.

Bảng từ `rounds_table.md`:

| vòng | model | ảnh train | box train | AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |
| 1 | yolov8n fine-tune vong 1..1 | 12 | 241 | 0.468 | -0.304 | 1.000 | 0.164 | 0.281 | 0.000 | 0.128 | 0.683 |

**Vòng 1:**
- Sửa nhãn: Model đề xuất 169 box, sau sửa còn 241 box (tăng 72 box). Cụ thể: accepted 141 box, edited 12 box, deleted 16 FP, added 88 FN (xem `round1_diff.md`).
- AP50: 0.468, giảm 0.304 so với cold start (0.771 - 0.468 = -0.304).
- P@0.25 tăng từ 0.925 lên 1.000 (không có false positive ở ngưỡng 0.25).
- R@0.25 giảm mạnh từ 0.489 xuống 0.164.
- F1 giảm từ 0.640 xuống 0.281.
- R small giảm xuống 0.000 (không phát hiện được xe nhỏ nào).
- R medium giảm xuống 0.128.
- R large tăng lên 0.683.

So với vòng 0, AP50 giảm đáng kể (-0.304). Điều này có vẻ trái ngược, nhưng có thể giải thích:
1. Nhãn của vòng 1 (12 ảnh) có thể chưa đủ đại diện cho toàn bộ không gian đặc trưng.
2. Các nhãn được sửa có thể có một số box sai (do quá trình gán nhãn thủ công) hoặc các xe được thêm có thể là những trường hợp khó mà model chưa thể học tốt với chỉ 12 ảnh.
3. Thiếu các frame train ở những khu vực thời gian khác nhau.

**Ca thay đổi trên `compare_round1.jpg`:** Trên ảnh so sánh, có thể thấy model sau fine-tune phát hiện thêm một số xe lớn (R large tăng từ 0.561 lên 0.683) nhưng lại bỏ sót nhiều xe nhỏ hơn (R small giảm xuống 0.000).

**Phân biệt quan sát độc lập, lỗi pre-label, kết quả mô hình:**
- **Quan sát độc lập** (`BLIND_SCAN.md`): Frame frame_0099.jpg, thấy 22 xe, hai vị trí dễ bỏ sót là xe ở góc dưới bên phải (bị cắt bởi mép ảnh, nhòe) và xe màu vàng ở bên phải (chỉ sáng 1 đèn).
- **Lỗi pre-label đã sửa** (`REVIEW_LOG.csv`): Trên frame_0099.jpg, thêm 2 box cho xe gần mép dưới (FN của model), thêm box cho xe cạnh nhóm 2 xe chung box, sửa box xe màu vàng (chưa bao trùm hết thân xe). Trên frame_0107.jpg, xóa 2 box chung cho 2 xe, thêm 3 box riêng cho từng xe.
- **Kết quả mô hình sau train** (`round1_diff.md`): Sau khi sửa, tổng số box tăng từ 169 lên 241, chứng tỏ model ban đầu bỏ sót rất nhiều xe.

**Ca khó theo guideline:** Xe bị che khuất bởi xe khác hoặc vật cản (theo GUIDELINE_LABEL.md: "Chỉ gán nhãn xe từ 4 bánh trở lên có ít nhất 50% diện tích xe hiển thị"). Ví dụ: hai xe đậu song song, xe phía sau bị xe phía trước che hơn 50% diện tích - không gán nhãn. Hoặc xe bị cây che nhưng vẫn thấy đủ 50% thân xe - vẫn gán nhãn.

## 5. Kết luận và giới hạn

Kết quả vòng này so với cold start ra sao? Vì sao bạn dừng hoặc tiếp tục? Đề xuất hai ca còn yếu
hoặc bất định cho vòng sau, kèm chi phí rà nhãn và nguy cơ ảnh gần trùng. Tập kiểm thử chỉ 20 ảnh,
có luật bỏ qua xe quá nhỏ và nhãn tham chiếu do mô hình tạo chưa được rà thủ công; các giới hạn đó
ảnh hưởng thế nào đến kết luận? Nếu AP50 giảm, bạn sẽ kiểm tra điều gì trước khi train thêm?

So với cold start (AP50=0.771), vòng 1 có AP50=0.468, giảm đáng kể (-0.304). Kết quả này cho thấy fine-tune trên 12 ảnh được chọn bằng active learning chưa cải thiện được mô hình. Lý do có thể:
- Số lượng ảnh train (12) quá ít so với không gian đặc trưng đa dạng.
- Nhãn của 12 ảnh có thể chưa đủ chất lượng hoặc đại diện.
- Model cần nhiều vòng học chủ động hơn để hội tụ.

Tôi quyết định **tiếp tục** vì một vòng chưa đủ để đánh giá hiệu quả của active learning. Theo lý thuyết, sau nhiều vòng, mô hình sẽ dần cải thiện.

**Đề xuất hai ca cho vòng sau:**
1. **frame_0372.jpg** (thứ 6, score 0.9101, 148.8s): U=0.9202, D=1.0, 42 box với 15 ambiguous. Chi phí rà 42 box, nguy cơ trùng với frame_0369 (147.6s) và frame_0374 (149.6s) là thấp (MIN_GAP_S đảm bảo khoảng cách).
2. **frame_0313.jpg** (thứ 31, score 0.8433, 125.2s): U=0.92, D=0.6111, 28 box với 11 ambiguous. Chi phí rà 28 box, điểm U cao chứng tỏ model rất bất định.

**Giới hạn ảnh hưởng đến kết luận:**
- Tập test chỉ 20 ảnh: Sai số thống kê cao, kết quả có thể biến động mạnh.
- Bỏ qua xe quá nhỏ (cao dưới 16 px): Loại trừ các trường hợp khó, có thể làm gia tăng giả tạo các số đo như recall.
- Nhãn test do mô hình tạo chưa được rà: Nhãn test có thể có sai sót, khiến đánh giá AP50 không hoàn toàn chính xác.

**Nếu AP50 giảm, sẽ kiểm tra:**
1. Kiểm tra chất lượng nhãn của các frame train (có box sai không, có đủ xe không).
2. Kiểm tra sự phân bố của các frame train (có phủ đủ các khu vực thời gian không).
3. Kiểm traem các tham số train (learning rate, epochs) có phù hợp không.
4. Kiểm tra liệu có sự không tương thích giữa nhãn train và nhãn test.