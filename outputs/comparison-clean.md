# CLEAN comparison: managed loop vs true local loop

Same T1-T3 tasks. N=8/cell. MANAGED = provider runs the loop in its sandbox. LOCAL = I run the loop here (base API per step + local Python execution, free CPU). Cost = token estimate at tier rates (+ OpenAI code_interpreter $0.03/container; Anthropic-managed cache pricing). Caching reported as-used: managed auto-caches its large agent context; local context is lean.

| Provider | Arm | N | Pass% | Median latency | Avg cost | % tool |
|---|---|---|---|---|---|---|
| anthropic | local | 24 | 100% | 2.38s | $0.00225 | 100% |
| anthropic | managed | 24 | 100% | 6.52s | $0.00250 | 67% |
| google | local | 24 | 100% | 2.54s | $0.00079 | 100% |
| google | managed | 24 | 100% | 12.14s | $0.00416 | 62% |
| openai | local | 24 | 100% | 5.61s | $0.00066 | 100% |
| openai | managed | 24 | 100% | 3.74s | $0.01054 | 33% |

## Managed vs local (per provider)

| Provider | managed $ | local $ | cost x | managed lat | local lat | lat x |
|---|---|---|---|---|---|---|
| anthropic | $0.00250 | $0.00225 | 1.1x | 6.52s | 2.38s | 2.7x |
| google | $0.00416 | $0.00079 | 5.3x | 12.14s | 2.54s | 4.8x |
| openai | $0.01054 | $0.00066 | 16.1x | 3.74s | 5.61s | 0.7x |

**Total estimated cost: $0.5014**