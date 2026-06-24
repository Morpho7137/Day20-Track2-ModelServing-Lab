# Bonus — Thread sweep

Model: `qwen2.5-1.5b-instruct-q4_k_m.gguf`  ·  GPU layers: `0`

| threads | tg128 (tok/s) |
|---:|---:|
| 1 | 9.0 |
| 2 | 14.8 |
| 3 | 17.4 |
| 6 | 19.8 |
| 12 | 9.4 |
| 24 | 2.8 |

**Best**: `-t 6` at 19.8 tok/s.

Look at the curve. If it peaks around your **physical** core count and drops as you go higher, that's the memory-bandwidth ceiling: extra threads fight over the same memory channels and slow each other down.
