# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Nguyễn Ngọc Thái An
**MSSV:** 2A202602462
**Cohort:** A20-K4
**Ngày submit:** 2026-10-06

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Windows 11
- **CPU:** AMD Ryzen 5 4600H with Radeon Graphics
- **Cores:** 6 physical / 12 logical
- **CPU extensions:** Không được ghi trong `hardware.json`
- **RAM:** 23.4 GB
- **Accelerator:** NVIDIA GeForce GTX 1650, 4096 MiB; runtime dùng Vulkan vì driver CUDA 11.4 không tương thích bản CUDA đã chọn
- **llama.cpp asset đã tải:** `llama-b10488-bin-win-vulkan-x64.zip`
- **Model đã dùng:** Qwen3.5 0.8B (`LAB_MODEL=qwen35-0.8b`)
- **Quantization:** `Q4_K_M` + `UD-Q2_K_XL`

**Chạy ở đâu:** Laptop Windows local

**Setup story** (≤ 80 chữ): điều gì cần thay đổi để lab chạy trên máy bạn? Có bước
nào fail rồi phải workaround không?

Máy dùng Windows 11, Python 3.13.4 và có đủ RAM cho model Qwen3.5 0.8B. Runtime llama.cpp đã chọn bản Vulkan vì driver CUDA 11.4 không chạy được bản CUDA tương ứng. Khi wrapper `lab.ps1` báo lỗi parser, mình chạy trực tiếp các script Python trong `labs/` bằng `.venv\Scripts\python.exe`.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 2922 | 410 / 452 | 28.7 / 30.6 | 2219 / 2380 / 2380 | 34.8 |
| UD-Q2_K_XL | 0.39 | 2806 | 488 / 539 | 27.0 / 30.3 | 2174 / 2450 / 2450 | 37.1 |

**Quan sát** (≤ 60 chữ): 2-bit nhanh hơn bao nhiêu, và **có đáng không**? Bạn đã thử
hỏi cùng một câu trên cả hai (`make serve` vs `.venv/bin/python labs/02-serve/serve.py --compare`)
chưa? Chất lượng khác nhau thế nào?

Q2 nhỏ hơn 0.11 GB và decode nhanh hơn 1.07× (37.1 so với 34.8 tok/s), nhưng TTFT P50 cao hơn. Qua cùng một câu hỏi, Q4 diễn đạt rõ và mạch lạc hơn; Q2 vẫn dùng được nhưng hơi lặp và kém chính xác hơn. Với máy này, Q2 đáng dùng nếu ưu tiên tốc độ/dung lượng.

---

## 3. Serving under load  (rubric 8, 9, 10 — 20 points)

> Results from the load tests and batching metrics.

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 0.69 | 12000 | 21000 | 21000 | 8.7 | 0.0% |
| 50 | 0.83 | 24000 | 50000 | 56000 | 21.7 | 0.0% |

- **Offered load increased 5x; throughput increased:** 1.19x
- **P95 increased:** 2.38x
- **Effective concurrency at 50 users:** 21.7 versus `--parallel` = 4 slots


**Peak `llamacpp:n_busy_slots_per_decode`:** 3.64 / 4 slots (91%); `requests_deferred` reached 46.

**Saturation reading:** The server was already saturated at or below 50 users. Increasing the offered load 5x raised throughput only 1.19x, while P95 latency increased 2.38x. The 3.64/4 busy-slot peak and 46 deferred requests show that requests were queued after the four decode slots were occupied. I would first test `--parallel 6` or `8`, then keep the setting only if P95 stays within the SLO.
---

## 4. Integration  (rubric 12, 13 — 15 points)

> The pipeline used the shipped toy data and keyword-overlap retrieval.

| Day | Piece | Real or stub? |
|---|---|---|
| N16 Cloud/IaC | Stub | Stub
| N17 Data pipeline | Stub |Stub
| N18 Lakehouse | Stub |Stub
| N19 Vector + features | Stub |Stub
| N20 Serving | `llama-server` — real |

**Latency split** (mean of 3 queries):

- embed: 0.0 ms
- retrieve: 0.1 ms
- llm: 7178.2 ms
- **dominant stage:** llm (100% of total)

**Reflection:** The LLM is the clear bottleneck, which matches my expectation because generation is much more expensive than toy keyword retrieval. To halve latency, I would first reduce output length or use a faster/smaller quantization, then optimize the serving settings.
---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** Tăng thread count từ 1 lên 6 (physical-core default)

```
before:  18.3 tok/s (1 thread)
after:   34.2 tok/s (6 threads)
speedup: 1.87×
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

_Giải thích như đang nói với bạn ngồi cạnh. Bám vào **cơ chế**, không phải "vibes":
memory bandwidth? vector width? cache residency? scheduling? queueing? Nếu kết quả
**khác** với kỳ vọng từ deck — nói rõ, và giải thích vì sao. Grader thưởng điểm cho
lập luận đúng về một kết quả bất ngờ, hơn là một con số đẹp không được giải thích._

6 threads là điểm tối ưu vì đúng bằng số physical cores của Ryzen 5 4600H. Tăng lên 12 threads dùng logical cores nhưng tốc độ giảm còn 32.3 tok/s do các threads cạnh tranh execution resources, cache và memory bandwidth. Ở 24 threads, CPU bị oversubscribe và context switching làm throughput giảm mạnh còn 21.7 tok/s. Vì vậy, 6 threads là lựa chọn tốt nhất cho workload này.

---

## 6. Bonus  *(optional — tối đa 10 điểm)*

> Bỏ trống nếu không làm. Xem `docs/bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** _<B1 build-compare / B2 sweep nào / B4 challenge nào / B5 lựa chọn nào>_

**Numbers:**

```
before:  <số>
after:   <số>
speedup: <X.Y>×
```

**Điều này nói lên gì mà deck chưa nói:**

_(để trống nếu bạn không làm phần này)_

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

_(1–2 câu. Không bắt buộc, nhưng grader đọc hết.)_

_(để trống nếu bạn không làm phần này)_

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

_(Công cụ nào, dùng vào việc gì. Ghi "Không dùng" nếu không dùng.)_
