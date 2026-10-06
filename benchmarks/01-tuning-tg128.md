# 01 - Tune: thread-count sweep

Model `Qwen3.5-0.8B-Q4_K_M.gguf` · host `Windows-AMD64` · llama.cpp `b10488`
CPU: **2 physical · 4 logical** cores · `ngl=0` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 4.5 | 62% |
| 2 | 7.1 | 100% |
| 4 | 6.2 | 86% |

**Best**: `-t 2` at 7.1 tok/s
**Slowest tested**: `-t 1` at 4.5 tok/s (1.60x spread)
**Against the physical-core default** (`-t 2`, 7.1 tok/s): 1.00x

Use this in your run:

```bash
LAB_N_THREADS=2 make bench
```

## Explanation

Điểm gãy nằm ở 2 threads, đúng bằng 2 nhân vật lý của i7-4500U: tốc độ tăng từ
4.5 tok/s ở 1 thread lên 7.1 tok/s ở 2 threads. Tăng tiếp lên 4 threads (2 luồng
logic trên mỗi nhân) lại giảm còn 6.2 tok/s, thấp hơn 13% so với mức tốt nhất.
Decode phải đọc trọng số model cho từng token; hai luồng logic trên cùng một nhân
không bổ sung thêm nhân hay băng thông bộ nhớ, mà chia sẻ tài nguyên thực thi và
cache, đồng thời tăng chi phí lập lịch. Vì vậy thêm luồng không tạo đủ công việc
hữu ích để bù phần tranh chấp tài nguyên. Đây là kết quả phù hợp với việc decode
bị giới hạn bởi tài nguyên bộ nhớ/nhân, dù sweep này không đo trực tiếp băng thông.
