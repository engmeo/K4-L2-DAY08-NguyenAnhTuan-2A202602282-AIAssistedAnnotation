# Báo cáo Lab Ngày 08: Học chủ động cho bộ phát hiện xe

Họ và tên: Nguyễn Anh Tuấn

Công cụ gán nhãn đã dùng: CVAT

Sao chép file này thành `reports/REPORT.md` rồi điền vào các chỗ ĐIỀN. Mọi con số phải truy được
từ `reports/rounds_table.md`, `outputs/selection_round1.csv`, `outputs/metrics_round*.json` hoặc
`outputs/round*_diff.md`. Không coi nhãn test do mô hình tạo là chân lý tuyệt đối.

## 1. Dữ liệu và cách chia tập

Tại sao tập chưa gán nhãn (pool) và tập kiểm thử (test set) được chia theo trục thời gian, có vùng
đệm ở giữa, thay vì chia ngẫu nhiên? Nếu chia ngẫu nhiên, số đo trên tập kiểm thử sẽ bị lệch theo
hướng nào, và vì sao?

Tập chưa gán nhãn (pool) và tập kiểm thử (test set) được chia theo trục thời gian và có vùng đệm ở giữa để hạn chế việc các ảnh rất giống nhau hoặc liên tiếp nhau xuất hiện ở cả hai tập. Với dữ liệu lấy từ video, các frame gần nhau thường có nội dung và vị trí xe tương tự nhau.

Nếu chia ngẫu nhiên theo từng frame, các frame gần như giống nhau có thể rơi vào cả train/pool và test. Khi đó mô hình có thể được đánh giá trên những cảnh rất gần với dữ liệu đã thấy, làm kết quả test lạc quan hơn và không phản ánh tốt khả năng tổng quát sang các đoạn thời gian khác.

Lab cố định cùng một tập 20 ảnh kiểm thử cho các vòng để việc so sánh AP50 giữa các vòng có cùng điều kiện. Các box tham chiếu cao dưới 16 pixel được bỏ qua khi chấm theo quy tắc của notebook.


## 2. Mô hình khởi đầu lạnh (cold start)

Chép dòng vòng 0 từ `rounds_table.md`. Dựa vào `outputs/compare_round0.jpg`, cho biết mô hình khởi
đầu lạnh không khớp nhãn tham chiếu ở những loại xe nào. Độ phủ (recall) theo kích thước xe cho
thấy điều gì? Một trường hợp nào cần người rà lại nhãn tham chiếu trước khi kết luận mô hình sai?

Mốc khởi đầu lạnh được tạo từ YOLOv8n pretrained trên COCO, chưa fine-tune trên dữ liệu của video. Trong bước đánh giá cold-start, notebook lấy các detection COCO thuộc car, bus và truck và quy về bài toán một lớp `car`. Điều này không có nghĩa dataset gán nhãn có ba lớp; nhãn của bài lab chỉ là `car`.

Dòng kết quả vòng 0 được lấy từ `reports/rounds_table.md` và `outputs/metrics_round0.json`.

Mô hình khởi đầu lạnh có thể không khớp nhãn tham chiếu ở các trường hợp xe nhỏ, xa hoặc khó quan sát. Độ phủ theo kích thước xe cần được đọc cùng với `recall_by_size` trong metrics để phân biệt nhóm kích thước mà mô hình bỏ sót nhiều hơn.

Một trường hợp cần người rà lại nhãn tham chiếu trước khi kết luận mô hình sai là xe rất nhỏ ở gần đường chân trời. Notebook quy định các box tham chiếu cao dưới 16 pixel được bỏ qua khi đánh giá, vì vậy không nên coi mọi trường hợp mô hình không phát hiện được xe nhỏ là lỗi của mô hình.

## 3. Chiến lược chọn mẫu

Giải thích bằng lời công thức `score = W_U·U + W_A·A + W_D·D` và vai trò của `MIN_GAP_S`.
Dẫn ba frame trong `reports/SELECTION.md` và một frame khác để chứng minh cách bạn cân nhắc
độ bất định, ảnh gần trùng và công gán nhãn. Điểm bất định có chứng minh ảnh đó sẽ cải thiện
mô hình không? Vì sao?

Điểm chọn mẫu được xây dựng theo:

`score = W_U·U + W_A·A + W_D·D`

Trong đó:

- `U` là độ bất định của mô hình, được tính từ các box khó/không chắc chắn.
- `A` thể hiện các trường hợp box có khả năng mơ hồ hoặc cần xem xét.
- `D` khuyến khích sự đa dạng theo thời gian để tránh chọn nhiều frame gần như giống nhau.
- `W_U`, `W_A`, `W_D` là trọng số của từng thành phần.

`MIN_GAP_S` được dùng để tạo khoảng cách tối thiểu giữa các frame được chọn. Mục đích là tránh việc cả batch gồm nhiều frame liên tiếp gần như trùng nhau, từ đó dành ngân sách gán nhãn cho các tình huống đa dạng hơn.

Trong vòng 1, mô hình đã chọn 12 ảnh để đưa vào quy trình gán nhãn. Khi rà các ảnh này, cần cân nhắc đồng thời điểm bất định, khả năng ảnh gần trùng nhau và chi phí chỉnh sửa box.

Điểm bất định không chứng minh rằng một ảnh chắc chắn sẽ cải thiện mô hình. Nó chỉ là tín hiệu để ưu tiên ảnh mà mô hình hiện tại chưa chắc chắn hoặc có khả năng chứa lỗi. Hiệu quả thật sự chỉ có thể kiểm tra sau khi nhãn được người sửa và mô hình được fine-tune rồi đánh giá lại trên cùng tập test.
## 4. Các vòng học chủ động (active learning)

Chép bảng từ `rounds_table.md`. Với mỗi vòng, trình bày:

- mức độ bạn đã sửa nhãn gợi ý (số box giữ nguyên, chỉnh sửa, xoá, thêm mới, lấy từ
  `outputs/round*_diff.md`);
- AP50 thay đổi bao nhiêu so với khởi đầu lạnh và so với vòng trước;
- nhóm xe nào tốt lên hoặc xấu đi theo số đo trên cùng tập test.

Dựa vào các ảnh `compare_round*.jpg`, chỉ ra một ca kết quả đổi sau fine-tune (tốt hơn hoặc xấu
đi), cùng lý do có thể kiểm. Dùng `BLIND_SCAN.md`, `REVIEW_LOG.csv` và `round1_diff.md` phân biệt
quan sát độc lập, lỗi pre-label đã sửa và kết quả mô hình sau train. Mô tả một ca khó theo guideline.

Quy trình của lab bắt đầu từ cold start, sau đó người dùng sửa các nhãn gợi ý, đóng gói chúng thành `labels/round1/`, rồi fine-tune lại YOLOv8n trên toàn bộ nhãn đã sửa của vòng 1.

### Vòng 0

Đây là cold start, chưa có dữ liệu train từ video.

AP50, precision và recall của vòng 0 được lấy từ `metrics_round0.json`/`rounds_table.md`. Đây là mốc dùng để tính mức thay đổi AP50 của vòng 1.

### Vòng 1

Có 12 ảnh được gán nhãn và đóng gói vào `labels/round1/`.

Kết quả sau khi sửa nhãn:

- 12 ảnh.
- 354 box sau khi hoàn thiện nhãn.
- 169 box do model đề xuất.
- 112 box được giữ nguyên (`accepted`).
- 40 box được chỉnh sửa (`edited`).
- 17 box bị xoá (`deleted`).
- 202 box được thêm mới (`added`).

Như vậy, quá trình review cho thấy phần nhãn cuối cùng không chỉ là việc chấp nhận nguyên trạng pre-label của AI; người gán nhãn phải bổ sung và chỉnh sửa đáng kể.

Các trường hợp trong `REVIEW_LOG.csv` gồm:

- `frame_0182.jpg`: thêm box cho xe ở góc dưới phải, bị cắt ở mép ảnh.
- `frame_0227.jpg`: thêm box cho xe sát mép phải ảnh.
- `frame_0270.jpg`: thêm box cho xe ở vùng dưới bên phải.
- `frame_0187.jpg`: giữ nguyên box AI vì phù hợp với phần thân xe nhìn thấy.

Kết quả đánh giá cuối vòng 1 trên cùng tập 20 ảnh test là:

- AP50 = 0.483
- Precision = 1.000
- Recall = 0.119
- F1 = 0.213

Kết quả này phải được phân biệt với bảng `mAP50` mà Ultralytics in trong quá trình train. Bảng đó được tính trên ảnh huấn luyện và không phải metric test của lab.

Mức thay đổi AP50 so với cold start được tính bằng:

`AP50 vòng 1 - AP50 vòng 0`

và mức thay đổi so với vòng trước cũng được lấy trực tiếp từ `rounds_table.md`.

Qua `compare_round1.jpg`, kết quả sau fine-tune có thể thay đổi ở một số frame: một số xe được phát hiện hoặc định vị khác so với cold start, nhưng cũng có thể xuất hiện trường hợp phát hiện kém đi. Khi xem một trường hợp thay đổi, cần đối chiếu với nhãn tham chiếu và guideline thay vì mặc định rằng prediction của model là đúng.

Cần phân biệt ba nguồn bằng chứng:

1. **AI initial labels**: các box do model đề xuất trước khi người gán nhãn sửa.
2. **Corrected labels**: các box sau khi người gán nhãn kiểm tra và sửa trong CVAT.
3. **Fine-tuned model**: prediction của model sau khi được train trên các nhãn đã sửa.

`REVIEW_LOG.csv` ghi lại các trường hợp con người đã xử lý trong quá trình sửa pre-label. Ví dụ, ở `frame_0182.jpg`, `frame_0227.jpg` và `frame_0270.jpg`, box được thêm vì AI bỏ sót xe nhưng phần thân xe vẫn nhìn thấy và thuộc phạm vi cần gán nhãn.
Một ca khó theo guideline là xe bị cắt ở mép ảnh. Khi xe vẫn còn phần thân nhìn thấy trong ảnh thì vẫn cần gán box cho phần xe nằm trong ảnh; không nên xoá chỉ vì xe bị crop ở mép.

## 5. Kết luận và giới hạn

## 5. Kết luận và giới hạn

Kết quả vòng này so với cold start ra sao? Vì sao bạn dừng hoặc tiếp tục? Đề xuất hai ca còn yếu
hoặc bất định cho vòng sau, kèm chi phí rà nhãn và nguy cơ ảnh gần trùng. Tập kiểm thử chỉ 20 ảnh,
có luật bỏ qua xe quá nhỏ và nhãn tham chiếu do mô hình tạo chưa được rà thủ công; các giới hạn đó
ảnh hưởng thế nào đến kết luận? Nếu AP50 giảm, bạn sẽ kiểm tra điều gì trước khi train thêm?

Sau vòng 1, mô hình đã được fine-tune trên 12 ảnh với tổng cộng 354 box đã được con người hoàn thiện. Kết quả test vòng 1 là AP50 = 0.483, Precision = 1.000 và Recall = 0.119.

Kết quả này cần được so sánh với cold start bằng `metrics_round0.json` và `rounds_table.md`. Không nên chỉ dựa vào mAP50 của Ultralytics trên tập train để kết luận mô hình tốt hơn.

Nếu dừng ở vòng này, lý do có thể dựa trên chi phí gán nhãn và việc cần kiểm tra kỹ những trường hợp mô hình còn bất định. Nếu tiếp tục vòng sau, có thể ưu tiên các ảnh có điểm selection cao nhưng đồng thời tránh các frame quá gần nhau để giảm chi phí gán các ảnh gần trùng.

Hai loại ca cần ưu tiên rà lại ở vòng sau là:

1. Xe nhỏ/xa hoặc có kích thước gần ngưỡng bỏ qua 16 pixel, vì đây là nhóm dễ bỏ sót và cần kiểm tra guideline trước khi kết luận lỗi model.
2. Xe nằm sát mép ảnh hoặc bị che/cắt một phần, vì các trường hợp này đã xuất hiện trong quá trình sửa nhãn vòng 1 và có thể tiếp tục tạo ra false negative hoặc box không nhất quán.

Chi phí rà nhãn của mỗi ảnh phụ thuộc vào số xe, mức độ che khuất và số box cần sửa. Các ảnh gần nhau theo thời gian cũng có nguy cơ chứa cùng một tình huống, nên không nên chọn quá nhiều frame liên tiếp.

Tập kiểm thử chỉ có 20 ảnh nên độ ổn định của AP50 còn hạn chế. Ngoài ra, các nhãn test được dùng trong quy trình không nên được coi là chân lý tuyệt đối nếu chưa được rà thủ công đầy đủ. Vì vậy kết quả hiện tại phù hợp hơn với việc theo dõi xu hướng giữa các vòng trong cùng một protocol hơn là khẳng định chất lượng tổng quát của mô hình.

Nếu AP50 giảm sau một vòng fine-tune, trước khi train thêm cần kiểm tra:

- nhãn mới có bị sửa sai hoặc thiếu box không;
- số lượng box added/edited/deleted có bất thường không;
- các ảnh mới có quá giống nhau hoặc không đại diện cho pool không;
- prediction thay đổi ở những frame nào trong `compare_round*.jpg`;
- và pipeline đánh giá có giữ nguyên cùng tập test, cùng guideline và cùng ngưỡng hay không.
