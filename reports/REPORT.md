# Báo cáo Lab Ngày 08: Học chủ động cho bộ phát hiện xe

Họ và tên: Nguyễn Văn Thịnh

Công cụ gán nhãn đã dùng: CVAT Docker trên máy cá nhân

Sao chép file này thành `reports/REPORT.md` rồi hoàn thiện các mục bên dưới. Mọi con số phải truy được
từ `reports/rounds_table.md`, `outputs/selection_round1.csv`, `outputs/metrics_round*.json` hoặc
`outputs/round*_diff.md`. Không coi nhãn test do mô hình tạo là chân lý tuyệt đối.

## 1. Dữ liệu và cách chia tập

Tại sao tập chưa gán nhãn (pool) và tập kiểm thử (test set) được chia theo trục thời gian, có vùng
đệm ở giữa, thay vì chia ngẫu nhiên? Nếu chia ngẫu nhiên, số đo trên tập kiểm thử sẽ bị lệch theo
hướng nào, và vì sao?

Pool và test được tách theo trục thời gian để hạn chế việc các khung hình gần như giống nhau xuất hiện ở cả hai tập. Vùng đệm ở giữa làm giảm rò rỉ theo thời gian: mô hình không được học một cảnh gần sát rồi được chấm trên một ảnh gần như cùng cảnh. Nếu chia ngẫu nhiên, các frame liên tiếp có thể lọt vào cả train/pool và test, khiến số đo test bị lạc quan và không phản ánh khả năng tổng quát sang thời điểm/cảnh mới. Ngược lại, chia theo thời gian khó hơn nhưng cho ước lượng thực tế hơn về việc mô hình gặp cảnh chưa thấy.

## 2. Mô hình khởi đầu lạnh (cold start)

Chép dòng vòng 0 từ `rounds_table.md`. Dựa vào `outputs/compare_round0.jpg`, cho biết mô hình khởi
đầu lạnh không khớp nhãn tham chiếu ở những loại xe nào. Độ phủ (recall) theo kích thước xe cho
thấy điều gì? Một trường hợp nào cần người rà lại nhãn tham chiếu trước khi kết luận mô hình sai?

| vòng | model | ảnh train | box train | AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |

Ở cold start, các dự đoán không khớp chủ yếu là xe nhỏ/ở xa, xe bị che hoặc bị nhoè; các xe trung bình và lớn dễ tìm hơn nhưng vẫn còn bỏ sót. Recall theo kích thước là `0.182` (small), `0.547` (medium) và `0.561` (large), cho thấy kích thước nhỏ là điểm yếu rõ rệt, còn recall medium/large chỉ khoảng một nửa. Một trường hợp cần rà lại nhãn tham chiếu trước khi kết luận mô hình sai là xe tối ở góc dưới trái trong `frame_0099.jpg`: đèn pha sáng và vệt phản chiếu có thể làm ranh giới thân xe khó xác định. Blind scan ghi nhận đúng vị trí này; cần áp dụng guideline, không tính vệt sáng trên mặt đường, rồi mới đối chiếu box.

## 3. Chiến lược chọn mẫu

Giải thích bằng lời công thức `score = W_U·U + W_A·A + W_D·D` và vai trò của `MIN_GAP_S`.
Dẫn ba frame trong `reports/SELECTION.md` và một frame khác để chứng minh cách bạn cân nhắc
độ bất định, ảnh gần trùng và công gán nhãn. Điểm bất định có chứng minh ảnh đó sẽ cải thiện
mô hình không? Vì sao?

Điểm được tính bằng tổng có trọng số `score = W_U·U + W_A·A + W_D·D`: `U` là độ bất định, `A` là mức mơ hồ/khó gán nhãn và `D` là thành phần ưu tiên độ đa dạng; các trọng số `W_U`, `W_A`, `W_D` điều chỉnh tầm quan trọng của từng tín hiệu. `MIN_GAP_S` là khoảng cách thời gian tối thiểu giữa hai frame được chọn, giúp không lấy nhiều ảnh gần như cùng một cảnh và dành ngân sách cho các cảnh khác.

Ví dụ, `frame_0182.jpg` (rank 1, score `0.9591`, `72.8 s`) được ưu tiên vì điểm cao và có `18` box mơ hồ trên `28` box, nên có công rà nhãn đáng kể. `frame_0369.jpg` (rank 2, `0.9324`, `147.6 s`) có `43` box và đại diện cảnh dày đối tượng ở cuối video. `frame_0099.jpg` (rank 8, `0.9063`, `39.6 s`) giúp phủ cụm đầu video. Để xét ảnh gần trùng, `frame_0372.jpg` (rank 6, `0.9101`, `148.8 s`) không được chọn dù điểm cao vì nằm sát `frame_0368.jpg`/`frame_0369.jpg`; chọn thêm ảnh trong cụm đó sẽ tốn ngân sách cho thông tin tương tự. Điểm bất định chỉ là tín hiệu để ưu tiên rà soát, không bảo đảm frame đó sẽ cải thiện mô hình: ảnh có thể khó gán, gần trùng, hoặc chứa lỗi hệ thống/nhãn.

## 4. Các vòng học chủ động (active learning)

Chép bảng từ `rounds_table.md`. Với mỗi vòng, trình bày:

- mức độ bạn đã sửa nhãn gợi ý (số box giữ nguyên, chỉnh sửa, xoá, thêm mới, lấy từ
  `outputs/round*_diff.md`);
- AP50 thay đổi bao nhiêu so với khởi đầu lạnh và so với vòng trước;
- nhóm xe nào tốt lên hoặc xấu đi theo số đo trên cùng tập test.

Dựa vào các ảnh `compare_round*.jpg`, chỉ ra một ca kết quả đổi sau fine-tune (tốt hơn hoặc xấu
đi), cùng lý do có thể kiểm. Dùng `BLIND_SCAN.md`, `REVIEW_LOG.csv` và `round1_diff.md` phân biệt
quan sát độc lập, lỗi pre-label đã sửa và kết quả mô hình sau train. Mô tả một ca khó theo guideline.

| vòng | model | ảnh train | box train | AP50 | Δ AP50 so cold start | P@0.25 | R@0.25 | F1 | R small | R medium | R large |
| ---: | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 0 | yolov8n cold start (COCO car+bus+truck) | 0 | 0 | 0.771 | — | 0.925 | 0.489 | 0.640 | 0.182 | 0.547 | 0.561 |
| 1 | yolov8n fine-tune vong 1..1 | 12 | 197 | 0.315 | -0.457 | 1.000 | 0.084 | 0.156 | 0.000 | 0.064 | 0.366 |

Ở vòng 1, 12 ảnh có 169 box model đề xuất và sau sửa thành 197 box: giữ nguyên `149`, chỉnh sửa `7`, xoá `13`, thêm mới `41`, accept rate `88%`. AP50 là `0.315`, giảm `0.457` theo bảng làm tròn, chính xác từ `0.7714` xuống `0.3147` là giảm `0.4567`; so với vòng trước cũng giảm `0.4567`. Precision tăng từ `0.925` lên `1.000`, nhưng recall giảm từ `0.489` xuống `0.084`, nên mô hình gần như chỉ giữ các dự đoán rất chắc và bỏ sót nhiều xe. Recall small giảm `0.182` xuống `0.000` (giảm `0.1818`), medium `0.547` xuống `0.064` (giảm `0.4831`), large `0.561` xuống `0.366` (giảm `0.1951`); cả ba nhóm đều xấu đi, medium giảm mạnh nhất.

Một thay đổi dễ kiểm trong ảnh so sánh là `frame_0099.jpg`: cold start còn bắt được nhiều xe nhìn thấy, trong khi sau fine-tune overlay thưa hơn và bỏ sót phần lớn xe nhỏ/xe xa. Nguyên nhân có thể là chỉ có 12 ảnh train, phân bố cảnh hẹp và nhãn sửa không đủ đại diện; cũng cần kiểm tra ngưỡng confidence/cách xuất model vì precision `1.000` đi cùng recall rất thấp.

Ba loại bằng chứng cần tách riêng. `BLIND_SCAN.md` là quan sát độc lập trước khi xem pre-label: người rà ước lượng 26 xe trong `frame_0099.jpg` và đánh dấu hai vị trí dễ bị bỏ sót. `REVIEW_LOG.csv` ghi lỗi pre-label đã sửa, gồm thêm hai xe bị bỏ sót, sửa hai box bị kéo vào vệt sáng và xoá box nhầm trên phản chiếu. `round1_diff.md` là thống kê thay đổi nhãn của 12 ảnh, không phải bằng chứng rằng model sau train đã dự đoán đúng trên test.

Ca khó theo guideline là xe tối, nhoè chuyển động và chỉ nổi bật bằng đèn pha: vẫn vẽ box ôm phần thân xe đoán được, tính cả gương/đèn, nhưng không tính vệt sáng phản chiếu trên mặt đường. Nếu xe bị che hoặc cắt mép thì chỉ vẽ phần nhìn thấy; hai xe sát nhau vẫn phải có hai box riêng. Xe cực xa có box cao dưới khoảng 16 px có thể bỏ qua vì bị loại khỏi chấm điểm.

## 5. Kết luận và giới hạn

Kết quả vòng này so với cold start ra sao? Vì sao bạn dừng hoặc tiếp tục? Đề xuất hai ca còn yếu
hoặc bất định cho vòng sau, kèm chi phí rà nhãn và nguy cơ ảnh gần trùng. Tập kiểm thử chỉ 20 ảnh,
có luật bỏ qua xe quá nhỏ và nhãn tham chiếu do mô hình tạo chưa được rà thủ công; các giới hạn đó
ảnh hưởng thế nào đến kết luận? Nếu AP50 giảm, bạn sẽ kiểm tra điều gì trước khi train thêm?

So với cold start, vòng 1 không đạt mục tiêu: AP50 giảm từ `0.7714` xuống `0.3147`, recall giảm từ `0.4888` xuống `0.0844`, dù precision tăng từ `0.9249` lên `1.000`. Vì vậy nên dừng train thêm ngay lúc này và kiểm tra dữ liệu, cách đóng gói nhãn, phân bố lớp/cảnh, confidence và kết quả dự đoán trước khi quyết định vòng 2.

Hai ca nên ưu tiên rà ở vòng sau là `frame_0099.jpg` và `frame_0331.jpg`. `frame_0099.jpg` có chi phí sửa cao trong diff (`9` thêm, `3` chỉnh, `1` xoá), lại có xe tối, đèn pha, phản chiếu và xe ở xa; nguy cơ gần trùng với `frame_0098.jpg` nên cần giữ khoảng cách thời gian. `frame_0331.jpg` có `5` thêm và `4` xoá, `18` box mơ hồ trên `47` box; đây là cảnh dày đối tượng nhưng gần cụm `frame_0326.jpg`/`frame_0330.jpg`, nên chỉ chọn nếu khoảng cách `MIN_GAP_S` cho phép. Cả hai có công rà nhãn cao nhưng có thể bổ sung các lỗi bỏ sót khác loại thay vì chọn thêm nhiều frame cùng cảnh.

Test chỉ có 20 ảnh và 403 box tham chiếu, trong đó 14 box rất nhỏ bị bỏ qua, nên sai khác trên vài ảnh có thể làm AP50 và recall biến động mạnh. Quy tắc bỏ qua xe dưới khoảng 16 px khiến kết luận về xe nhỏ không bao quát toàn bộ xe nhỏ trong thực tế. Ngoài ra, nhãn tham chiếu do mô hình tạo chưa được rà thủ công tuyệt đối, nên metric có thể chứa lỗi nhãn và không nên xem là chân lý cuối cùng.

Nếu AP50 giảm, trước khi train thêm cần: kiểm tra diff để chắc không có hàng loạt box bị xoá hoặc thêm sai; mở ảnh và overlay nhãn train để tìm box lệch, box gộp/tách sai và phản chiếu bị gán nhãn; kiểm tra class id, định dạng YOLO, kích thước ảnh và việc train đúng 12 ảnh/197 box; sau đó xem confusion theo kích thước và thử lại confidence/evaluation trên cùng test. Chỉ khi dữ liệu và pipeline hợp lệ mới cân nhắc thêm ảnh, điều chỉnh chiến lược chọn hoặc train vòng tiếp theo.
