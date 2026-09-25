# Báo cáo Lab Ngày 08: Học chủ động cho bộ phát hiện xe

Họ và tên: Lê Vĩnh Hưng

Công cụ gán nhãn đã dùng: CVAT Docker trên máy cá nhân

## 1. Dữ liệu và cách chia tập

Tại sao tập chưa gán nhãn (pool) và tập kiểm thử (test set) được chia theo trục thời gian, có vùng đệm ở giữa, thay vì chia ngẫu nhiên? Nếu chia ngẫu nhiên, số đo trên tập kiểm thử sẽ bị lệch theo hướng nào, và vì sao?

Tập pool và tập test được chia theo trục thời gian với một khoảng đệm ở giữa nhằm đảm bảo tính độc lập và phân bố thực tế giữa dữ liệu huấn luyện/chưa gán nhãn và dữ liệu kiểm thử. Trong video giám sát giao thông, các khung hình liền kề nhau có độ tương quan cực kỳ cao (gần trùng lặp về bối cảnh, góc quay, ánh sáng và vị trí xe). Nếu chia ngẫu nhiên, các ảnh trong tập test sẽ xuất hiện rất gần hoặc trùng lặp với ảnh trong tập train, dẫn đến hiện tượng rò rỉ thông tin (data leakage). Khi đó, số đo trên tập kiểm thử (như AP50) sẽ bị thổi phồng (lệch theo hướng cao giả tạo) và không phản ánh đúng năng lực tổng quát hóa của mô hình đối với các tình huống mới, thời điểm mới trong thực tế.

## 2. Mô hình khởi đầu lạnh (cold start)

Chép dòng vòng 0 từ `rounds_table.md`. Dựa vào `outputs/compare_round0.jpg`, cho biết mô hình khởi đầu lạnh không khớp nhãn tham chiếu ở những loại xe nào. Độ phủ (recall) theo kích thước xe cho thấy điều gì? Một trường hợp nào cần người rà lại nhãn tham chiếu trước khi kết luận mô hình sai?

- Số liệu vòng 0: 
  - Model: `yolov8n cold start (COCO car+bus+truck)`
  - Ảnh train: `0`, Box train: `0`
  - AP50: `0.771`, P@0.25: `0.925`, R@0.25: `0.489`, F1: `0.640`
  - R small: `0.182`, R medium: `0.547`, R large: `0.561`
- Dựa vào `compare_round0.jpg`, mô hình khởi đầu lạnh (chạy pre-trained COCO) thường gặp khó khăn và không khớp với nhãn tham chiếu ở các nhóm: xe ở xa (vật thể nhỏ, chỉ nhìn thấy hai chấm đèn mờ), xe bị che khuất một phần trong dòng phương tiện đông đúc, hoặc các xe có ánh sáng đèn pha chói lóa làm nhiễu đường biên thân xe.
- Độ phủ theo kích thước xe (`R small = 0.182` rất thấp so với `R medium = 0.547` và `R large = 0.561`) cho thấy mô hình YOLOv8n gốc rất kém trong việc phát hiện các vật thể nhỏ (xe ở xa), trong khi xử lý khá hơn đối với xe vừa và lớn.
- Trường hợp cần rà lại nhãn tham chiếu: Các xe ở cực xa (chỉ còn chấm đèn nhỏ dưới 16 pixel) hoặc các vùng phản chiếu ánh đèn đường/mặt đường mà nhãn tham chiếu tự động có thể gán nhãn nhầm hoặc bỏ sót, cần con người kiểm tra kỹ trước khi vội kết luận mô hình dự đoán sai.

## 3. Chiến lược chọn mẫu

Giải thích bằng lời công thức `score = W_U·U + W_A·A + W_D·D` và vai trò của `MIN_GAP_S`. Dẫn ba frame trong `reports/SELECTION.md` và một frame khác để chứng minh cách bạn cân nhắc độ bất định, ảnh gần trùng và công gán nhãn. Điểm bất định có chứng minh ảnh đó sẽ cải thiện mô hình không? Vì sao?

- Công thức `score = W_U·U + W_A·A + W_D·D`:
  - `U` (Uncertainty): Đo lường mức độ bất định của mô hình (các dự đoán có độ tự tin lấp lửng quanh ngưỡng quyết định).
  - `A` (Ambiguity / Appearance complexity): Đo lường độ phức tạp hoặc mật độ đối tượng/mức độ mập mờ trong khung hình.
  - `D` (Diversity / Distance): Đo lường sự khác biệt hoặc khoảng cách không gian/thời gian so với các mẫu đã chọn nhằm tránh chọn các ảnh bị lặp lại.
  - `W_U, W_A, W_D` là các trọng số tương ứng.
- Vai trò của `MIN_GAP_S`: Đảm bảo khoảng cách thời gian tối thiểu giữa các ảnh được chọn, tránh việc chọn hàng loạt các khung hình liên tiếp có nội dung gần như giống hệt nhau (giảm thiểu redundancy, tối ưu hóa công sức gán nhãn).
- Dẫn chứng từ `SELECTION.md`: 
  - Các frame như `frame_0182.jpg` (hạng 1, điểm cao nhất) và `frame_0369.jpg` (hạng 2, mật độ cao 43 boxes) được ưu tiên để khai thác thông tin phong phú.
  - `frame_0331.jpg` được chọn kết hợp kiểm tra khung lân cận (`frame_0330.jpg`) để quản lý ảnh gần trùng.
  - Trường hợp `frame_0372.jpg` (hạng 6, điểm cao 0.9101 nhưng không chọn) minh họa việc loại bỏ các ảnh dư thừa nằm sát các frame đã chọn để tiết kiệm công gán nhãn.
- Điểm bất định cao **không** đảm bảo chắc chắn rằng việc đưa ảnh đó vào huấn luyện sẽ làm mô hình tốt lên, bởi vì độ bất định cao có thể xuất phát từ nhiễu (ảnh quá mờ, điều kiện ánh sáng cực đoan, hoặc nhãn bị lỗi) thay vì mang lại thông tin cấu trúc hữu ích cho quá trình học của mô hình.

## 4. Các vòng học chủ động (active learning)

Chép bảng từ `rounds_table.md`. Với mỗi vòng, trình bày:
- mức độ bạn đã sửa nhãn gợi ý (số box giữ nguyên, chỉnh sửa, xoá, thêm mới, lấy từ `outputs/round*_diff.md`);
- AP50 thay đổi bao nhiêu so với khởi đầu lạnh và so với vòng trước;
- nhóm xe nào tốt lên hoặc xấu đi theo số đo trên cùng tập test.
Dựa vào các ảnh `compare_round*.jpg`, chỉ ra một ca kết quả đổi sau fine-tune (tốt hơn hoặc xấu đi), cùng lý do có thể kiểm. Dùng `BLIND_SCAN.md`, `REVIEW_LOG.csv` và `round1_diff.md` phân biệt quan sát độc lập, lỗi pre-label đã sửa và kết quả mô hình sau train. Mô tả một ca khó theo guideline.

- Bảng dữ liệu vòng 0 (`rounds_table.md`):

| vòng | model | ảnh train | box train | AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |

- Vòng 1 (`round1_diff.md`):
  - Số lượng ảnh sửa: 12 ảnh. Model đề xuất 169 box, sau khi sửa thành 336 box.
  - Chi tiết thao tác: 122 box giữ nguyên (`accepted`), 30 box chỉnh sửa (`edited`), 17 box xóa (`deleted` - FP của model), và 184 box bổ sung (`added` - FN của model). Tỉ lệ chấp nhận (accept rate) là 72%.
- Phân biệt quan sát độc lập, lỗi pre-label và kết quả sau train:
  - Quan sát độc lập (`BLIND_SCAN.md` với `frame_0369.jpg`): Nhìn bằng mắt thường ghi nhận 38 xe, lưu ý các vùng khuất ở góc trái trên và góc phải dưới.
  - Lỗi pre-label đã sửa (`round1_diff.md`): Mô hình ban đầu bỏ sót nhiều xe ở vùng tối/xa (cần thêm 184 box) và nhận diện nhầm một số vùng sáng (xóa 17 FP).
  - Kết quả mô hình sau train: Phản ánh qua các chỉ số đánh giá trên tập test sau khi fine-tune.
- Xử lý xe khó theo `GUIDELINE_LABEL.md`: Đối với các xe ở rất xa chỉ còn chấm đèn hoặc xe bị mờ do chuyển động, tuân thủ nguyên tắc vẽ box ôm sát vùng thân xe đoán được quanh cụm đèn, không khoanh vùng vệt sáng phản chiếu trên mặt đường.

## 5. Kết luận và giới hạn

Kết quả vòng này so với cold start ra sao? Vì sao bạn dừng hoặc tiếp tục? Đề xuất hai ca còn yếu hoặc bất định cho vòng sau, kèm chi phí rà nhãn và nguy cơ ảnh gần trùng. Tập kiểm thử chỉ 20 ảnh, có luật bỏ qua xe quá nhỏ và nhãn tham chiếu do mô hình tạo chưa được rà thủ công; các giới hạn đó ảnh hưởng thế nào đến kết luận? Nếu AP50 giảm, bạn sẽ kiểm tra điều gì trước khi train thêm?

- Kết quả & Hướng đi: Vòng 1 cung cấp lượng dữ liệu gán nhãn bổ sung đáng kể (thêm 184 box từ 12 ảnh được chọn lọc kỹ qua chiến lược active learning). Việc tiếp tục sang vòng 2 giúp củng cố độ chính xác cho các vùng biên, xe ở xa và xe bị che khuất.
- Đề xuất hai ca còn yếu/bất định cho vòng sau: Các khung hình có mật độ xe dày đặc vào ban đêm (tương tự `frame_0369.jpg` hoặc các frame có độ sáng phức tạp ở góc cua) với chi phí rà nhãn cao do số lượng xe lớn, đi kèm nguy cơ ảnh gần trùng cao nếu chọn các frame lân cận trong cùng một nhịp di chuyển.
- Giới hạn: 
  - Tập kiểm thử nhỏ (20 ảnh) có thể chưa đại diện đầy đủ cho toàn bộ dải tình huống giao thông dài.
  - Luật bỏ qua xe nhỏ (<16px) loại bỏ nhiễu nhưng cũng làm mất đi khả năng đánh giá năng lực phát hiện xe ở xa.
  - Nhãn tham chiếu do mô hình tạo chưa được rà thủ công tuyệt đối có thể chứa sai sót tiềm ẩn.
- Nếu AP50 giảm sau khi train: Cần kiểm tra lại chất lượng gán nhãn thủ công (có bị lệch chuẩn so với guideline không), kiểm tra xem có hiện tượng overfitting do dữ liệu train quá tương đồng với test, hoặc kiểm tra xem learning rate và các siêu tham số huấn luyện có phù hợp hay không trước khi tiếp tục.