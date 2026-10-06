# 02 - Serve: load test + saturation reading

Host `Linux-x86_64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=8` ·
`ngl=0`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 25 | 0.44 | 18000 | 30000 | 32000 | 8.1 | 0.0% |
| 50 | 27 | 0.47 | 41000 | 56000 | 57000 | 17.1 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **1.07x** (21% of linear) |
| P95 latency | **1.87x** |
| Effective concurrency at 50 users | 17.1 vs `--parallel 4` slots (occupancy/slot ratio 4.27) |

**Saturated.** Throughput delivered only 1.07x for 5x the offered load, and effective concurrency (17.1) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 1.07x while P95 moved 1.87x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Your reading (required -- replace this line)

### 1. Điểm bão hòa và bằng chứng thực nghiệm
- Vị trí bão hòa: Server đã bắt đầu chạm trần từ khoảng 10 users và rơi vào trạng thái bão hòa hoàn toàn (Saturated) ở 50 users.
- Con số chứng minh thuyết phục:
  - Tải đưa vào (offered load) tăng 5x (từ 10 lên 50 users) nhưng throughput thực tế (RPS) chỉ tăng 1.07x (từ 0.44 lên 0.47 RPS), chỉ đạt 21% so với mức kỳ vọng tuyến tính.
  - Ngược lại, độ trễ P95 tăng tới 1.87x (từ 30,000 ms lên 56,000 ms).
  - Effective concurrency vọt lên 17.1, tương ứng tỷ lệ occupancy/slot đạt 4.27 lần so với cấu hình --parallel 4 slots.
  - Kết hợp với gauge requests_deferred đạt đỉnh 46 ở file metrics u50, điều này khẳng định 4 slot decode luôn kín và toàn bộ phần tải đưa thêm vào đã bị dồn thành queue time (thời gian chờ trong hàng đợi) thay vì tăng thêm throughput.

### 2. Knob ưu tiên thay đổi để nâng goodput@SLO
- Giả định mục tiêu SLO: P95 latency <= 30 giây (mức đáp ứng tốt ở tải 10 users). Tại tải 50 users, goodput gần như bằng 0 vì P95 đã vọt lên 56 giây.
- Knob đổi đầu tiên: Giảm mức quantization của mô hình từ UD-Q4_K_XL xuống UD-Q2_K_XL.
- Lý do chọn knob này thay vì knob khác:
  - Trên môi trường CPU serving, decode phase bị thắt nút cổ chai bởi băng thông bộ nhớ (memory bandwidth). Khi chạy continuous batching với 4 slots, CPU vẫn phải nạp liên tục toàn bộ trọng số mô hình qua memory bus cho mỗi token sinh ra.
  - Hạ quantization xuống UD-Q2_K_XL giúp giảm kích thước mô hình từ 2.97 GB xuống 2.24 GB (giảm 25% lưu lượng truyền tải trên RAM bus). Điều này trực tiếp kéo giảm TPOT và compute time trên từng slot, giúp hoàn thành request sớm hơn và xả hàng đợi nhanh hơn, đưa P95 trở lại dưới ngưỡng SLO.
  - Nếu chỉ tăng số slot (ví dụ nâng --parallel lên 8) mà giữ nguyên dung lượng mô hình, các slot sẽ càng tranh chấp băng thông bộ nhớ gay gắt hơn trên CPU, khiến độ trễ per-slot tăng lên và không cải thiện được P95.
