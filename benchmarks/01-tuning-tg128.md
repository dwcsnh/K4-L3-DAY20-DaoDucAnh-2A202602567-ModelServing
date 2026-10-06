# 01 - Tune: thread-count sweep

Model `gemma-4-E2B-it-UD-Q4_K_XL.gguf` · host `Linux-x86_64` · llama.cpp `b10488`
CPU: **8 physical · 16 logical** cores · `ngl=0` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 8.4 | 42% |
| 4 | 19.5 | 98% |
| 8 | 20.0 | 100% |
| 16 | 9.8 | 49% |
| 32 | 9.7 | 49% |

**Best**: `-t 8` at 20.0 tok/s
**Slowest tested**: `-t 1` at 8.4 tok/s (2.37x spread)
**Against the physical-core default** (`-t 8`, 20.0 tok/s): 1.00x

Use this in your run:

```bash
LAB_N_THREADS=8 make bench
```

## Your explanation

### 1. Vị trí Knee
- Knee nằm tại `-t 8` (20.0 tok/s), tương ứng đúng số lượng physical cores (8 cores) của máy.
- Từ 1 → 4 threads: Throughput tăng từ 8.4 lên 19.5 tok/s do khai thác tốt hơn các kênh bộ nhớ và đơn vị SIMD/AVX.
- Từ 4 → 8 threads: Throughput tăng từ 19.5 → 20.0 tok/s do hệ thống đã bắt đầu chạm trần băng thông.
- Từ 8 → 16 threads (Logical/SMT) & 32 threads (Oversubscription): Hiệu năng giảm xuống còn 9.8 tok/s ở 16 threads và 9.7 tok/s ở 32 threads.

### 2. Nguyên nhân và Cơ chế 
- Decode là tác vụ Memory Bandwidth Bound: Ở giai đoạn decode (`tg128`), arithmetic intensity rất thấp (~1 FLOP/byte). Mỗi token sinh ra đòi hỏi CPU phải stream toàn bộ ~2.97 GB trọng số từ DRAM vào cache/registers. Băng thông DRAM (Dual-channel) đã đạt ngưỡng bão hòa hoàn toàn ở khoảng 4–8 threads; cấp thêm threads không thể tăng thêm tốc độ nạp dữ liệu.
- Tranh chấp tài nguyên do SMT (Hyper-Threading Contention): Khi nâng lên 16 threads, 2 thread logic cùng chạy trên 1 physical core và phải chia sẻ L1/L2 cache, các vector execution units (AVX2/FMA) và memory execution buffers. Hai thread nặng về tensor computation và memory load liên tục giành giật tài nguyên của nhau.
- Cache Thrashing & Eviction: Ở 16 và 32 threads, footprint bộ nhớ của các thread vượt quá dung lượng L1/L2 cache riêng của từng core, dẫn tới hiện tượng cache thrashing — các thread liên tục đẩy dữ liệu của nhau ra khỏi cache, làm tăng mạnh L1/L2/L3 cache misses và gây tắc nghẽn memory bus.
- Barrier Synchronization Overhead & Preemption: `llama.cpp` đồng bộ các worker threads qua barriers sau mỗi layer computation. Ở mức 16 threads (SMT) và đặc biệt 32 threads (oversubscription 2× logical cores), OS scheduler phải liên tục context-switch giữa các compute threads đang spin-wait. Chỉ cần một thread bị trễ (do chờ memory hoặc bị preempted) sẽ kéo toàn bộ các threads khác phải đứng chờ tại barrier, khiến thông lượng tụt hơn 50%.

