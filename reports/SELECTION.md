# Vì sao chọn lô này?

Trong 50 dòng đứng đầu `outputs/selection_round1.csv`, chọn năm frame bạn sẽ ưu tiên nếu chỉ có
ngân sách rà năm ảnh. Ghi tên, điểm, thời điểm, thứ tự và lý do; tối thiểu một quyết định phải xét
ảnh gần trùng hoặc trường hợp model không dự đoán được box:

- `frame_0182.jpg` — score `0.9591`, `72.8 s`, rank `1`: điểm cao nhất; có `28` box nhưng `18` box mơ hồ, nên có giá trị rà chất lượng nhãn/model.
- `frame_0369.jpg` — score `0.9324`, `147.6 s`, rank `2`: điểm rất cao và `43` box; đại diện cảnh dày đối tượng ở cuối video.
- `frame_0331.jpg` — score `0.9154`, `132.4 s`, rank `5`: bổ sung một thời điểm/cảnh khác, với `47` box và `18` box mơ hồ.
- `frame_0099.jpg` — score `0.9063`, `39.6 s`, rank `8`: phủ cụm đầu video, khác xa các frame cuối video.
- `frame_0392.jpg` — score `0.8874`, `156.8 s`, rank `15`: điểm thấp hơn nhưng phủ thời điểm muộn; `12` box mơ hồ, giúp kiểm tra một trường hợp model ít bất định hơn.

Quyết định `frame_0369.jpg` cũng xét tính gần trùng: `frame_0368.jpg` ở `147.2 s` (rank `9`, score `0.9003`) và `frame_0372.jpg` ở `148.8 s` (rank `6`, score `0.9101`) nằm ngay cùng cụm trên contact sheet. Chỉ rà `frame_0369.jpg` trong cụm này để dành ngân sách cho các thời điểm khác.

Ba frame thuộc lô 12 ảnh model chọn và bằng chứng trong CSV/ảnh contact sheet: `frame_0182.jpg` (rank `1`, score `0.9591`, `72.8 s`, được đánh dấu `selected=True`), `frame_0369.jpg` (rank `2`, score `0.9324`, `147.6 s`, `selected=True`) và `frame_0099.jpg` (rank `8`, score `0.9063`, `39.6 s`, `selected=True`). Contact sheet cho thấy chúng thuộc các vị trí/thời điểm khác nhau; CSV xác nhận cả ba nằm trong lô 12 ảnh được chọn.

Một frame có điểm cao nhưng không chọn hoặc một frame có điểm thấp vẫn nên xem, và lý do: `frame_0372.jpg` có score `0.9101`, rank `6` nhưng `selected=False`; nó ở `148.8 s`, sát cụm `frame_0368.jpg`/`frame_0369.jpg` trên contact sheet, nên bỏ qua để giảm ảnh gần trùng. Ngược lại, `frame_0392.jpg` có score chỉ `0.8874` (rank `15`) nhưng vẫn được chọn vì nằm ở `156.8 s`, mở rộng độ phủ thời gian.

Phép chọn này chưa chứng minh về chất lượng mô hình: đây chỉ là chiến lược ưu tiên ảnh để rà soát, dựa trên score, độ bất định, số box và độ phủ thời gian. Nó không thay thế nhãn chuẩn hoặc đánh giá precision/recall, IoU, lỗi bỏ sót và false positive trên một tập kiểm tra độc lập; score cao cũng không đảm bảo dự đoán đúng.
