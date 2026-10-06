# 02 - Continuous batching under load (u50)

Host `Windows-AMD64` · `--parallel 4` · 14 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 2.05 of 4 slots (51%) |
| `requests_processing` | 4 |
| `requests_deferred` | 46 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 668 |

Highest sampled value was **2.05 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Observation

Peak `n_busy_slots_per_decode` was **2.05 of 4** (51%), above 1 and therefore
evidence that multiple requests were sharing decode steps under load. The server
also reported 4 requests processing and up to 46 deferred. This does not equal
the **3.2** effective concurrency in `02-server-results.md`: the gauge is an
average of useful slots per decode step, while effective concurrency is
`RPS × average response time` for completed Locust requests and includes queue
time. They measure different things; the 60-second Locust sample was also small
(5 completed requests), so neither number should be read as a precise capacity
estimate. `kv_cache_usage_ratio` was not exported by build b10488.
