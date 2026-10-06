# 02 - Continuous batching under load (u50)

Host `Linux-x86_64` · `--parallel 4` · 30 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.91 of 4 slots (98%) |
| `requests_processing` | 4 |
| `requests_deferred` | 46 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 3243 |

Highest sampled value was **3.91 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation (required -- replace this line)

### 1. Peak Batch Width và so sánh với Effective Concurrency
- Peak batch width quan sát được: Giá trị đỉnh của `n_busy_slots_per_decode` đạt 3.91 / 4 slots (98%) và `requests_processing` chạm mức tối đa 4 / 4 slots. Điều này chứng minh scheduler của `llama-server` đã gom (pack) và chạy continuous batching gần như tối đa công suất song song của 4 decode slots.
- So sánh với `02-server-results.md`: Con số này khác với `effective concurrency` đo được là 17.1 (tính theo Định luật Little: $L = \lambda \times W$).

### 2. Lý giải sự chênh lệch & Đánh giá độ tin cậy
- Nguyên nhân chênh lệch:
  - `n_busy_slots_per_decode` (3.91) và `requests_processing` (4) đo tài nguyên tính toán thực tế tại compute engine của server tại một thời điểm, bị giới hạn cứng bởi tham số cấu hình `--parallel 4`.
  - `effective concurrency` (17.1) đo toàn bộ số request đang tồn đọng trong hệ thống từ góc nhìn của client, bao gồm cả thời gian tính toán và thời gian nằm chờ trong hàng đợi.
  - Bằng chứng là gauge `requests_deferred` đã tăng lên tới 46. Do chỉ có 4 slots xử lý, 46 requests còn lại bị hoãn và phải xếp hàng, khiến độ trễ P95 tăng lên 56 giây.
- Tin số nào và vì sao:
  - Cả hai số liệu đều chính xác nhưng mô tả hai khía cạnh khác nhau:
    - `n_busy_slots_per_decode` (3.91/4) khi đánh giá hiệu suất batching / slot utilization: Cho thấy llama.cpp scheduler đang hoạt động hiệu quả, không để slot nào bị lãng phí.
    - `effective concurrency` (17.1) khi đánh giá mức độ quá tải / bão hòa hệ thống (system saturation & queueing): Phản ánh đúng thực tế hệ thống đã bị nghẽn cổ chai và khách hàng đang phải chịu độ trễ hàng đợi rất lớn.
