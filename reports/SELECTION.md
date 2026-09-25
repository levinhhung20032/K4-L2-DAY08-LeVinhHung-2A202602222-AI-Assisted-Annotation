# Vì sao chọn lô này?

Trong 50 dòng đứng đầu `outputs/selection_round1.csv`, chọn năm frame bạn sẽ ưu tiên nếu chỉ có
ngân sách rà năm ảnh. Ghi tên, điểm, thời điểm, thứ tự và lý do; tối thiểu một quyết định phải xét
ảnh gần trùng hoặc trường hợp model không dự đoán được box: 
1. `frame_0182.jpg` (Điểm: 0.9591, Thời điểm: 72.8s, Thứ tự: Hạng 1): Là ảnh có điểm số cao nhất trong tập dữ liệu, mô hình tự tin rất lớn, giúp kiểm tra độ chính xác của các phát hiện rõ nét nhất.
2. `frame_0369.jpg` (Điểm: 0.9324, Thời điểm: 147.6s, Thứ tự: Hạng 2): Có mật độ xe cộ cao (43 boxes), kiểm tra khả năng xử lý của mô hình ở vùng có độ phức tạp lớn.
3. `frame_0331.jpg` (Điểm: 0.9154, Thời điểm: 132.4s, Thứ tự: Hạng 5): Điểm số cao, đồng thời xét đến các khung hình lân cận (`frame_0330.jpg` ở hạng 12) để kiểm tra hiện tượng ảnh gần trùng (redundancy) xem mô hình có bị lặp khung hình thừa thãi hay không.
4. `frame_0187.jpg` (Điểm: 0.8995, Thời điểm: 74.8s, Thứ tự: Hạng 10): Nằm ngay sau cụm xung quanh `frame_0182.jpg`, giúp đánh giá sự thay đổi trạng thái giao thông trong khoảng thời gian ngắn.
5. `frame_0392.jpg` (Điểm: 0.8874, Thời điểm: 156.8s, Thứ tự: Hạng 15): Đại diện cho khung hình ở cuối chuỗi thời gian được chọn, kiểm tra độ ổn định của mô hình khi video kéo dài.

Ba frame thuộc lô 12 ảnh model chọn và bằng chứng trong CSV/ảnh contact sheet: 
1. `frame_0182.jpg`: Xếp hạng 1 trong CSV với điểm tổng hợp 0.9591 (`selected = True`), xuất hiện ở vị trí đầu tiên trên ảnh ghép (`outputs/selection_round1.jpg`), thể hiện rõ các xe di chuyển ngược chiều/cùng chiều với ánh đèn pha rõ nét.
2. `frame_0369.jpg`: Xếp hạng 2 trong CSV với điểm tổng hợp 0.9324 (`selected = True`), xuất hiện ở giữa ảnh ghép hàng trên, có tới 43 boxes phát hiện đối tượng, phản ánh năng lực phát hiện tốt trong điều kiện giao thông đông đúc.
3. `frame_0380.jpg`: Xếp hạng 3 trong CSV với điểm tổng hợp 0.9170 (`selected = True`), nằm ở góc phải hàng trên của ảnh ghép, xác nhận mô hình ưu tiên chọn các frame có điểm độ tin cậy và không gian mẫu cao.

Một frame có điểm cao nhưng không chọn hoặc một frame có điểm thấp vẫn nên xem, và lý do: 
`frame_0372.jpg` (Hạng 6, điểm 0.9101, `selected = False`) có điểm số rất cao (đứng ngay sau top 5), nằm sát `frame_0369.jpg` (Hạng 2) và `frame_0368.jpg` (Hạng 9, không chọn). Việc không chọn frame này dù điểm cao giúp tránh tình trạng lãng phí nguồn lực rà soát các ảnh quá gần nhau (redundant frames) khi nội dung và các xe trên đường chưa có sự thay đổi đáng kể so với các frame xung quanh đã được chọn.

Điều phép chọn này chưa chứng minh về chất lượng mô hình: 
Phương pháp chọn lọc này mới chỉ đánh giá được điểm số tin cậy và mức độ tự tin tổng hợp của mô hình dựa trên các tiêu chí nội tại (như số lượng box, điểm số ước lượng), nhưng chưa chứng minh được độ chính xác thực tế trên mặt đất (ground truth accuracy) như: tỷ lệ nhận diện sót đối tượng (false negatives), tỷ lệ nhận diện nhầm/dương tính giả (false positives), hay khả năng phân định chính xác các xe trong điều kiện ánh sáng chói lóa từ đèn pha ô tô ban đêm.