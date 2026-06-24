# Track 02 Server Results

Native `llama-server` was built on WSL ext4 and exposed `/metrics` on `:8080`.

## Load test summary

| Concurrency | Total RPS | TTFB P50 (ms) | E2E P95 (ms) | E2E P99 (ms) | Failures |
|--:|--:|--:|--:|--:|--:|
| 10 | 0.31 | 25000 | 31000 | 31000 | 0 |
| 50 | 0.46 | 26000 | 41000 | 41000 | 0 |

## Metrics summary

- Peak `llamacpp:n_busy_slots_per_decode`: 3.91063
- Peak `llamacpp:requests_processing`: 4
- Peak `llamacpp:requests_deferred`: 46

## Notes

- Metrics CSV: `benchmarks/02-server-metrics-50.csv`
- The server saturated all four native slots under concurrency 50 and queued additional work instead of failing.
