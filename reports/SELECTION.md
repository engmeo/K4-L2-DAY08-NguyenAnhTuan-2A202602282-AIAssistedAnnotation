# Vì sao chọn lô này?

Trong 50 dòng đứng đầu `outputs/selection_round1.csv`, chọn năm frame bạn sẽ ưu tiên nếu chỉ có
ngân sách rà năm ảnh. Ghi tên, điểm, thời điểm, thứ tự và lý do; tối thiểu một quyết định phải xét
ảnh gần trùng hoặc trường hợp model không dự đoán được box: Trong 50 dòng đứng đầu `outputs/selection_round1.csv`, chọn năm frame bạn sẽ ưu tiên nếu chỉ có 
ngân sách rà năm ảnh. Ghi tên, điểm, thời điểm, thứ tự và lý do; tối thiểu một quyết định phải xét 
ảnh gần trùng hoặc trường hợp model không dự đoán được box:

1. `frame_0182.jpg` — score = 0.9591, t = 72.8 s, rank 1.
   Đây là frame có điểm cao nhất trong 50 dòng. U = 0.9182 và A = 1.0000, cho thấy
   mô hình có độ bất định cao và nhiều box mơ hồ. Frame này cũng nằm trong lô 12 ảnh
   được chọn.

2. `frame_0369.jpg` — score = 0.9324, t = 147.6 s, rank 2.
   Có U = 0.9315 và A = 0.8889, đồng thời D = 1.0. Đây là một frame có mức
   bất định cao và nhiều vùng cần xem xét, nên ưu tiên rà nhãn.

3. `frame_0380.jpg` — score = 0.9170, t = 152.0 s, rank 3.
   U = 0.9340 là cao nhất trong ba frame đầu, A = 0.8333 và D = 1.0.
   Đây là trường hợp có nhiều tín hiệu bất định nên có giá trị để kiểm tra.

4. `frame_0326.jpg` — score = 0.9155, t = 130.4 s, rank 4.
   U = 0.9310 và A = 0.8333 đều cao. Frame có 39 box và được model đưa vào
   lô 12 ảnh cần rà.

5. `frame_0331.jpg` — score = 0.9154, t = 132.4 s, rank 5.
   A = 1.0000 và U = 0.8308. Đây là frame có mức mơ hồ rất cao và cũng nằm
   trong lô 12 ảnh được chọn.

Ba frame thuộc lô 12 ảnh model chọn và bằng chứng trong CSV/ảnh contact sheet: - `frame_0182.jpg`: rank 1, score 0.9591, selected=True.
- `frame_0369.jpg`: rank 2, score 0.9324, selected=True.
- `frame_0380.jpg`: rank 3, score 0.9170, selected=True.

Các frame trên đều có D = 1.0 và điểm tổng hợp cao. Contact sheet
`outputs/selection_round1.jpg` được dùng để xem trực quan các frame được chọn
cùng với các frame lân cận trong danh sách.

Một frame có điểm cao nhưng không chọn hoặc một frame có điểm thấp vẫn nên xem, và lý do: `frame_0372.jpg` — rank 6, score 0.9101 nhưng `selected=False`.
Điểm của frame này vẫn rất cao, với U = 0.9202 và A = 0.8333, nhưng nó không
được đưa vào lô 12 ảnh. Đây là ví dụ cho thấy điểm score không phải tiêu chí
duy nhất để quyết định chọn; cơ chế chọn lô còn phải xét sự đa dạng và tránh
chọn các frame quá gần nhau.

Điều phép chọn này chưa chứng minh về chất lượng mô hình: Điểm selection chỉ là điểm ưu tiên để chọn dữ liệu cần rà nhãn, được tính từ
các thành phần bất định, mơ hồ và đa dạng/khoảng cách thời gian. Vì vậy,
score cao cho biết frame đáng được xem xét, nhưng không chứng minh rằng việc
gán nhãn frame đó chắc chắn sẽ làm mô hình tốt lên. Muốn đánh giá chất lượng
mô hình cần dựa vào kết quả đánh giá trên cùng tập test, chẳng hạn AP50,
precision và recall, sau khi huấn luyện.
