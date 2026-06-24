# Reflection - Lab 20 (Báo cáo cá nhân)

> Đây là báo cáo cá nhân. Mỗi học viên chạy lab trên laptop của mình, với spec của mình. Số liệu của bạn không so sánh được với bạn cùng lớp - chỉ so sánh **before vs after trên chính máy bạn**. Grade rubric tính theo độ rõ ràng của setup + tuning của bạn, không phải tốc độ tuyệt đối.

---

**Họ Tên:** Nguyễn Anh Kiệt
**Cohort:** K2
**Ngày submit:** 2026-06-24

---

## 1. Hardware spec (từ `00-setup/detect-hardware.py`)

> Paste output của `python 00-setup/detect-hardware.py` vào đây, hoặc điền thủ công:

- **OS:** Windows 11 + WSL2 Ubuntu 22.04
- **CPU:** Intel Core i7-8850H
- **Cores:** 6 physical / 12 logical
- **CPU extensions:** AVX2
- **RAM:** 15.8 GB
- **Accelerator:** NVIDIA CUDA-capable GPU, but this run used the CPU native server build
- **llama.cpp backend chosen:** CPU for the source-built `llama-server`
- **Recommended model tier:** Qwen2.5-1.5B

**Setup story** (<= 80 chữ): những gì cần thay đổi để lab chạy được trên máy bạn?

Dùng WSL2 cho Python và native build stack, giữ model trong `models/`, và chuyển source build sang filesystem ext4 của WSL vì build `llama.cpp` trực tiếp từ `/mnt/d` quá nặng và không ổn định trên máy này.

---

## 2. Track 01 - Quickstart numbers (từ `benchmarks/01-quickstart-results.md`)

> Paste bảng từ `benchmarks/01-quickstart-results.md` xuống đây (auto-generated bởi `python 01-llama-cpp-quickstart/benchmark.py`).

| Model | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode rate (tok/s) |
|---|---:|---:|---:|---:|---:|
| Q4_K_M | 93455 | 3009 / 3717 | 63.6 / 73.6 | 7075 / 7861 / 7958 | 15.7 |
| Q2_K   | 65735 | 2958 / 3743 | 50.9 / 61.9 | 6066 / 7195 / 7381 | 19.6 |

**Một quan sát** (<= 50 chữ): Q4_K_M vs Q2_K trên máy bạn - số liệu nói gì?

Q2_K nhanh hơn, nhưng Q4_K_M có tradeoff chất lượng/kích thước tốt hơn cho máy này. Chênh lệch đủ lớn để tôi giữ Q4_K_M làm mặc định nếu không ưu tiên throughput.

---

## 3. Track 02 - llama-server load test

> Chạy locust hai lần ở concurrency 10 và 50, rồi paste tóm tắt bên dưới.

| Concurrency | Total RPS | TTFB P50 (ms) | E2E P95 (ms) | E2E P99 (ms) | Failures |
|--:|--:|--:|--:|--:|--:|
| 10 | 0.31 | 25000 | 31000 | 31000 | 0 |
| 50 | 0.46 | 26000 | 41000 | 41000 | 0 |

**Batching observation** (từ `record-metrics.py`): peak `llamacpp:n_busy_slots_per_decode` / `requests_processing` ở concurrency 50 = 3.91 / 4, nghĩa là server đã bão hòa 4 native slots và bắt đầu xếp hàng request mới.

Peak `llamacpp:n_busy_slots_per_decode` đạt 3.91 với `requests_processing` = 4 và `requests_deferred` = 46. Nghĩa là server bão hòa toàn bộ bốn native slots và bắt đầu queue work mới thay vì drop request.

---

## 4. Track 03 - Milestone integration

- **N16 (Cloud/IaC):** stub: localhost only
- **N17 (Data pipeline):** stub: in-memory dict
- **N18 (Lakehouse):** stub: SQLite
- **N19 (Vector + Feature Store):** stub: TOY_DOCS

**Nơi tốn nhiều ms nhất** trong pipeline (đo bằng `time.perf_counter` trong `pipeline.py`):

- embed: about 0.1 ms
- retrieve: about 0.0 to 0.1 ms
- llama-server: about 2278.1 to 12403.1 ms

Đã chạy 3 query end to end. Hai query khoảng 12.4 s tổng, một query khoảng 2.3 s tổng. Retrieval và embedding gần như không đáng kể; lời gọi LLM chiếm phần lớn latency.

**Reflection** (<= 60 chữ): bottleneck nằm ở đâu? Có khớp với kỳ vọng không?

Bottleneck nằm ở model invocation, không phải retrieval. Điều này khớp kỳ vọng vì pipeline của lab là local và bước LLM luôn chiếm phần lớn end-to-end latency.

---

## 5. Bonus - Thay đổi đơn lẻ quan trọng nhất

> **Phần quan trọng nhất.** Chọn **một** thay đổi từ bonus track (build flag, thread sweep, quant pick, GPU offload, KV-cache quantization, speculative decoding, hoặc challenge nào trong `BONUS-llama-cpp-optimization/CHALLENGES.md`) tạo ra speedup lớn nhất trên máy bạn.

**Change:** Tuned the CPU decode thread count to the physical core count instead of oversubscribing all logical threads.

**Before vs after** (paste 2-3 dòng từ sweep output):

```
before: -t 24 -> 2.8 tok/s
after:  -t 6 -> 19.8 tok/s
speedup: ~7.1x
```

**Vì sao nó hiệu quả** (1-2 đoạn ngắn - đây là phần grader đọc kỹ nhất):

Decode path trên CPU này bị giới hạn bởi memory bandwidth. Khi số worker chạm tới số core vật lý, thêm thread chỉ làm tăng contention trên cùng cache và memory channels thay vì tạo thêm throughput thực.

Vì vậy `-t 6` thắng trên máy này. `-t 24` dàn công việc lên hyperthread và làm oversubscribe shared execution resources, nên throughput giảm thay vì tăng. Đường cong này đúng với một workload bị giới hạn bởi băng thông.

---

## 6. (Optional) Điều ngạc nhiên nhất

Điều ngạc nhiên nhất là server chạm trần throughput rất nhanh khi native slots đầy. Tăng concurrency không làm latency tốt hơn; nó chỉ làm queue dài hơn.

---

## 7. Self-graded checklist

- [ ] `hardware.json` đã commit
- [ ] `models/active.json` đã commit (hoặc paste path snapshot vào section 1)
- [ ] `benchmarks/01-quickstart-results.md` đã commit
- [ ] `benchmarks/02-server-results.md` (hoặc CSV từ `record-metrics.py`) đã commit
- [ ] `benchmarks/bonus-*.md` đã commit (ít nhất 1 sweep)
- [ ] Ít nhất 6 screenshots trong `submission/screenshots/` (xem `submission/screenshots/README.md`)
- [ ] `make verify` exit 0 (chạy ngay trước khi push)
- [ ] Repo trên GitHub ở chế độ **public**
- [ ] Đã paste public repo URL vào VinUni LMS

---

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Nếu private, grader không xem được -> 0 điểm.
