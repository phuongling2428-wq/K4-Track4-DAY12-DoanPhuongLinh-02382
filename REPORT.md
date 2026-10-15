# REPORT — Lab Day 12

| | |
|---|---|
| Họ và tên | Đoàn Phương Linh |
| MSSV (đúng bằng `--student-id` bạn đã chạy) | 2A202602382 |
| GitHub | phuongling2428-wq |
| Seed chia dữ liệu / seed mô hình (xem `evidence/closedloop.json`, mục `student`) | 4801 / 4046 |
| Số seed huấn luyện bạn dùng (`--seeds`) | 5 (mặc định) |

## Kết quả cuối

Số lấy từ `evidence/closedloop.json`, `openloop.json`, `gate.json`. Với mô hình học: L2 là trung bình 5 seed; va chạm là **dải qua 5 seed** (min..max, trung bình trong ngoặc); trên 2.883 quỹ đạo giữ lại.

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity | 1,259 | 21,0 (605/2883) | CHẶN (R2, R3, R4, R6) |
| IDM | 1,526 | 0,10 (3/2883) | (baseline) |
| EgoMLP | 0,771 | 12,0 .. 34,6 (tb 23,1) | CHẶN (R2–R6) |
| PercMLP | 0,631 | 0,4 .. 2,5 (tb 1,03) | CHẶN (R2–R6) |
| PercMLP+ (sau khi sửa) | 0,820 | 0,2 .. 0,3 (tb 0,28) | QUA (cả 7 quy tắc) |

## Câu 1 · Dữ liệu

Ba vấn đề trong `data_audit.json`. (1) Cột `Acc_LV` không dùng được: tương quan với d(Speed_LV)/dt chỉ 0,045, trong khi cột gia tốc xe theo sau là 0,929. Mình dẫn gia tốc xe dẫn từ `Speed_LV` đã làm mượt để khai thác ca phanh. (2) Có 383 quỹ đạo trùng hệt nhau (9.993 thô → 9.610 sau khi loại). (3) Các bản trùng nằm ở id cách xa nhau, nên chia ngẫu nhiên sẽ để cùng một quỹ đạo nằm ở cả train và test, rò rỉ làm điểm đẹp giả. Chia theo quỹ đạo (không theo khung hình) vì các khung liên tiếp của một quỹ đạo gần như giống nhau; chia theo khung sẽ đưa gần-trùng lặp vào test.

## Câu 2 · Phát lại

Bản refactor float32 khác bit ở 600/600 đoạn nhưng vượt dung sai (atol = rtol = 1e-3) ở 0 đoạn; sai khác lớn nhất 1,2e-4. Phép cộng float không có tính kết hợp (ví dụ `(0.1+0.2)+0.3 = 0.6000000000000001` còn `0.1+(0.2+0.3) = 0.6`), nên so bằng bit là phép thử sai; dung sai phải nhỏ hơn nhiều so với mức ảnh hưởng đến hành vi. Chạy lại cùng mã: 0 đoạn khác. Thay đổi AEB (phanh −6 m/s² khi TTC < 1,5 s) làm lệch 8/600 đoạn (60, 70, 185, 220, 351, 498, 553, 569), sai khác lớn nhất 1,98 m/s². Mở đoạn 60: lệch ở 3 khung (75–77), TTC 1,48 → 1,46 s, lệnh cũ −4,38 → mới −6,0; đoạn 70 chỉ lệch khung 82 (TTC 1,47 s). Lệch đúng ở khung có TTC dưới ngưỡng 1,5 s, còn mọi khung khác AEB không chạm tới. Phát nhanh nhất cho 0 khung rớt, 0 khung trễ; phát theo thời gian thật trên cùng 59.713 khung làm rớt 578 khung và trễ 583 khung (spike 140 ms vượt hạn 100 ms), khiến 376 đoạn vượt dung sai, sai khác lớn nhất 109,9. Phát nhanh nhất không có hạn chót nên khung chậm vô hình.

## Câu 3 · Kịch bản

Lưới: cruise 10 m/s, khoảng cách 2,76 .. 59,3 m, tiến gần 0 .. 10 m/s, 2.700 ô. Ô không ai tránh được: 0,96 % khi trễ 0 s và 5,15 % khi trễ 0,6 s (tăng khoảng 5,3 lần). Vùng đâm của IDM trùng đúng vùng bất khả thi ở cả hai độ trễ (0 ô lệch xa biên): IDM đâm 0,96 % và 5,15 % nhưng phần "thất bại của bộ điều khiển" (attributable) bằng 0. Ngược lại constant-velocity đâm 73,4 % lưới, trong đó 68,2 % là lỗi của nó (trễ 0,6 s). Biên giúp phân biệt "ai cũng đâm" với "bộ điều khiển đâm". Log chưa phủ vùng khó: người lái phanh mạnh nhất cỡ 3,62 m/s² (`human_brake_limit`), trong khi 5,2 % lưới đòi phanh mạnh hơn mức đó; mô hình học từ log chưa từng thấy vùng này. Khai thác được 614 ca phanh (6,4 % quỹ đạo), phanh đỉnh trung vị −2,47 và p99 −3,92 m/s².

## Câu 4 · Vòng hở và vòng kín

| | L2 (m) | hạng hở | Va chạm (%) | hạng kín |
|---|---|---|---|---|
| constant-velocity | 1,259 | 3 | 21,0 | 3 |
| IDM | 1,526 | 4 | 0,10 | 1 |
| EgoMLP | 0,771 | 2 | 23,1 (tb 5 seed) | 4 |
| PercMLP | 0,631 | 1 | 1,03 (tb 5 seed) | 2 |

Bốn cặp đổi thứ tự: (const-vel, IDM), (const-vel, EgoMLP), (IDM, EgoMLP), (IDM, PercMLP). IDM có L2 tệ nhất (1,53 m) vì nó dự báo "xe dẫn giữ tốc độ" còn người lái thì không làm đúng theo luật ấy; L2 đo độ giống người lái, không đo độ an toàn. IDM phản ứng với khoảng cách và tốc độ tiến gần nên vòng kín chỉ đâm 3/2883. EgoMLP có L2 tốt hơn constant-velocity (0,77 so với 1,26) vì học được xu hướng tốc độ trung bình của người lái, nhưng nó chỉ nhìn tốc độ của chính nó, không thấy khoảng cách hay xe dẫn, nên không có thông tin để phanh khi xe trước phanh: trên lát cắt phanh nó đâm 46,8 % (seed đầu), gần như constant-velocity (53,8 %). Sai số nhỏ cộng dồn khi tự lái vòng kín.

## Câu 5 · Một lần chạy chưa đủ

EgoMLP dao động từ 12,0 % đến 34,6 % qua 5 seed (gấp khoảng 2,9 lần; tb 23,1 %). Wilson cho seed xấu nhất (4046) [32,9 %, 36,4 %] và seed tốt nhất (4050) [10,9 %, 13,2 %]: hai khoảng không chồng nhau, nên khác biệt giữa các seed là thật, không phải nhiễu lấy mẫu; khoảng tin cậy chỉ phản ánh số quỹ đạo chứ không phản ánh sự khác nhau giữa các lần huấn luyện. Nếu chỉ chạy một seed, mình có thể báo 12 % (nghe "đỡ hơn constant-velocity 21 %") hoặc 34,6 % (tệ hơn nhiều), tuỳ may rủi: kết luận ngược nhau từ cùng một mã, nên phải báo dải qua seed. Cũng vì thế R5 (dải ≤ 1 điểm) chặn EgoMLP (12,0..34,6 %) và PercMLP (0,4..2,5 %).

## Câu 6 · Tác nhân phản ứng

`T` hiệu chỉnh = 1,4 s (W1 = 0,348, so với xe sau không bao giờ phản ứng 0,888; thời khoảng trung vị của người lái 1,86 s). Phần diện tích lưới (trễ 0,6 s):

| | phát lại: trước / sau / any | phản ứng: trước / sau / any |
|---|---|---|
| const-vel | 73,4 / 0 / 73,4 | 73,4 / 0 / 73,4 |
| IDM | 5,1 / 64,9 / 70,1 | 5,1 / 0 / 5,1 |
| EgoMLP | 73,3 / 0 / 73,3 | 73,3 / 0 / 73,3 |
| PercMLP | 16,3 / 60,8 / 77,1 | 16,4 / 0 / 16,4 |
| PercMLP+ | 5,2 / 64,6 / 69,7 | 5,3 / 0 / 5,3 |

Xe sau phát lại làm đúng theo log, không biết xe thử vừa phanh gấp, nên đâm vào xe thử: đó là "va chạm giả" vì lỗi nằm ở xe sau không phản ứng, không phải của bộ điều khiển. IDM phanh tốt nên bị tính 64,9 % va chạm phía sau; constant-velocity không phanh nên không bị đâm từ phía sau (0 %) dù bị 73,4 % va chạm phía trước. Khi xe sau phản ứng, IDM từ 70,1 % xuống 5,1 %, còn constant-velocity vẫn 73,4 %. Loại tác nhân làm đổi kết luận: với xe sau phát lại, IDM (70,1) gần như ngang constant-velocity (73,4), còn thực tế hơn rất xa.

## Câu 7 · Cổng phát hành và sửa

PercMLP rớt R2 (cận trên 3,2 % > 0,1 % + 1 điểm), R3 (lát cắt phanh 8,6 %, cận trên 13,5 %), R4 (11,2 % lưới là lỗi của nó so với 0,0 % của IDM), R5 (0,4..2,5 %) và R6 (16,4 % so với 5,1 %). Hai nguyên nhân gốc: (a) **kịch bản log không chứa**: cắt vào với tốc độ tiến gần lớn đòi phanh mạnh hơn 3,62 m/s² của người lái, mô hình chưa thấy nên đâm ở 11,2 % lưới (R4) và 16,4 % ở thế giới phản ứng (R6); trên cruise 20 m/s (chưa từng thấy) PercMLP đâm 15..48 % lưới. (b) **sai số tích luỹ** khi tự lái (covariate shift): L2 tốt (0,63 m) nhưng vòng kín dao động 0,4..2,5 % (R5) và 8,6 % trên lát cắt phanh (R3). Sửa: (1) thêm 20.000 kịch bản cắt vào mô phỏng gắn nhãn bằng IDM, bù vùng log không có; (2) DAgger 2 vòng, cho mô hình tự lái rồi gắn nhãn IDM tại các trạng thái nó tự đi vào, bù sai số tích luỹ. Sau sửa PercMLP+ qua cả 7 quy tắc: va chạm 0,2..0,3 %, lưới 0,1 %, thế giới phản ứng 5,3 % (IDM 5,1 %), và trên cruise 20 m/s chỉ còn 0,1..2,9 % lưới. Nhưng PercMLP+ **không được chứng minh là tốt hơn IDM**, chỉ là **không kém hơn** trong biên đã đặt: IDM va chạm 0,10 %, thấp hơn trung bình PercMLP+ 0,28 %, và PercMLP+ học nhãn từ chính IDM nên không có lý do vượt thầy. L2 vòng hở của PercMLP+ còn xấu đi (0,631 → 0,820 m) trong khi vòng kín tốt lên rất nhiều, càng cho thấy điểm vòng hở không dự báo độ an toàn. Mình không đề xuất thay IDM ngay: nên thử thêm nhiều tốc độ cruise, nhiễu cảm biến, chạy shadow mode trên xe thật, và chỉ phát hành nếu có lợi ích khác mà IDM không có.

## Câu 8 · Bài lab này không chứng minh được gì

(1) **Xe dẫn trong log không phản ứng với xe thử**: log chỉ phát lại, không có tương tác thật; câu 6 cho thấy kết luận đổi từ 70,1 % xuống 5,1 % chỉ vì đổi tác nhân phía sau. Cần tác nhân có hành vi đa dạng (và ta chưa biết tác nhân nào "đúng"). (2) **Thế giới mô phỏng và lưới là giả định của ta**: một chiều dọc, vật lý đơn giản, trễ 0,6 s cố định, phanh tối đa cố định, cruise 10 m/s; PercMLP đâm 15..48 % ở cruise 20 m/s mà lưới chính không có. Cần đo trễ và giới hạn phanh thật. (3) **Dữ liệu và phủ kịch bản**: một bộ dữ liệu (Argoverse 2, 7–11 s mỗi cặp), người lái trong log phanh không quá 3,62 m/s² nên không dạy được ca khẩn cấp; 5,2 % lưới nằm ngoài. Cần thu thêm ca hiếm và cảm biến nhiễu. (4) **Cỡ mẫu**: 2.883 quỹ đạo, 186 ca phanh và 5 seed; dải seed của PercMLP+ (0,2..0,3 %) hẹp nhưng vẫn ít để nói về đuôi xấu. Để thu hẹp khoảng cách với thế giới thật cần shadow mode / hardware-in-the-loop, nhiều dữ liệu hơn, và tiêu chí an toàn theo từng lớp tình huống trước khi quyết định phát hành.
