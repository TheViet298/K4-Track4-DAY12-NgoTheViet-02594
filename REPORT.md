# REPORT — Lab Day 12

> Điền **bằng lời và số liệu của chính bạn** (lấy từ màn hình và từ `evidence/*.json`). Mỗi câu 4–8 dòng là đủ. Số không có trong bằng chứng của bạn sẽ bị trừ điểm. Báo cáo công khai trên GitHub; giảng viên đối chiếu với `evidence/` và với bài của các bạn khác.

| | |
|---|---|
| Họ và tên | `Ngô Thế Việt` |
| MSSV (đúng bằng `--student-id` bạn đã chạy) | `02594` |
| GitHub | `TheViet298` |
| Seed chia dữ liệu / seed mô hình (xem `evidence/closedloop.json`, mục `student`) | `9302 / 9360 (năm seed: 9360–9364)` |
| Số seed huấn luyện bạn dùng (`--seeds`) | `5` |

## Kết quả cuối

Điền từ `evidence/closedloop.json`, `openloop.json`, `gate.json` (cho mỗi mô hình học, lấy **dải qua các seed**).

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity | 1,248 | 19,2 | BLOCK |
| IDM | 1,509 | 0,1 | (baseline) |
| EgoMLP | 0,761 ± 0,003 | 12,4 .. 33,6 | BLOCK |
| PercMLP | 0,624 ± 0,003 | 0,8 .. 2,3 | BLOCK |
| PercMLP+ (sau khi sửa) | 0,813 | 0,2 .. 0,3 | RELEASE |

## Câu 1 · Dữ liệu
Bước 0 cho thấy cột `Acc_LV` không dùng được: tương quan với d(tốc độ)/dt chỉ 0,045, trong khi cột gia tốc xe sau đạt 0,929. Gia tốc xe dẫn vì vậy được tính lại từ `Speed_LV` đã làm mượt. Khoảng cách trong file khớp với hiệu vị trí trừ chiều dài xe (lệch tối đa 3,6e-11 m), nên cột này đáng tin.
Có 383 quỹ đạo trùng hoàn toàn bị loại, từ 9.993 xuống còn 9.610. Các bản trùng nằm ở những id cách xa nhau, nên nếu chia ngẫu nhiên thì cùng một quỹ đạo có thể rơi vào cả tập train lẫn test và làm điểm đẹp giả. Vì thế chia theo quỹ đạo (split seed 9302) và loại trùng trước khi chia.

## Câu 2 · Phát lại
Chạy lại cùng mã cho 0/600 đoạn lệch. Bản refactor float32 làm cả 600 đoạn không giống từng bit (lệch tối đa 4,0e-04) nhưng 0 đoạn vượt dung sai, nên so bằng bit là phép thử sai, phải so có dung sai.
Thêm AEB làm 10/600 đoạn lệch (ví dụ đoạn 55, 76, 152, 207, 276). Mở ra xem thì lệch đúng ở những khung AEB can thiệp, tức là hành vi thật sự đổi, không phải sai số làm tròn.
Phát nhanh nhất cho 0 khung trễ và 0 khung rơi vì không có hạn chót. Phát theo thời gian thật lộ ra 583 khung trễ và 574 khung rơi trên 59.525 khung, kéo theo 380/600 đoạn đổi lệnh. Phát nhanh nhất che đi hẳn lỗi về thời gian này.

## Câu 3 · Kịch bản
Lưới gồm 60 khoảng cách × 45 tốc độ tiến gần = 2.700 ô, tốc độ cruise 10 m/s, khoảng cách lấy từ p2 đến p98 của log. Với độ trễ phản ứng 0,6 s, 5,1% ô không ai tránh được (1,0% khi không trễ).
Độ trễ từ 0 lên 0,6 s làm vùng đâm của IDM tăng từ 1,0% lên 5,1%, cũng bằng đúng phần không thể tránh, nên IDM gần như chỉ đâm ở nơi vật lý không cho phép tránh.
Log không phủ hết vùng khó: người lái trong log phanh mạnh nhất khoảng 3,6 m/s² (phân vị 99,9), trong khi cơ cấu chấp hành làm được 9 m/s². Khoảng 5% lưới đòi phanh mạnh hơn mức đó, tức là vùng mà mô hình học từ log chưa từng thấy mẫu nào.

## Câu 4 · Vòng hở và vòng kín
| Bộ điều khiển | L2 vòng hở (m) | Hạng | Va chạm vòng kín | Hạng |
|---|---|---|---|---|
| constant-velocity | 1,248 | 3 | 19,2% | 3 |
| IDM | 1,509 | 4 | 0,1% | 1 |
| EgoMLP | 0,761 | 2 | 20,5% (trung bình 5 seed) | 4 |
| PercMLP | 0,624 | 1 | 1,5% (trung bình 5 seed) | 2 |

Bốn cặp đổi thứ tự giữa hai cách chấm: const-vel/IDM, const-vel/EgoMLP, IDM/EgoMLP, IDM/PercMLP.
IDM có L2 tệ nhất (1,509 m) mà vẫn lái tốt nhất vì L2 chỉ đo bám theo hành vi của người lái, còn IDM không bắt chước ai: nó giữ khoảng cách an toàn nên không đâm.
EgoMLP có L2 0,761 m tốt hơn const-vel (1,248 m) nhưng đâm trung bình 20,5%, vì nó chỉ thấy tốc độ xe mình, không thấy khoảng cách và xe dẫn. Sai số nhỏ cộng dồn theo vòng kín rồi đẩy xe vào trạng thái người lái chưa bao giờ gặp.

## Câu 5 · Một lần chạy chưa đủ
EgoMLP dao động mạnh giữa năm seed: 33,6%, 26,0%, 16,4%, 14,0%, 12,4%. Với seed tệ nhất (9360), 969/2.883 quỹ đạo đâm, khoảng Wilson 95% là [31,9; 35,4]%, hẹp nhưng chỉ nói về seed đó.
Nếu chỉ chạy một seed thì có thể báo 12,4% hoặc 33,6% tùy may rủi, chênh gần 3 lần. Khoảng tin cậy chỉ phản ánh số quỹ đạo test, không phản ánh độ ngẫu nhiên của huấn luyện, nên một seed dễ cho cảm giác chắc chắn giả. Vì vậy cổng R5 đòi chênh lệch giữa các seed không quá 1 điểm.

## Câu 6 · Tác nhân phản ứng
Trên lưới cut-in với độ trễ 0,6 s, tỷ lệ ô có va chạm bất kỳ (trước hoặc sau):

| Bộ điều khiển | Xe sau phát lại | Xe sau phản ứng |
|---|---|---|
| constant-velocity | 73,4% | 73,4% |
| IDM | 70,1% | 5,1% |
| EgoMLP | 72,7% | 72,7% |
| PercMLP | 80,8% | 22,8% |

T hiệu chỉnh được là 1,4 s (W1 = 0,36); tài xế không phản ứng có W1 = 0,85, nên xe sau phản ứng gần log thật hơn rõ rệt. Độ giãn cách thời gian trung vị của người lái trong log là 1,86 s.
Với xe sau chỉ phát lại, IDM bị tính 64,9% va chạm từ phía sau, dù IDM phanh hợp lý: xe sau trong log không biết IDM đang phanh nên cứ đâm vào. Đó là "va chạm giả". Khi xe sau phản ứng, va chạm phía sau về 0 và IDM chỉ còn 5,1%, đúng bằng phần vật lý không tránh được. Kết luận về IDM vì vậy đổi từ "tệ" sang "gần như tối ưu".

## Câu 7 · Cổng phát hành và sửa
PercMLP bị chặn bởi 5 quy tắc: R2 (2,3%, cận trên 2,9% so với baseline 0,1% + 1 điểm), R3 (9,9% so với 1,1% + 5 điểm), R4 (17,6% lưới so với 0,0% + 2 điểm), R5 (0,8 đến 2,3% giữa các seed) và R6 (22,8% so với 5,1%). Nguyên nhân gốc là log không có cut-in gấp, nên mô hình chưa từng gặp tình huống đó và sai ở đúng vùng này.
Cách sửa là thêm 197.110 cửa sổ từ 20.000 cut-in mô phỏng do IDM lái, rồi dùng DAgger (2 vòng): cho mô hình tự lái trên log, ghi lại các trạng thái nó đi qua và gán nhãn bằng việc IDM sẽ làm ở chính các trạng thái đó. Cách này chữa đúng chỗ thiếu dữ liệu và lỗi tích lũy.
Sau sửa, PercMLP+ qua cả 7 quy tắc (RELEASE): va chạm 0,2 đến 0,3%, lưới 0,3%, thế giới phản ứng 5,4%. Nhưng nó không tốt hơn IDM: 0,3% so với 0,1% trên log, 5,4% so với 5,1% trong thế giới phản ứng, L2 vòng hở cũng tệ hơn PercMLP (0,813 so với 0,624 m). Ở tốc độ cruise 20 m/s (huấn luyện chỉ thấy tới 16), PercMLP+ vẫn đâm 0,2 đến 2,4% lưới, còn IDM là 0,0%.
Chưa nên phát hành ngay. Trước đó cần thử thêm ở tốc độ ngoài vùng huấn luyện, thêm nhiễu cảm biến và nhiều seed hơn, rồi mới quyết định. Nếu chỉ cần hiệu năng thì IDM vẫn là lựa chọn an toàn hơn.

## Câu 8 · Bài lab này không chứng minh được gì
1. **Mô phỏng một chiều dọc, không có cảm biến thật**: không có đổi làn, người đi bộ, mưa, che khuất hay nhiễu camera. Mô hình qua cổng ở đây chưa nói gì về những tình huống đó.
2. **Xe dẫn trong log không phản ứng với xe thử**: xe dẫn chạy theo kịch bản đã ghi, không đáp lại xe mình. Thế giới phản ứng (xe sau) cũng chỉ là một mô hình IDM hiệu chỉnh với T = 1,4 s, là giả định của mình chứ không phải người thật.
3. **Dữ liệu và cổng có giới hạn**: log chỉ phanh tới khoảng 3,6 m/s² và tốc độ tới khoảng 16 m/s, trong khi PercMLP+ vẫn đâm ở 20 m/s. Ngưỡng của cổng cũng do mình chọn, nên qua cổng không đồng nghĩa với an toàn.
Để thu hẹp khoảng cách cần thử trên track kín hoặc xe thật ở tốc độ thấp, dữ liệu phủ vùng phanh gấp và tốc độ cao hơn, và kiểm thử với tác nhân do người điều khiển.
