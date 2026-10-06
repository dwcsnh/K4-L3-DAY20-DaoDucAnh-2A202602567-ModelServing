# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Đào Đức Anh
**MSSV:** 2A202602567
**Cohort:** A20-K4
**Ngày submit:** 2026-10-06

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Linux Ubuntu 24.04 (kernel 7.0.0-34-generic x86_64)
- **CPU:** AMD Ryzen 9 6900HS Creator Edition
- **Cores:** 8 physical / 16 logical
- **CPU extensions:** AVX2
- **RAM:** 27.1 GB
- **Accelerator:** CPU only
- **llama.cpp asset đã tải:** llama-b10488-bin-linux-x64.tar.gz
- **Model đã dùng:** Gemma 4 E2B (`LAB_MODEL=gemma4-e2b`)
- **Quantization:** UD-Q4_K_XL + UD-Q2_K_XL (từ `models/active.json`)

**Chạy ở đâu:** laptop của tôi

**Setup story** (≤ 80 chữ): Setup trên Linux x86_64 chạy mượt mà ngay từ đầu với prebuilt binary b10488. Dung lượng RAM 27 GB thoải mái để tải cả hai mô hình 4-bit và 2-bit mà không cần cấu hình swap hay workaround.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 3073 | 276 / 325 | 52.4 / 54.4 | 3487 / 3725 / 3725 | 19.1 |
| UD-Q2_K_XL | 2.24 | 3031 | 423 / 500 | 44.4 / 47.1 | 3205 / 3288 / 3288 | 22.5 |

**Quan sát** (≤ 60 chữ): Mô hình 2-bit decode nhanh hơn 1.18x (22.5 vs 19.1 tok/s), tiết kiệm 0.73 GB RAM. Với các câu hỏi đơn giản tiếng Việt, hai bản trả lời tương đồng; nhưng với câu hỏi suy luận phức tạp 2-bit dễ suy giảm chất lượng nên 4-bit vẫn là mặc định an toàn.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 0.44 | 18000 | 30000 | 32000 | 8.1 | 0.0% |
| 50 | 0.47 | 41000 | 56000 | 57000 | 17.1 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 1.07×
- **P95 tăng:** 1.87×
- **Effective concurrency ở 50 users:** 17.1 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): 3.91 / 4 slots

**Saturation reading** (≤ 80 chữ): Server bão hòa ở 50 users: tải tăng 5x nhưng RPS chỉ tăng 1.07x, còn P95 tăng vọt 1.87x. Độ trễ tăng thêm hoàn toàn là queue time vì compute slots đã kín (3.91/4) và requests_deferred đạt 46. Để nâng goodput@SLO (P95 <= 30s), tôi sẽ đổi sang UD-Q2_K_XL trước để giảm tải memory bandwidth trên CPU.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | Local Linux setup | stub |
| N17 Data pipeline | Ingestion toy docs | stub |
| N18 Lakehouse | In-memory toy docs | stub |
| N19 Vector + features | Keyword overlap | stub |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.0 ms
- llm: 2911.3 ms
- **stage chiếm nhiều nhất:** llm (100% của total)

**Reflection** (≤ 60 chữ): Bottleneck 100% ở stage llm (2911.3 ms), đúng như kỳ vọng do retrieval là stub in-memory. Để giảm latency 2x, tôi sẽ tấn công llm bằng prompt caching để triệt tiêu prefill time (~1500 ms) và dùng mô hình 2-bit để tăng tốc decode.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** Hạ số luồng thực thi từ mức logical/oversubscribe (-t 16) về đúng số physical cores (-t 8)

```
before:  9.8 tok/s
after:   20.0 tok/s
speedup: 2.04×
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

Ở bước decode (-metric tg128), tác vụ hoàn toàn bị giới hạn bởi băng thông bộ nhớ (memory bandwidth-bound). Mỗi token sinh ra đòi hỏi CPU phải stream toàn bộ ~2.97 GB trọng số mô hình từ RAM qua bus bộ nhớ vào registers, với arithmetic intensity rất thấp (~1 FLOP/byte). Khi tăng từ 1 lên 8 threads, các kênh bộ nhớ DRAM của CPU Ryzen 9 6900HS được khai thác tối đa và đạt ngưỡng bão hòa tại 8 physical cores (20.0 tok/s).

Khi tăng lên 16 threads (kích hoạt SMT / Hyper-Threading) hoặc 32 threads (oversubscription), hai luồng cùng chia sẻ execution units và L1/L2 cache trên một core vật lý. Điều này gây ra hiện tượng cache thrashing và tranh chấp bộ nhớ dữ dội. Đồng thời, do llama.cpp sử dụng barrier synchronization sau mỗi layer, việc context-switching liên tục khiến các luồng bị chậm giữ chân toàn bộ các luồng khác, làm thông lượng rơi thẳng đứng từ 20.0 xuống còn 9.8 tok/s. Việc khóa số luồng về đúng 8 physical cores đã loại bỏ hoàn toàn các overhead này.


---

## 6. Bonus  *(optional — tối đa 10 điểm)*

> Bỏ trống nếu không làm. Xem `docs/bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** B2 batch-size-sweep (`make sweep-batch`) và B5/C8 semantic-cache (`make semantic-cache-offline`)

**Numbers:**

```
before:  80.8 tok/s (-b 128 -ub 128)
after:   86.9 tok/s (-b 1024 -ub 512)
speedup: 1.08×
```

**Điều này nói lên gì mà deck chưa nói:**
Deck lý thuyết nhấn mạnh rằng chunked prefill với batch size lớn sẽ tối ưu hóa throughput nhờ gom tính toán vào kernel. Tuy nhiên, thực nghiệm trên CPU cho thấy throughput prefill tăng rất nhanh từ 80.8 tok/s (-b 128) lên 86.6 tok/s (-b 512 -ub 256), nhưng sau đó gần như đi ngang dù tăng tiếp micro-batch lên -ub 512 (chỉ đạt 86.9 tok/s, chênh lệch chưa tới 0.4%). 

Trong môi trường serving thực tế, cấu hình -b 512 -ub 256 là điểm cân bằng lý tưởng: giữ được 99.7% throughput tối đa trong khi micro-batch đủ nhỏ để tránh chiếm dụng CPU quá lâu cho một prefill chunk, ngăn chặn hiện tượng làm đói (starvation) các lượt decode đồng thời và bảo vệ P95 latency.

Bên cạnh đó, thử nghiệm C8 Semantic Cache chỉ ra rằng trước khi request chạm vào KV cache và LLM, một tầng cache ngữ nghĩa phía trước có thể tiết kiệm 100% chi phí tính toán đối với các câu hỏi paraphrase (đạt 38% hit rate trong demo), đóng vai trò then chốt trong kiến trúc serving nhiều tầng cache.

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

Điểm ngạc nhiên nhất là hiện tượng hiệu năng rơi thẳng đứng (tụt hơn 50% tok/s) khi tăng số luồng từ 8 lên 16 threads ở bước tuning. Dù chip có 16 logical cores, việc các luồng Hyper-Threading cùng chia sẻ vector units và tranh chấp cache L1/L2 đã biến lợi thế đa luồng thành nghẽn cổ chai nghiêm trọng do barrier synchronization.


---

## 8. Self-check trước khi push

- [ ] `hardware.json` committed
- [ ] `models/active.json` committed
- [ ] `benchmarks/01-quickstart-results.md` committed (`make bench`)
- [ ] `benchmarks/01-tuning-tg128.md` committed (`make tune`)
- [ ] `benchmarks/02-server-results.md` committed (`make load-report`)
- [ ] `benchmarks/02-server-batching-u50.md` hoặc `-metrics-u50.csv` committed (`make metrics`)
- [ ] `benchmarks/locust-10_stats.csv` + `locust-50_stats.csv` committed (`make load-10` / `load-50`)
- [ ] `benchmarks/03-integration-results.md` committed (`make pipeline`)
- [ ] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md`
      đã được thay bằng nhận xét của bạn
- [ ] 5 screenshots trong `submission/screenshots/`
- [ ] `make verify` → **exit 0**
- [ ] Repo tên đúng mẫu `K4-L3-DAY20-HoVaTen-MSSV-ModelServing` (xem `docs/SUBMISSION.md`)
- [ ] Repo GitHub ở chế độ **public**
- [ ] Đã push và paste public URL vào VinUni LMS **trước 23:59 (UTC+7) ngày làm lab**
- [ ] **Không** commit `models/*.gguf`, `runtime/` hay `.env` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.

---

## 9. Khai báo sử dụng AI  *(xem `docs/RULES.md` §3)*

Sử dụng Antigravity IDE (Gemini AI assistant) để hỗ trợ tìm hiểu khái niệm, đọc log và debug lệnh kiểm thử, kiểm tra đối chiếu checklist rubric. Mọi số liệu đo lường và benchmark đều được chạy trực tiếp trên máy cá nhân.

