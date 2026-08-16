# Spike Test Plan - EShop (`{StudentID}_Spike_{YYYYMMDD}.jmx`)

Scenario (one iteration per VU; fire-hose style, no think times):

`forgot-password` -> `reset-password` -> `login` -> `search-product` -> `add-to-cart`

The plan fires two rapid high-VU bursts separated by a cool-down:

| Phase     | ThreadGroup                  | Threads | Ramp | Duration | Startup delay | Notes                                                                                  |
|-----------|------------------------------|---------|------|----------|---------------|----------------------------------------------------------------------------------------|
| Burst A   | `Spike-A-200VU-30s`          | 200     | 5s   | 30s      | 0s            | First spike; ramp ends at t=5s, runs through t=35s                                       |
| (silence) | —                            | —       | —    | —        | —             | **No Cooldown ThreadGroup.** JMeter top-level ThreadGroups start in parallel; you cannot "wait for the previous TG to finish" with a dummy TG. The cooldown is implemented entirely as a startup delay on Spike-B below. |
| Burst B   | `Spike-B-300VU-30s`          | 300     | 3s   | 30s      | 80s           | Startup delay = `spikeA_ramp + spikeA_dur + cooldown` = 5 + 30 + 45 = 80 s. Note: `ThreadGroup.delay` is a plain integer (longProp), so if you change `spikeA_dur` or `cooldown` you must re-sum and update the literal value on Spike-B's `<longProp name="ThreadGroup.delay">`. Harder spike than A; used to validate recovery between bursts. |

**Total wall-clock time:** 5s + 30s + 45s + 3s + 30s = **~113 s** (ramp_A + dur_A + cool + ramp_B + dur_B), plus ramp/down overhead.

> **Why no separate Cooldown ThreadGroup?** I had one in an earlier version and learned the hard way that JMeter treats top-level ThreadGroups as parallel-by-default — listing "A, Cooldown, B" in the tree doesn't make them sequential, and a dummy TG with `delay=0` finishes in under a second (1 iteration × 500 ms timer), not the 45 s its `duration` field nominally specifies. The clean approach is **just give Spike-B a startup delay equal to (Spike-A's ramp + duration + desired cooldown)**.

## Listener (unique to this plan)

**Aggregate Report** (Summary Report is reserved for the Load plan, View Results Tree for any debugging only).

Two listeners are wired: one inside each spike ThreadGroup, writing to `results/spike-A.jtl` and `results/spike-B.jtl` so each burst can be analyzed on its own.

## Files in this folder

- `{StudentID}_Spike_{YYYYMMDD}.jmx` — main test plan
- `strike_users.csv` — the data-driven source for the spike plan, **independent from `../load/load_users.csv`**

> **Why a different CSV?** The Strike plan needs more rows than the Load plan to keep VU -> row binding stable under 300 concurrent threads without constant recycling mid-burst. `strike_users.csv` has **500 data rows** that cycle through the same 50 seeded `perf-user-NNN@load.com` accounts (10 reads per user during Burst B), while rotating through a much wider keyword set. That extra keyword variety stresses the SQLite `LIKE '%kw%'` query in different cache paths.

### `strike_users.csv` keyword mix

500 rows, distributed roughly:

| Bucket                              | Count | %    | Behaviour against seeded catalog                                                                                                                                                              |
|-------------------------------------|-------|------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Hits (substrings of product names)  | 150   | 30%  | Returns >=1 row from `/api/products?search=`. Examples: `iPhone`, `Pro`, `Max`, `S24`, `M3`, `M`, `15`, `2`, `Phone`, `Keychron`.                                                            |
| Misses (real-shopper terms)         | 300   | 60%  | Returns `[]`. Examples: `ao thun`, `giay the thao`, `laptop`, `dien thoai`, `tai nghe`, `dong ho`, etc. — same Vietnamese terms as the Load plan plus more, so the empty-result path is also stressed.|
| LIKE-wildcards and SQL-noise chars  | 50    | 10%  | `_`, `%`, `'`, `"`, `;`, `--`. The `_` is a real `LIKE` wildcard (matches any single char); the others should each return `[]` without breaking the SQL.                                       |

Total unique keywords: 73. Password column is constant `Newpass123!` to match the Load plan.

### Known limit: CSV vs. seeded-user count

The CSV currently cycles through **only 50** unique accounts (`perf-user-001..050@load.com`), but **Spike-B runs 300 concurrent VUs**. With `shareMode=currentThread` and `recycle=true`, every VU always gets a row, but during Burst B ~6 threads land on the same account at the same time. The flow stays safe (the `forgot -> reset -> login` chain happily resets the password and re-extracts a JWT for each thread), but you'll see account-lockout contention (`login_attempts >= 3` -> 3-min lock) inside Spike-B because some accounts log in many times within the 30-second burst.

**If you want clean per-VU authentication behavior** (each VU holds its own account, no cross-thread lockouts), grow `database.js`'s seeder from 50 to ≥300 perf-users AND extend this CSV accordingly. That is a backend-side change and was intentionally **not** done in this patch.

## Workflow mapped to endpoint groups

| Step | Endpoint                          | Group         | Why                                       |
|------|-----------------------------------|---------------|-------------------------------------------|
| 1    | `POST /api/forgot-password`       | Auth-heavy    | trigger OTP; response carries `resetToken` in the JSON body — the JSONExtractor parses it directly |
| 2    | `POST /api/reset-password`        | Auth-heavy    | exchange OTP for a new password           |
| 3    | `POST /api/login`                 | Auth-heavy    | gate before JWT-bearing call              |
| 4    | `GET /api/products?search=`       | Read-heavy    | SQLite SELECT with WHERE                  |
| 5    | `POST /api/cart`                  | Transactional | requires JWT + real product               |

## Key parameters (and why)

- **Spike-A**: `Threads=200, Ramp=5s, Duration=30s, delay=0`. Hits ~40 RPS as a warm spike. Sufficient to flatten any in-process pool.
- **Cooldown**: implemented as **`ThreadGroup.delay = 80` on Spike-B** (no separate TG). 45 s covers the server's 30-second login-attempt lockout window (`server.js` line 56 — `login_attempts` cycles every 30 s) plus a safety margin.
- **Spike-B**: `Threads=300, Ramp=3s, Duration=30s, delay=80`. Harder spike than A. Expected to surface any leftover state from A.
- All four `thinkStepN` are `0` — the spike is a fire-hose, no human pacing. Each burst is its own ramp-up so the scheduler ramps fast but not instantly.
- `token guard`: one `IfController` per spike, wrapping `04` and `05`. If `03 POST /api/login` somehow fails to return a JWT (e.g. due to server-side saturation), the cart step is skipped instead of failing with `401`.

## Pre-run checklist

1. Make sure the backend is running **with a freshly-seeded DB**:
   ```
   cd eshop-sut\backend
   node database.js       # drops + re-creates every table, seeds 50 perf users
   node server.js         # http://localhost:3000
   ```
2. Rename `{StudentID}_Spike_{YYYYMMDD}.jmx` to your real StudentID + today's date before running.
3. The `strike_users.csv` next to the JMX is independent — no need to copy anything from the Load folder.

## Run

```
cd test-plans\strike
jmeter -n -t {StudentID}_Spike_20260816.jmx -e -o results\spike-html
```

The two `Aggregate Report` listeners inside the JMX automatically append their samples to `results/spike-A.jtl` and `results/spike-B.jtl` (relative to the JMX's working directory).

Total wall-clock time: 5 + 30 + 45 + 3 + 30 = ~113s, plus ramp/down overhead.

## Reading the report

- Open `results/spike-html/index.html` for the HTML dashboard.
- The Aggregate Report listener in the GUI shows: `# Samples, Average, Median, 90% Line, 95% Line, 99% Line, Min, Max, Error %, Throughput, KB/sec`. Compare Burst A vs Burst B's rows on the same label to see if the system degraded across the spike.
- Compare against the Load plan's `results/load.jtl` to confirm the Spike plan is hitting the server harder (higher Throughput per sampler).

## Anti-cheat reminder

- Filename must be `{StudentID}_Spike_{YYYYMMDD}.jmx` exactly.
- Raw `.jtl` files (`spike-A.jtl`, `spike-B.jtl`) are required deliverables; do not only attach the Aggregate Report screenshot.