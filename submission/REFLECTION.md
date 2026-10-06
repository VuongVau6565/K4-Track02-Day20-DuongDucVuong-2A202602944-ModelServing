# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Dương Đức Vương
**MSSV:** 2A202602944
**Cohort:** A20-K4
**Ngày submit:** 2026-10-06

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Windows 10 (host Windows-AMD64)
- **CPU:** Intel Core i7-4500U @ 1.80 GHz
- **Cores:** 2 physical / 4 logical (theo `hardware.json`)
- **CPU extensions:** AVX2 (Intel Haswell)
- **RAM:** 11.9 GB
- **Accelerator:** Vulkan device detected; inference ran with `ngl=0` (CPU)
- **llama.cpp asset đã tải:** prebuilt llama.cpp b10488, Windows AMD64, Vulkan
- **Model đã dùng:** Qwen3.5 0.8B (`LAB_MODEL=qwen35-0.8b`)
- **Quantization:** Q4_K_M + UD-Q2_K_XL (từ `models/active.json`)

**Chạy ở đâu:** máy local Windows, không dùng Colab/Kaggle.

**Setup story** (≤ 80 chữ): điều gì cần thay đổi để lab chạy trên máy bạn? Có bước
nào fail rồi phải workaround không?

Setup và tải runtime/model hoàn tất trên máy local, không cần compiler hay GPU. Trên
PowerShell, lần đầu chạy benchmark cần đặt `PYTHONUTF8=1` để tránh lỗi in ký tự
Unicode theo code page mặc định. Cổng 8080 ban đầu đang được dùng; sau khi tiến
trình cũ dừng, server lab chạy bình thường trên cổng mặc định.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 8870 | 819 / 1381 | 84.3 / 170.5 | 5874 / 11853 / 11853 | 11.9 |
| UD-Q2_K_XL | 0.39 | 3704 | 1316 / 1893 | 82.6 / 125.0 | 6723 / 9192 / 9192 | 12.1 |

**Quan sát** (≤ 60 chữ): 2-bit nhanh hơn bao nhiêu, và **có đáng không**? Bạn đã thử
hỏi cùng một câu trên cả hai (`make serve` vs `.venv/bin/python labs/02-serve/serve.py --compare`)
chưa? Chất lượng khác nhau thế nào?

Bản Q2 decode 12.1 tok/s, nhanh hơn Q4 1.7%, và nhỏ hơn 0.11 GB (22%), nhưng TTFT
và E2E median lại cao hơn; qua so sánh câu trả lời, Q2 cũng kém đúng trọng tâm hơn.
Vì vậy tôi ưu tiên Q4 nếu cần chất lượng; chênh lệch decode quá nhỏ để đổi lấy
trade-off đó. Số liệu không cho thấy Q2 nhanh hơn toàn diện.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 0.12 | 30000 | 49000 | 49000 | 3.8 | 0.0% |
| 50 | 0.11 | 47000 | 47000 | 47000 | 4.0 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 0.87× (RPS đo được giảm)
- **P95 tăng:** 0.96× (mẫu quá nhỏ để so sánh đáng tin cậy)
- **Effective concurrency ở 50 users:** 4.0 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): 2.05 / 4 slots; `requests_deferred` peak = 46.

**Saturation reading** (≤ 80 chữ): server của bạn bão hoà ở đâu, và **bằng chứng nào**
thuyết phục bạn? Nếu P95 tăng nhanh hơn RPS thì phần latency thêm đó là queue time hay
compute time — bạn biết bằng cách nào? Nếu bạn phải nâng goodput@SLO, bạn sẽ đổi knob
nào **trước**, và vì sao knob đó?

Ở 50 users, metrics ghi nhận 4 request đang xử lý, peak 46 bị defer và
`n_busy_slots_per_decode` đạt 2.05/4: có queue. Tuy nhiên Locust chỉ hoàn tất 6
request ở 10 users và 5 ở 50 users; RPS đo được là 0.87×, P95 là 0.96× và effective
concurrency là 4.0/4. Mẫu nhỏ nên chưa thể định vị saturation hoặc tách queue khỏi
compute trong P95. Tôi sẽ thử giới hạn output tokens để giải phóng decode slot sớm hơn.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | Stub — localhost only, no cluster/IaC connected |
| N17 Data pipeline | Stub — documents are supplied in-memory |
| N18 Lakehouse | Stub — toy `TOY_DOCS`, no lakehouse connected |
| N19 Vector + features | Stub — keyword overlap, no vector index/features |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.2 ms
- llm: 19,576.9 ms
- **stage chiếm nhiều nhất:** llm (xấp xỉ 100% của total 19,577.2 ms)

**Reflection** (≤ 60 chữ): bottleneck ở đâu? Có khớp với kỳ vọng của bạn không? Nếu
phải giảm latency của pipeline này 2×, bạn sẽ tấn công vào đâu?

LLM chiếm gần như toàn bộ latency (19,576.9/19,577.2 ms), đúng như dự đoán với
model chạy CPU. Embed không dùng server nên mất 0 ms; keyword retrieval trên sáu
tài liệu đồ chơi chỉ mất 0.2 ms. Muốn giảm tổng latency 2×, tôi sẽ giới hạn số
token sinh ra trước: decode chiếm phần lớn thời gian LLM, đổi lại câu trả lời ngắn hơn.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** Tăng số thread decode từ 1 lên 2 (`-t 1` → `-t 2`), bằng số nhân vật lý.

```
before:  -t 1 = 4.5 tok/s (llama-bench tg128, trung bình 2 lần chạy)
after:   -t 2 = 7.1 tok/s (llama-bench tg128, trung bình 2 lần chạy)
speedup: 1.58×
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

Trên CPU i7-4500U 2 nhân vật lý/4 luồng logic, tăng từ 1 lên 2 thread làm decode
tăng từ 4.5 lên 7.1 tok/s (1.58×), vì hai nhân vật lý có thể xử lý công việc song
song. Đây là điểm tốt nhất trong sweep và cũng là knee: thêm thread thứ ba và thứ
tư chỉ dùng SMT trên hai nhân đó, không thêm nhân vật lý hay băng thông bộ nhớ.

Ở 4 threads tốc độ giảm còn 6.2 tok/s, thấp hơn 13% so với 2 threads. Mỗi bước
decode phải truy cập trọng số model; các luồng SMT cùng chia sẻ tài nguyên thực thi
và cache trên nhân, còn nhiều luồng cạnh tranh nguồn dữ liệu bộ nhớ thay vì làm
tăng khả năng xử lý độc lập. Kết quả này phù hợp với giới hạn tài nguyên của
decode, nhưng benchmark không đo trực tiếp băng thông nên không khẳng định riêng
băng thông là nguyên nhân duy nhất. Baseline trước đó đã dùng 2 threads, đúng với
mức tốt nhất tìm được.

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

Tôi dùng Copilot để hỗ trợ đọc lỗi, chạy và diễn giải các phép đo thật trên máy
này, cũng như điền report/reflection dựa trên log sinh ra. Tôi đã xem lại các số
liệu và cần có thể tự giải thích các kết luận đã nộp.
