MSSV: 202602673

# Lab 11 · Scenario Forge — báo cáo

Mỗi câu: trả lời ngắn, **trích số từ `evidence/`** (ghi tên file và trường, ví dụ `mc.json: crashes = 4`). Grader chạy lại với MSSV của bạn; số không khớp `evidence/` thì mất điểm phần D (RUBRIC), và câu dựa trên số sai khó được điểm đầy đủ. Không cần văn hay: một bảng nhỏ và hai, ba câu giải thích là đủ.

Thị trấn của bạn (`meta.json: _run.town`): A (`meta.json: _run.town = A`)

## 1. Phát lại log có trung thực không? (bước A)

Trong `replay.json`: bao nhiêu kịch bản bị bỏ vì người lái gây va chạm, bao nhiêu va chạm khi planner chạy trên log phát lại, bao nhiêu trong thế giới phản ứng, theo từng loại tiếp xúc. Giải thích vì sao phần lớn va chạm của "phát lại" là `rear`, và vì sao đó là va chạm **giả**. Chọn một kịch bản trong `replay_only` và nói chuyện gì đã xảy ra.

Số log bị bỏ: `replay.json: dropped_human_crash_logs = 0`. Tổng va chạm: `replay.json: replay.crashes = 103` khi phát lại và `replay.json: reactive.crashes = 2` trong thế giới phản ứng.

| Tiếp xúc | Phát lại | Phản ứng |
|---|---|---|
| rear | `replay.json: replay.contact.rear = 93` | Không có khóa rear |
| side | `replay.json: replay.contact.side = 10` | `replay.json: reactive.contact.side = 1` |
| front | Không có khóa front | `replay.json: reactive.contact.front = 1` |

Khóa vắng mặt nghĩa là không ghi nhận loại tiếp xúc đó. Phần lớn là rear: planner phanh sớm hơn người lái trong log nhưng xe bám sau TG vẫn chạy theo quỹ đạo đã ghi, không phanh theo planner. Va chạm này là giả do mất phản hồi tương tác.

Ví dụ `replay.json: replay_only[1].scenario = 3`, `replay.json: replay_only[1].contact = rear`: TG chạy theo log đâm đuôi planner khi planner giảm tốc sớm; trong thế giới phản ứng TG có thể giảm tốc và không va chạm. `replay.json: replay_only[1].min_ttc_reactive = 7.460629498692649` giây. Đây là diễn giải cơ chế; JSON tổng kết không ghi thời điểm phanh cụ thể.

## 2. Kịch bản bất khả thi (bước B)

Liệt kê các luật trong `is_feasible` và căn cứ vật lý của từng luật (ma sát, gia tốc, tốc độ người). Trong `hunt.json`, cuộc săn nào sinh nhiều kịch bản bất khả thi nhất, và vì sao CEM lại bị kéo về phía đó? Điều gì xảy ra với ước lượng nếu bạn **đếm** chúng là va chạm?

Các ngưỡng sau là hằng số của is_feasible theo đề bài, không phải số đo evidence:

- Đổi làn cosine rộng 3.5 m cần gia tốc ngang a_lat = (π/cutDur)² × 3.5/2 ≤ μg, để không vượt độ bám ngang của lốp.
- Phanh sau nhập làn cutBrake ≤ μg, do ma sát lốp giới hạn gia tốc hãm.
- Người đi bộ pedV ≤ 2.5 m/s; vượt ngưỡng là chạy, ngoài giả định đi bộ.
- Xe cắt vào có cutV trong (0, 22] m/s, bảo đảm xe tiến và thuộc miền tốc độ cho phép.

Nhiều mẫu bất khả thi nhất là `hunt.json: crosswalk_block.infeasible = 12`; so với `hunt.json: rear_ended.infeasible = 5`, `hunt.json: cut_in.infeasible = 2`, `hunt.json: bus_bypass.infeasible = 0`. CEM ưu tiên mẫu nguy hiểm nên dễ bị kéo về chuyển động vượt độ bám hoặc ngoài miền vật lý. Đếm chúng là va chạm sẽ thổi phồng xác suất bằng tai nạn không thể xảy ra theo giả định mô hình.

## 3. Monte Carlo cho bạn biết gì (bước C1)

`mc.json`: k, n, p̂ và khoảng Wilson 95 %. Với p̂ đó, cần bao nhiêu lần chạy để khoảng tin cậy hẹp lại còn ±20 % của p? Có kiểu thất bại nào **chưa hề xuất hiện** trong 3 000 lần chạy không? Con số đúng cần báo cho kiểu đó là gì?

Số va chạm k: `mc.json: crashes = 3`; số lượt `mc.json: n = 3000`; p̂: `mc.json: p = 0.001`. Khoảng Wilson 95 %: từ `mc.json: lo = 0.0003401410630165598` đến `mc.json: hi = 0.002936196810458527`.

Với mục tiêu nửa độ rộng tương đối ±20 %, n ≈ 1.96²(1−p)/(0.2²p) ≈ 96/p. Thay `mc.json: p = 0.001`, suy ra khoảng 96 000 lượt; dùng biểu thức chưa làm tròn được 95 943.96 lượt, làm tròn lên 95 944. Đây là phép tính từ evidence, không phải ngân sách đã chạy.
| Kiểu chưa thấy | Số lần | Cận trên Wilson cần báo |
|---|---|---|
| cut_in | `mc.json: per_mode.cut_in.k = 0` | `mc.json: per_mode.cut_in.hi = 0.0012788956648046956` |
| crosswalk_block | `mc.json: per_mode.crosswalk_block.k = 0` | `mc.json: per_mode.crosswalk_block.hi = 0.0012788956648046956` |
| rear_ended | `mc.json: per_mode.rear_ended.k = 0` | `mc.json: per_mode.rear_ended.hi = 0.0012788956648046956` |
| other | `mc.json: per_mode.other.k = 0` | `mc.json: per_mode.other.hi = 0.0012788956648046956` |

Không quan sát thấy không có nghĩa là xác suất bằng không; phải báo cận trên cho từng kiểu.

## 4. Importance sampling, ESS và độ suy biến (bước C3)

Bảng: với từng kiểu và từng seed trong `estimate.json`, ghi `p`, `hits`, `ess`, `top_share`. Ba seed có đồng ý với nhau không, và có đồng ý với Monte Carlo không? Ở đâu `top_share` lớn (> 0,5), và vì sao khi đó con số thực ra chỉ là trọng số của một mẫu? So ngân sách (`budget`) với số lần chạy Monte Carlo cần để có cùng độ chính xác. Phần `other` trong tổng lấy từ đâu, và vì sao không săn nó?

Mỗi ô trích trực tiếp file và đường dẫn trường để đối chiếu.

| Mode | Seed | p trong JSON | hits | ess | top_share |
|---|---|---|---|---|---|
| bus_bypass | `estimate.json: seeds[0].seed = 0` | `estimate.json: seeds[0].per.bus_bypass.p = 0.0006869319093288169` | `estimate.json: seeds[0].per.bus_bypass.hits = 37` | `estimate.json: seeds[0].per.bus_bypass.ess = 127.76654972827936` | `estimate.json: seeds[0].per.bus_bypass.top_share = 0.1887295589654736` |
| cut_in | `estimate.json: seeds[0].seed = 0` | `estimate.json: seeds[0].per.cut_in.p = 0.0` | `estimate.json: seeds[0].per.cut_in.hits = 0` | `estimate.json: seeds[0].per.cut_in.ess = 128.7817950887538` | `estimate.json: seeds[0].per.cut_in.top_share = 0.0` |
| crosswalk_block | `estimate.json: seeds[0].seed = 0` | `estimate.json: seeds[0].per.crosswalk_block.p = 1.653099068390009e-06` | `estimate.json: seeds[0].per.crosswalk_block.hits = 93` | `estimate.json: seeds[0].per.crosswalk_block.ess = 116.98226539207195` | `estimate.json: seeds[0].per.crosswalk_block.top_share = 0.22465869139326042` |
| rear_ended | `estimate.json: seeds[0].seed = 0` | `estimate.json: seeds[0].per.rear_ended.p = 0.0` | `estimate.json: seeds[0].per.rear_ended.hits = 0` | `estimate.json: seeds[0].per.rear_ended.ess = 118.9158800319421` | `estimate.json: seeds[0].per.rear_ended.top_share = 0.0` |
| bus_bypass | `estimate.json: seeds[1].seed = 1` | `estimate.json: seeds[1].per.bus_bypass.p = 0.000421344802583463` | `estimate.json: seeds[1].per.bus_bypass.hits = 26` | `estimate.json: seeds[1].per.bus_bypass.ess = 127.08784732092106` | `estimate.json: seeds[1].per.bus_bypass.top_share = 0.30876724792195026` |
| cut_in | `estimate.json: seeds[1].seed = 1` | `estimate.json: seeds[1].per.cut_in.p = 0.0` | `estimate.json: seeds[1].per.cut_in.hits = 0` | `estimate.json: seeds[1].per.cut_in.ess = 131.5575154106157` | `estimate.json: seeds[1].per.cut_in.top_share = 0.0` |
| crosswalk_block | `estimate.json: seeds[1].seed = 1` | `estimate.json: seeds[1].per.crosswalk_block.p = 2.483743758241341e-05` | `estimate.json: seeds[1].per.crosswalk_block.hits = 80` | `estimate.json: seeds[1].per.crosswalk_block.ess = 123.1733035299762` | `estimate.json: seeds[1].per.crosswalk_block.top_share = 0.9315809461819868` |
| rear_ended | `estimate.json: seeds[1].seed = 1` | `estimate.json: seeds[1].per.rear_ended.p = 0.0` | `estimate.json: seeds[1].per.rear_ended.hits = 0` | `estimate.json: seeds[1].per.rear_ended.ess = 123.56470108185567` | `estimate.json: seeds[1].per.rear_ended.top_share = 0.0` |
| bus_bypass | `estimate.json: seeds[2].seed = 2` | `estimate.json: seeds[2].per.bus_bypass.p = 0.0005377381726433769` | `estimate.json: seeds[2].per.bus_bypass.hits = 36` | `estimate.json: seeds[2].per.bus_bypass.ess = 120.86673845634054` | `estimate.json: seeds[2].per.bus_bypass.top_share = 0.4261024176411109` |
| cut_in | `estimate.json: seeds[2].seed = 2` | `estimate.json: seeds[2].per.cut_in.p = 0.0` | `estimate.json: seeds[2].per.cut_in.hits = 0` | `estimate.json: seeds[2].per.cut_in.ess = 126.33117294892448` | `estimate.json: seeds[2].per.cut_in.top_share = 0.0` |
| crosswalk_block | `estimate.json: seeds[2].seed = 2` | `estimate.json: seeds[2].per.crosswalk_block.p = 5.208421696825843e-07` | `estimate.json: seeds[2].per.crosswalk_block.hits = 89` | `estimate.json: seeds[2].per.crosswalk_block.ess = 124.15610594958535` | `estimate.json: seeds[2].per.crosswalk_block.top_share = 0.39492614958320127` |
| rear_ended | `estimate.json: seeds[2].seed = 2` | `estimate.json: seeds[2].per.rear_ended.p = 0.0` | `estimate.json: seeds[2].per.rear_ended.hits = 0` | `estimate.json: seeds[2].per.rear_ended.ess = 134.01743664657906` | `estimate.json: seeds[2].per.rear_ended.top_share = 0.0` |

Các seed bus_bypass tương đối nhất quán: khoảng lo–hi giao nhau và các điểm IS nằm trong Wilson MC đã dẫn. Tuy nhiên, khoảng riêng của seed giữa không chứa điểm MC (`mc.json: p = 0.001`); không nên đòi các điểm ước lượng trùng nhau. Với crosswalk_block, khoảng của seed đầu và cuối không giao nhau: độ ổn định kém, dù các điểm đều dưới cận trên MC cho kiểu này.

Trường vượt ngưỡng chẩn đoán top_share > 0.5 là `estimate.json: seeds[1].per.crosswalk_block.top_share = 0.9315809461819868`. Tổng trọng số va chạm gần như do một mẫu quyết định; nhiều hits không bảo đảm độ chính xác. ESS đo phân tán trọng số toàn bộ mẫu, không thay thế chẩn đoán trọng số trên mẫu va chạm.

Các ô p bằng không của cut_in và rear_ended chỉ ghi nhận không có hits, không chứng minh xác suất thật bằng không. Phải dùng cận `estimate.json: seeds[0].per.cut_in.upper_zero_hits = 0.025`, `estimate.json: seeds[1].per.cut_in.upper_zero_hits = 0.025`, `estimate.json: seeds[2].per.cut_in.upper_zero_hits = 0.025`, `estimate.json: seeds[0].per.rear_ended.upper_zero_hits = 0.025`, `estimate.json: seeds[1].per.rear_ended.upper_zero_hits = 0.025`, `estimate.json: seeds[2].per.rear_ended.upper_zero_hits = 0.025`, thay vì các trường hi bằng không.

`estimate.json: budget = 16800` nhỏ hơn khoảng 96 000 lượt MC suy ra từ `mc.json: p = 0.001` theo công thức ở câu trước. Tuy nhiên, biến động seed và top_share chưa chứng minh IS đạt cùng mục tiêu ±20 %; không thể chỉ so ngân sách rồi khẳng định cùng độ chính xác.

Phần other lấy từ `estimate.json: other.source = mc.json`; đây là nhóm dư gồm nhiều cơ chế nên không có một cơ chế đơn nhất để proposal nhắm tới. Dù `estimate.json: other.k = 0` trên `estimate.json: other.n = 3000`, phải báo cận `estimate.json: other.hi = 0.0012788956648046956`, không coi xác suất bằng không. Khi diễn giải tổng cần giữ bất định của other và các cận upper_zero_hits của cơ chế chưa thấy.

## 5. Các kiểu thất bại (bước C2 + 3D)

Mỗi kiểu bạn tìm thấy: một ảnh chụp trình xem 3D (chương 9) ở đúng lúc va chạm, và một câu mô tả chuỗi sự kiện (ai thấy gì, ai chờ gì). Kiểu nào cuộc săn **không** tìm thấy, và bạn tin điều đó ở mức nào? Nếu chỉ chạy **một** cuộc săn chung cho mọi kiểu thì điều gì xảy ra (gợi ý: `hunt.json: found` của từng cuộc săn)?

## 6. Sửa planner (bước E + 3D)

`fix.json`: policy được chọn, `fail` và `fail_hi` trên TRAIN, trên TEST (`fail_hi_seen` và `fail_hi`), và của policy gốc trên TEST; vì sao `fail_hi` của bản sửa có thể lớn hơn `fail` của policy gốc dù bản sửa tốt hơn thật? quãng đường và số lần phanh gấp. Một ảnh chụp 3D có **bóng mờ** (policy gốc) cho một kịch bản mà bản sửa tránh được. Có núm chỉnh nào "nghe thì an toàn hơn" nhưng thực tế làm tăng thất bại? Giải thích bằng cách thế giới phản ứng lại.

## 7. Từ mô phỏng ra đường thật

Phân phối tự nhiên trong `space.py` là giả định. Chọn **ba** giả định ảnh hưởng nhiều nhất đến con số của bạn (ví dụ khe chấp nhận của xe chuyển làn, tỉ lệ người đi bộ ngó qua xe buýt, ma sát đường ướt). Với mỗi cái: dữ liệu thật nào sẽ thay nó, lấy từ đâu, và nếu nó sai thì con số của bạn sai theo hướng nào?

## 8. Giới hạn

Ba điều mà xác suất bạn báo **không** nói, ví dụ: thế giới chỉ có một con phố; người lái chỉ là IDM; cảm biến hoàn hảo trừ phần bị xe buýt che. Nếu được thêm một tuần, bạn sẽ đo gì tiếp?

## 9. Rắc rối của bạn (bước G + chương 10)

- **Rắc rối:** bạn dựng gì, và ngoài đời nó giống tình huống nào.
- **Căn cứ:** vì sao bạn nghĩ planner sẽ xử lý kém.
- **Giả định:** `space` và `exposure` lấy từ đâu (`hazard.json: why`).
- **Kết quả**, theo `mine.json`:
  - số va chạm `avoidable` (tránh được bằng phanh hết cỡ, biết trước), `unavoidable`, `unattributed`, và `low_speed`, với policy gốc và với policy bạn chọn; `accepted` (bao nhiêu phần `space` của bạn là khả thi);
  - P(lỗi planner | rắc rối) kèm khoảng Wilson;
  - tần suất mỗi chuyến, đặt cạnh tần suất lách xe buýt của bạn.
- **Hình:** một ảnh chương 9 hoặc 10 ở lúc va chạm.
- **Bạn tự lái thì sao:** nếu bạn đã thử lái, bạn tránh được không, và bạn làm gì khác planner?
- **Cách sửa:** núm chỉnh nào sửa được rắc rối này? Nếu không núm nào sửa được, planner cần thêm **logic gì** (mô tả, không cần code)?
