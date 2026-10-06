# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=2` ·
`ngl=0`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 6 | 0.12 | 30000 | 49000 | 49000 | 3.8 | 0.0% |
| 50 | 5 | 0.11 | 47000 | 47000 | 47000 | 4.0 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **0.87x** (17% of linear) |
| P95 latency | **0.96x** |
| Effective concurrency at 50 users | 4.0 vs `--parallel 4` slots (occupancy/slot ratio 1.00) |

**Queueing observed; load-scaling result is inconclusive.** The 50-user run had
effective concurrency 4.0 against 4 slots, but only 5 requests completed. Its
0.87x RPS ratio and 0.96x P95 ratio are too noisy to locate the saturation point
or compare latency reliably. The concurrent metrics sample independently showed
all 4 requests processing and up to 46 deferred, which is direct evidence that
requests queued during the 50-user run.

Since only 6 and 5 requests completed in the two runs, the measured P95 ratio
does not establish whether queueing or compute caused the difference. A longer
run is needed before estimating goodput at a chosen SLO.

> **Small sample.** Only 5 requests completed in the
> shorter run, so these percentiles are indicative rather than solid. Note also that
> locust averages only *completed* requests: when the run ends with requests still
> queued, effective concurrency is an **under**-estimate. Treat both throughput
> scaling and concurrency as indicative, not capacity estimates; run longer
> (`-t 3m`) for firmer numbers.

## Saturation reading

Metrics during the 50-user run showed all 4 slots processing and `requests_deferred`
peaking at 46, so queueing occurred. The Locust runs completed only 6 and 5 requests;
their 0.87x RPS ratio, 0.96x P95 ratio and 4.0 effective concurrency are too noisy
to locate saturation or attribute P95 to queue versus compute. I would first test a
shorter output-token limit to release decode slots sooner, then validate with a longer run.
