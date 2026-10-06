# 01 - Measure: latency baseline

Model `Qwen3.5 0.8B` � host `Windows-AMD64` � llama.cpp `b10488`
Settings: `threads=2` `ngl=0` `ctx=2048`
`max_tokens=64` � warm-up discarded
Completed requests: `Q4_K_M` 10/10 � `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 8870 | 819 / 1381 | 84.3 / 170.5 | 5874 / 11853 / 11853 | 11.9 |
| UD-Q2_K_XL | 0.39 | 3704 | 1316 / 1893 | 82.6 / 125.0 | 6723 / 9192 / 9192 | 12.1 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` and `Q4_K_M` decode within 2% of each other here, for 0.11 GB difference on disk.

## Observation

Bản UD-Q2_K_XL decode 12.1 tok/s, chỉ nhanh hơn khoảng 1.7% so với 11.9 tok/s của Q4_K_M; file nhỏ hơn 0.11 GB (22%). Tuy nhiên Q2 có TTFT P50 1316 ms và E2E P50 6723 ms, cao hơn Q4 (819 ms và 5874 ms). Khi so sánh câu trả lời, Q2 kém đúng trọng tâm hơn. Vì vậy mức tiết kiệm dung lượng/tốc độ decode nhỏ không đáng nếu chất lượng quan trọng; phép đo này riêng lẻ không chứng minh memory bandwidth là giới hạn duy nhất.