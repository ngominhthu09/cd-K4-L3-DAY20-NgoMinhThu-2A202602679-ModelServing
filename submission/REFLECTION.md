# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Ngô Minh Thu
**MSSV:** 2A202602679
**Cohort:** AI20K Cohort 4
**Ngày submit:** 2026-10-06

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Windows 11 (AMD64)
- **CPU:** Intel Core i5-10400H @ 2.60 GHz
- **Cores:** 4 physical / 8 logical
- **CPU extensions:** AVX2
- **RAM:** 15.6 GB
- **Accelerator:** NVIDIA Quadro P620 4 GB via CUDA (`ngl=99`); Vulkan also detected
- **llama.cpp asset đã tải:** `llama-b10488-bin-win-cuda-12.4-x64.zip`
- **Model đã dùng:** Qwen3.5 0.8B (`LAB_MODEL=qwen35-0.8b`)
- **Quantization:** Q4_K_M (primary) + UD-Q2_K_XL (compare)

**Chạy ở đâu:** laptop của tôi
_(Nếu dùng cloud fallback: nói rõ vì sao — RAM < 8 GB, setup fail, v.v. Không mất điểm.)_

**Setup story** (≤ 80 chữ): điều gì cần thay đổi để lab chạy trên máy bạn? Có bước
nào fail rồi phải workaround không?

Tôi chọn Qwen3.5 0.8B để giảm thời gian tải và chạy thí nghiệm dù máy đủ RAM cho
Gemma. `lab.ps1` ban đầu lỗi parse vì Windows PowerShell 5.1 đọc sai UTF-8 không BOM;
tôi lưu lại script dưới UTF-8 có BOM rồi setup thành công. CUDA offload được tự động
bật trên Quadro P620.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 44734 | 525 / 555 | 30.2 / 30.3 | 2420 / 2459 / 2459 | 33.1 |
| UD-Q2_K_XL | 0.39 | 4556 | 615 / 860 | 34.7 / 37.5 | 2784 / 3094 / 3094 | 28.8 |

**Quan sát** (≤ 60 chữ): 2-bit nhanh hơn bao nhiêu, và **có đáng không**? Bạn đã thử
hỏi cùng một câu trên cả hai (`make serve` vs `.venv/bin/python labs/02-serve/serve.py --compare`)
chưa? Chất lượng khác nhau thế nào?

Q2 nhỏ hơn 22% nhưng decode chậm hơn Q4 khoảng 13%; TTFT P50 cao hơn 17% và E2E
P50 cao hơn 15%. Với cùng prompt, Q4 ngắn gọn hơn; cả hai chưa định nghĩa hoàn toàn
đúng, nhưng Q2 vòng vo và lẫn Goodput@SLO với SLA. Vì vậy Q2 không đáng dùng trên máy này.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 0.92 | 8900 | 12000 | 14000 | 8.4 | 0.0% |
| 50 | 1.03 | 27000 | 49000 | 51000 | 28.9 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 1.12×
- **P95 tăng:** 4.08×
- **Effective concurrency ở 50 users:** 28.9 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): 3.91 / 4 slots

**Saturation reading** (≤ 80 chữ): server của bạn bão hoà ở đâu, và **bằng chứng nào**
thuyết phục bạn? Nếu P95 tăng nhanh hơn RPS thì phần latency thêm đó là queue time hay
compute time — bạn biết bằng cách nào? Nếu bạn phải nâng goodput@SLO, bạn sẽ đổi knob
nào **trước**, và vì sao knob đó?

Server đã có queue ở 10 users vì effective concurrency 8.4 vượt 4 slots, và bão hòa
nặng ở 50 users: throughput chỉ tăng 1.12× trong khi P95 tăng 4.08×. Peak 3.91/4 slots
và 46 deferred requests chứng minh latency tăng chủ yếu là queue time. Với SLO P95 ≤
15 giây, run 10 users đạt nhưng run 50 users không đạt. Tôi sẽ thử `--parallel 8`
trước vì bốn decode slots là giới hạn trực tiếp.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | localhost only | stub |
| N17 Data pipeline | in-memory list | stub |
| N18 Lakehouse | `TOY_DOCS` dictionary | stub |
| N19 Vector + features | keyword overlap, no embeddings/index | stub |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.1 ms
- llm: 6106.0 ms
- **stage chiếm nhiều nhất:** LLM (gần 100% của total)

**Reflection** (≤ 60 chữ): bottleneck ở đâu? Có khớp với kỳ vọng của bạn không? Nếu
phải giảm latency của pipeline này 2×, bạn sẽ tấn công vào đâu?

LLM là bottleneck đúng như kỳ vọng vì keyword retrieval trên corpus nhỏ chỉ tốn 0.1
ms. Muốn giảm latency pipeline 2×, tôi sẽ tối ưu LLM decode bằng cách giảm output-token
budget hoặc dùng serving hardware nhanh hơn; tối ưu retrieval gần như không ảnh hưởng tổng thời gian.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** đổi quantization từ UD-Q2_K_XL sang Q4_K_M

```
before:  28.8 tok/s (UD-Q2_K_XL)
after:   33.1 tok/s (Q4_K_M)
speedup: 1.15×
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

_Giải thích như đang nói với bạn ngồi cạnh. Bám vào **cơ chế**, không phải "vibes":
memory bandwidth? vector width? cache residency? scheduling? queueing? Nếu kết quả
**khác** với kỳ vọng từ deck — nói rõ, và giải thích vì sao. Grader thưởng điểm cho
lập luận đúng về một kết quả bất ngờ, hơn là một con số đẹp không được giải thích._

Kỳ vọng thông thường là Q2 nhỏ hơn sẽ decode nhanh hơn nhờ đọc ít byte từ bộ nhớ.
Nhưng Q4 chỉ 0.50 GB nên đã nằm trọn trong 4 GB VRAM của Quadro P620; giảm thêm 0.11
GB không thay đổi model residency hay loại bỏ PCIe transfer trong mỗi bước decode.

Trên CUDA backend này, chi phí unpack/dequantization của Q2 và mức tối ưu kernel cho
định dạng đó lớn hơn lợi ích bandwidth. Kết quả là TPOT P50 tăng từ 30.2 lên 34.7 ms
và throughput giảm từ 33.1 xuống 28.8 tok/s. Thread sweep cũng cho thấy 4–16 CPU
threads gần như phẳng, củng cố rằng thêm tài nguyên CPU không giải quyết đường chạy
GPU/dequantization này. Vì vậy Q4 là lựa chọn nhanh hơn và cũng trả lời tốt hơn.

---

## 6. Bonus  *(optional — tối đa 10 điểm)*

> Bỏ trống nếu không làm. Xem `docs/bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** Không làm bonus.

**Numbers:**

```
before:  N/A
after:   N/A
speedup: N/A
```

**Điều này nói lên gì mà deck chưa nói:**

Không áp dụng.

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

_(1–2 câu. Không bắt buộc, nhưng grader đọc hết.)_

Q2 nhỏ hơn nhưng lại chậm hơn Q4, còn tăng từ 4 lên 16 CPU threads gần như không cải
thiện throughput vì phần lớn tính toán đã được offload sang GPU.

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

Tôi sử dụng OpenAI Codex để đọc hướng dẫn lab, chẩn đoán lỗi encoding PowerShell,
giải thích số liệu benchmark/load test và hỗ trợ biên tập các phần nhận xét trong báo
cáo. Toàn bộ số liệu đo được tạo bằng các script của lab trên máy của tôi.
