# Vì sao chọn lô này?

Trong 50 dòng đứng đầu `outputs/selection_round1.csv`, chọn năm frame bạn sẽ ưu tiên nếu chỉ có
ngân sách rà năm ảnh. Ghi tên, điểm, thời điểm, thứ tự và lý do; tối thiểu một quyết định phải xét
ảnh gần trùng hoặc trường hợp model không dự đoán được box:

1. **frame_0182.jpg** (thứ 1, 72.8s, score 0.9591): Điểm bất định U=0.9182 và độ đa dạng D=1.0, chứng tỏ model rất bất định về nhãn. Frame này có 28 box với 18 box bị đánh dấu ambiguous, cho thấy nhiều xe có thể bị model bỏ sót hoặc gán nhãn sai. Xét đến chi phí rà 28 box, frame này có score cao nhất và đáng đầu tư.

2. **frame_0369.jpg** (thứ 2, 147.6s, score 0.9324): U=0.9315 và D=0.8889, model rất bất định. Có 43 box với 16 ambiguous, nhiều xe đặc biệt cần rà. Vị trí thời gian 147.6s không trùng với các frame đã chọn.

3. **frame_0380.jpg** (thứ 3, 152.0s, score 0.9170): U=0.934, D=1.0, score cao với độ đa dạng tối đa. 40 box với 15 ambiguous, cho thấy nhiều trường hợp khó. Thời gian 152.0s không trùng với frame 0182 (72.8s) hay 0369 (147.6s).

4. **frame_0326.jpg** (thứ 4, 130.4s, score 0.9155): U=0.931, D=1.0, 39 box với 15 ambiguous. Thời gian 130.4s nằm giữa 0182 và 0369, đảm bảo phủ khác vùng thời gian.

5. **frame_0187.jpg** (thứ 10, 74.8s, score 0.8995): Mặc dù thứ 10 nhưng có A=0.9444 (độ linh hoạt cao) và D=1.0. Có 39 box với 17 ambiguous. Thời gian 74.8s rất gần với frame_0182 (72.8s) - đây là trường hợp ảnh gần trùng. Tuy nhiên, do có nhiều box ambiguous và model rất bất định về nhãn, frame này vẫn đáng xem xét. Chi phí rà 39 box là chấp nhận được.

Ba frame thuộc lô 12 ảnh model chọn và bằng chứng trong CSV/ảnh contact sheet:

- **frame_0182.jpg** (thứ 1 trong top 50, được model chọn): Có score cao nhất 0.9591, model đã chọn frame này và sau khi rà, số box tăng từ 13 lên 21, thêm 8 box (FN của model). Điều này chứng minh điểm bất định cao tương quan với việc model bỏ sót nhiều xe.
- **frame_0369.jpg** (thứ 2 trong top 50, được model chọn): Score 0.9324, model đã chọn. Sau rà, số box tăng từ 14 lên 26, thêm 12 box. Điểm U cao (0.9315) chứng tỏ model rất bất định về nhãn ở frame này.
- **frame_0331.jpg** (thứ 5 trong top 50, được model chọn): Score 0.9154, model đã chọn. Sau rà, số box tăng từ 20 lên 22, thêm 9 box và xóa 7 box FP. Điểm D=1.0 cho thấy đa dạng cao.

Một frame có điểm cao nhưng không chọn hoặc một frame có điểm thấp vẫn nên xem, và lý do:

Frame **frame_0000.jpg** (thứ 92, score 0.787) có điểm khá cao nhưng không được chọn vì có t_sec=0.0 (gần đầu video) và n_boxes=24, n_ambiguous=8. Mặc dù điểm U=0.9074 cao, nhưng do nằm ở đầu video (gần frame train), nguy cơ ảnh gần trùng với tập train là rất cao. Chi phí rà 24 box là đáng kể so với lợi ích dùng cho fine-tune.

Điều phép chọn này chưa chứng minh về chất lượng mô hình: Phép chọn dựa trên độ bất định và độ đa dạng của model chứ không dựa trên nhãn thật. Những frame có điểm cao có thể chứa nhiều nhiễu (FP) hoặc nhiều xe khó (FN) khiến model bất định, nhưng chưa chắc rằng sửa những frame này sẽ cải thiện đáng kể chất lượng mô hình. Chất lượng mô hình được đánh giá trên tập test độc lập (20 ảnh), không phụ thuộc vào phép chọn này.