# Stress Test Plan - EShop

HW05 Stress test for the EShop backend (Node.js + SQLite).

## What "Stress" means here

Stress testing is **sustained load above the system's normal capacity**, run long enough for *gradual* failure modes to appear. This is fundamentally different from:

- **Load** (`../load/`) — measures baseline capacity at expected traffic (50 VU for 5 min). Answers "how does the system perform normally?"
- **Spike** (`../strike/`) — measures elasticity / recovery (200 VU → cool-down → 300 VU, 30 s each). Answers "what happens when traffic suddenly jumps?"
- **Stress** (this plan) — measures stability *above* the normal capacity over time (200 VU sustained for 10 min). Answers "what breaks first, and how does it break?"

Stress testing exists to expose:

- **Memory leaks** — response time degrades monotonically over the soak
- **Resource exhaustion** — file descriptor / connection pool / heap pressure
- **Slow degradation** — sqlite WAL file growth, WAL-checkpoint backpressure, OTP-token-table bloat
- **Breaking point** — the exact VU count where error rate first climbs
- **Recovery behavior** — does the system return to clean state after the soak, or does it stay degraded?

## Profile (defaults — overridable via `-J`)

| Property      | Default | Meaning                                                   |
|---------------|---------|-----------------------------------------------------------|
| `stress_users`| 200     | Peak concurrent virtual users. ~4× Load plan's 50 VU.      |
| `stress_ramp` | 30 s    | Ramp-up time. Linear ramp from 0 → 200 VU.                |
| `stress_dur`  | 600 s   | Soak time at peak. 10 minutes is long enough for slow leaks to surface. |
| `thinkStep1..4`| 0 ms   | No artificial think time — maximum pressure on the server. |

> The 4× Load ceiling is intentional: 50 VU is the verified comfortable load, 200 VU pushes past it. If the breaking point is *below* 200, the soak will show climbing error rate; if it's *above* 200, you'll see clean sustained throughput and can re-run with a higher peak.

## Sampler flow

Same 5-step flow as Load and Spike plans, inside a `GenericController` named "Stress workflow", wrapped in an `IfController` so we don't run cart requests on auth failures:

1. `POST /api/forgot-password` — JSON Extractor for `resetToken`
2. `POST /api/reset-password` — Response Assertion: 200 OK
3. `POST /api/login` — Response Assertion: 200 OK + JSON Extractor for `token`
4. *If `token` is not `NOT_FOUND`*:
   1. `GET /api/products?search=${search_keyword}` — JSON Extractor for `product_id`
   2. `POST /api/cart` — Authorization header `Bearer ${token}`, Response Assertion: 2xx

Listeners: `Aggregate Report` (`StatVisualizer` — verified class name for JMeter 5.6.3), writes to `results/stress.jtl`.

## Files in this folder

- `{StudentID}_Stress_{YYYYMMDD}.jmx` — main test plan
- `stress_users.csv` — the data-driven source for this plan, **independent from `../load/load_users.csv` and `../strike/strike_users.csv`**

> **Why a different CSV?** Stress runs 200 VU sustained for 10 min with `loops=-1` — each VU runs the 5-sampler flow as many times as it can in those 10 minutes. With 1000 data rows cycling through 50 seeded accounts, each `perf-user-NNN@load.com` is reused ~20 times during the 30 s ramp-up alone and ~250 times during the full 10 min soak. That heavy reuse is **exactly the point** of stress testing: hit the same DB rows repeatedly to surface contention, token-table bloat, and DB-row hot spots that Load/Spike never see.

### `stress_users.csv` keyword mix

1000 rows, distributed roughly:

| Bucket                              | Count | %    | Behaviour against seeded catalog                                                                                                                                                              |
|-------------------------------------|-------|------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Hits (substrings of product names)  | 300   | 30%  | Returns >=1 row from `/api/products?search=`. Examples: `iPhone`, `Pro`, `Max`, `S24`, `M3`, `M`, `15`, `2`, `Phone`, `Keychron`.                                                            |
| Misses (real-shopper terms)         | 600   | 60%  | Returns `[]`. Examples: `ao thun`, `giay the thao`, `laptop`, `dien thoai`, `tai nghe`, etc. — same Vietnamese terms as Load and Strike, stressing the empty-result LIKE path during the soak.   |
| LIKE-wildcards and SQL-noise chars  | 100   | 10%  | `_`, `%`, `'`, `"`, `;`, `--`. The `_` is a real `LIKE` wildcard (matches any single char); the others should each return `[]` without breaking the SQL.                                       |

Total unique keywords: 73. Password column is constant `Newpass123!` to match the Load and Spike plans.

## Pre-run checklist

1. Make sure the backend is running **with a freshly-seeded DB**:
   ```
   cd eshop-sut\backend
   node database.js       # drops + re-creates every table, seeds 50 perf users
   node server.js         # http://localhost:3000
   ```
2. (Optional) Capture a baseline of the SQLite file size before the run:
   ```
   ls -la eshop-sut\backend\data\*.db*
   ```
3. Rename `{StudentID}_Stress_{YYYYMMDD}.jmx` to your real StudentID + today's date before running.
4. `stress_users.csv` next to the JMX is independent — no need to copy anything from the Load or Strike folders.

## Run

```
cd test-plans\stress
jmeter -n -t {StudentID}_Stress_20260816.jmx -l results\stress.jtl -e -o results\stress-html
```

The `Aggregate Report` listener inside the JMX automatically appends its samples to `results/stress.jtl` (relative to the JMX's working directory).

Total wall-clock time: 30 s ramp + 600 s soak + ~5 s ramp-down = **~10 min 35 s**, plus ramp/down overhead.

## What to look for in the results

- **Climbing error rate over time** — system passing first minute, failing later. Memory leak / FD exhaustion.
- **Response time slope** — p95 / p99 climbing across the soak but stable at the start. Same.
- **Distribution across the 5 samplers** — which step slows down first? `/api/forgot-password` is a DB insert, `/api/reset-password` is a DB read+write, `/api/login` is a DB read, `/api/cart` is an in-memory op (no DB). The auth-DB steps should be the canary.
- **Connection-refused / ECONNRESET patterns** — backend is refusing connections → likely Node.js event loop saturated or heap full.
- **WAL file growth** — check `data/` folder mid-soak; if `*.db-wal` grows unbounded, the checkpoint isn't running.

## Comparison with Load and Spike

| Plan   | VU peak  | Duration | Profile     | What it answers                          |
|--------|----------|----------|-------------|-------------------------------------------|
| Load   | 50       | 5 min    | sustained   | Baseline capacity at normal traffic       |
| Spike  | 200→300  | ~2 min   | 2 bursts    | Elasticity / recovery under sudden jumps  |
| Stress | 200      | 10 min   | sustained   | Stability above capacity over time        |

Run them in this order: **Load → Spike → Stress**. Load establishes the baseline, Spike shows the burst behavior, Stress pushes past the baseline to expose slow failures.