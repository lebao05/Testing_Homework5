# Load Test Plan - EShop (`{StudentID}_Load_{YYYYMMDD}.jmx`)

Scenario (one iteration per VU, repeated for 5 minutes):  
`forgot-password` -> `reset-password` -> `login` -> `search-product` -> `add-to-cart`

## Listener (unique to this plan)

**Summary Report** (Aggregate Report is reserved for the Stress plan, View Results Tree for the Spike plan).

## Files in this folder

- `{StudentID}_Load_{YYYYMMDD}.jmx` — main test plan
- `load_users.csv` — the data-driven source for the main plan (consumed one row per VU, recycled)

## Seed data is handled by `database.js`

The 50 `perf-user-NNN@load.com` accounts are created by `eshop-sut/backend/database.js` (the loop inside `initDatabase()` that calls `node database.js`). Run it **once before the Load run**:

```
cd eshop-sut\backend
node database.js       # drops + re-creates every table and seeds 50 perf users
node server.js         # http://localhost:3000
```

The seed password (`Seed12345!`) is **never used** during the Load run — the scenario resets it via `POST /api/reset-password` to `Newpass123!` (the `new_password` column in `load_users.csv`) before logging in.

> If `database.js` is run twice in a row, that's fine — it `DROP TABLE`s first, so the DB ends up identical.

## Workflow mapped to endpoint groups

| Step | Endpoint                                 | Group         | Why                                       |
|------|------------------------------------------|---------------|-------------------------------------------|
| 1    | `POST /api/forgot-password`              | Auth-heavy    | trigger OTP; response carries `resetToken` in the JSON body — the JSONExtractor parses it directly |
| 2    | `POST /api/reset-password`               | Auth-heavy    | exchange OTP for a new password          |
| 3    | `POST /api/login`                        | Auth-heavy    | gate before JWT-bearing call             |
| 4    | `GET  /api/products?search=`             | Read-heavy    | SQLite SELECT with WHERE                  |
| 5    | `POST /api/cart`                         | Transactional | requires JWT + real product               |

## Key parameters (and why)

- `Number of Threads = 50`, `Ramp-up = 30s`, `Duration = 300s` — light, steady load to establish baseline.
- `Same user on next iteration = true` — each VU reuses its CSV row, no leakage across iterations.
- Think times: 2s/5s/3s/5s/0-1s (jitter). Avoids artificial synchronization on the lockout window.
- `CSV share mode = currentThread` — predictable VU->row binding.
- `token guard`: one `IfController` ("If token missing, skip cart") wraps samplers 04/05. The `resetToken` extraction is no longer guarded because `/api/forgot-password-token` is a test-only endpoint that always returns the OTP per the SUT's `devHint`.

## Pre-run checklist

1. Make sure the backend is running:
   ```
   cd eshop-sut\backend
   node database.js       # drops + re-creates every table, seeds 50 perf users
   node server.js         # http://localhost:3000
   ```
2. Rename `{StudentID}_Load_{YYYYMMDD}.jmx` to your real StudentID + today's date before running.
3. Reset any 30-second login lockout state from prior runs:
   - Wait 30 seconds, or
   - Re-seed the DB (`node database.js`).

## Run

```
cd test-plans\load
jmeter -n -t {StudentID}_Load_20260816.jmx -l results\load.jtl -e -o results\load-html
```

At the 2-minute mark, take a screenshot of:
- Task Manager (Performance tab, picking the `node.exe` process running `server.js`)
- The CLI window showing the Summary Report

Then `dxdiag` for the hardware spec (required deliverable).

## Collector listening

Save run logs as `results/load.jtl` (CSV — required for the AI analysis task).
The HTML dashboard goes to `results/load-html/`.

## Anti-cheat reminder

- Filename must be `{StudentID}_Load_{YYYYMMDD}.jmx` exactly.
- Raw `.jtl` is attached; do not only attach the Summary Report screenshot.
- Hostname of the `dxdiag` must match the prior HWs.
