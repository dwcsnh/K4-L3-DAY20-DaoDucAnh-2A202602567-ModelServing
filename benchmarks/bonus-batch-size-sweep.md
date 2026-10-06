# Bonus - Batch-size sweep (chunked prefill)

Host `Linux-x86_64` · llama.cpp `b10488` ·
`threads=8` `ngl=0` · metric `pp512`

| -b (logical) | -ub (micro) | pp512 (tok/s) | vs best |
|:--|--:|--:|--:|
| 128 | 128 | 80.8 | 93% |
| 256 | 256 | 86.2 | 99% |
| 512 | 256 | 86.6 | 100% |
| 512 | 512 | 82.9 | 95% |
| 1024 | 512 | 86.9 | 100% |
| 2048 | 512 | 86.6 | 100% |

Best: `-b 1024 -ub 512` at 86.9 tok/s
(1.08x the slowest point tested).

This sweep only measures the throughput half of the trade. The cost it hides is
TTFT for queued requests: a larger micro-batch holds the device longer per step,
so anything waiting behind it waits longer. To see both halves, re-run
`make load-50` with your best and worst settings via
`.venv/bin/python labs/02-serve/serve.py -- -b N -ub M` and compare P95.

## Your finding (required -- replace this line)

### 1. Lựa chọn cấu hình cho môi trường Production
- Cấu hình đề xuất chạy production: `-b 512 -ub 256`.
- Lý do:
  - Throughput prefill của `-b 512 -ub 256` đạt 86.6 tok/s, tương đương 99.7% so với mức cao nhất `-b 1024 -ub 512` (86.9 tok/s).
  - Sử dụng micro-batch `-ub 256` nhỏ hơn cho phép áp dụng chunked prefill hiệu quả: prefill được chia thành các lát nhỏ, xen kẽ với các bước decode của các slot khác. Nếu dùng `-ub 512`, compute engine bị chiếm giữ liên tục trong thời gian dài hơn cho mỗi bước prefill, gây hiện tượng bỏ đói (starvation) đối với các request decode đang chạy đồng thời, khiến TPOT tăng vọt.

### 2. Chỉ số cần đo lường trên server chịu tải
- Đo TTFT P95/P99 và TPOT P95 dưới tải thực tế (bằng `make load-50` kết hợp các thiết lập `-b` và `-ub` khác nhau).
- Cần giám sát thời gian chờ trong hàng đợi và sự biến động của inter-token latency để đảm bảo các request prompt dài không gây ra decode bubbles hay làm đội P95 của toàn bộ hệ thống.
