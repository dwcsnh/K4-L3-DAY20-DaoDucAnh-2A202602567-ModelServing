# 03 - Integrate: RAG pipeline run

Host `Linux-x86_64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.0 | 3418.2 | 3418.3 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.0 | 2665.9 | 2665.9 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.0 | 2649.7 | 2649.8 |

Mean per stage (ms): embed **0.0** · retrieve **0.0** ·
llm **2911.3** · total **2911.3**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Goodput@SLO counts only the requests per second that met the TTFT and TPOT targets. Throughput at saturation ignores SLOs.

**What problem does PagedAttention actually solve?**

> PagedAttention stores the KV cache in non-contiguous pages, which removes the internal fragmentation that wasted most GPU memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps because prefill is compute-bound and decode is memory-bandwidth-bound.


## Which N16-N19 pieces are real (required -- replace this line)

### 1. Phân loại Real và Stub
- N16 Cloud/IaC: stub
- N17 Data pipeline: stub
- N18 Lakehouse: stub (dữ liệu mẫu TOY_DOCS lưu trong bộ nhớ RAM)
- N19 Vector + features: stub (tìm kiếm theo keyword overlap in-memory, không dùng vector database)
- N20 Serving: real (llama-server chạy cục bộ tại port 8080 xử lý inference thực tế)

### 2. Phân tích Dominant stage và Kỳ vọng
- Stage chiếm nhiều nhất: stage llm chiếm 2911.3 ms (100% tổng thời gian thực thi). Các bước embed và retrieve đều mất 0.0 ms.
- Mức độ khớp với kỳ vọng: Hoàn toàn đúng như kỳ vọng vì các tầng trước đều là stub chạy in-memory với vài document ngắn nên chi phí tính toán xấp xỉ 0. Toàn bộ thời gian xử lý dồn vào bước neural network inference trên CPU (prefill prompt context tốn khoảng 1.5s và decode sinh câu trả lời tốn khoảng 1.3s).

### 3. Phương án giảm latency 2x cho pipeline
- Stage cần tấn công: Bắt buộc phải tấn công vào stage llm vì theo định luật Amdahl, stage này chiếm trọn 100% latency của pipeline.
- Các biện pháp cụ thể:
  - Tận dụng Prompt Caching: Giữ nguyên system prompt cố định giữa các truy vấn để server tái sử dụng prefix cache, loại bỏ gần như toàn bộ thời gian prefill (~1500 ms) sau request đầu tiên.
  - Hạ mức quantization xuống 2-bit (UD-Q2_K_XL): Giúp giảm 25% kích thước trọng số nạp qua memory bus, tăng tốc độ decode.
  - Tối ưu hóa Context Budget: Chọn lọc các chunk ngắn gọn và trúng đích nhất để giảm độ dài prompt, trực tiếp kéo giảm số token phải prefill.
