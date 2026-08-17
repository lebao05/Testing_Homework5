# Skill: JMeter Log Analysis & Report Generation

## Purpose

Help the AI analyze the JTL log produced by a JMeter run and write the performance report **directly as text**. The user runs JMeter themselves (after using `testplan-skill.md` to generate the test plan) and hands the AI the resulting `.jtl` file. The AI reads the CSV, computes the statistics in its own reasoning, and writes the markdown report — **no Python scripts, no helper executables**.

The skill supports:

- Aggregate statistics (avg, p50, p95, p99, min, max)
- Throughput calculations (TPS, TPM)
- Per-sampler breakdown by `label`
- Latency & connect-time breakdown
- Stability window (first 10 vs last 10 samples)
- Pass / fail assessment against the thresholds

> ⚠️ **No Python scripts.** The AI reads the JTL with the Read tool and computes the statistics in its own reasoning. No shell helpers, no `awk`, no `csvkit`.

---

## 1. Required Input

Before analyzing, the AI needs:

### The JTL file

- **Absolute path** to the `.jtl` (e.g. `D:\Testing\HW5\test-plans\load\results\load.jtl`)
- The `Aggregate Report` listener produces this file with the standard JMeter 5.6.3 column set (see §3).

### The original test plan

- The corresponding `.jmx` file (so the AI can confirm the workflow, the `label` prefixes, and the listener configuration).

### Test run metadata

- **Test run date** (YYYY-MM-DD) — usually today
- **Expected duration** in seconds (from the `ThreadGroup.duration` in the JMX)
- **Pass criteria** — error rate ≤ 1%, p95 ≤ ?, etc. (defaults: error rate ≤ 1%, all response times < 1000 ms unless the user specifies otherwise)

---

## 2. Column Schema

The JTL is a CSV with these columns (JMeter 5.6.3 default `SampleSaveConfiguration`):

```
timeStamp,elapsed,label,responseCode,responseMessage,threadName,dataType,success,
failureMessage,bytes,sentBytes,grpThreads,allThreads,URL,Latency,IdleTime,Connect,SampleCount
```

| Column | Meaning |
|---|---|
| `timeStamp` | Epoch milliseconds when the sample finished |
| `elapsed` | Total response time in **milliseconds** |
| `label` | Sampler label, e.g. `03 POST /api/login` |
| `responseCode` | HTTP status, e.g. `200`, `400`, `500` |
| `success` | `"true"` or `"false"` (string, not bool) |
| `failureMessage` | Empty when `success=true` |
| `grpThreads` | Active threads in the group at the moment of this sample |
| `Latency` | Time-to-first-byte in ms |
| `Connect` | TCP connect time in ms |
| `IdleTime` | Time spent idle since the last sample (ms) |

> ⚠️ **`success` is the string `"true"`, not the boolean.** Count `r.success == "true"`.

---

## 3. Statistics to Compute

After reading the JTL with the Read tool, compute everything in the AI's own reasoning — no Python.

### Overall

| Stat | Formula |
|---|---|
| Total samples | `len(rows)` |
| Duration (s) | `(rows[-1].timeStamp - rows[0].timeStamp) / 1000` |
| Success rate | `count(r.success == "true") / total * 100` |
| Failure count | `count(r.success == "false")` |
| Throughput (TPS) | `total / duration` |
| Throughput (TPM) | `TPS * 60` |
| Peak concurrent threads | `max(r.grpThreads)` |

### Per-sampler (group by `label`)

For each unique label, compute over the subset:

| Stat | Formula |
|---|---|
| `n` | `len(rows_label)` |
| `errors` | `count(r.success == "false")` |
| `error_rate` | `errors / n * 100` |
| `avg` | `sum(elapsed) / n` |
| `min` | `min(elapsed)` |
| `max` | `max(elapsed)` |
| `p50` | `sorted_vals[floor(n * 0.50)]` |
| `p95` | `sorted_vals[floor(n * 0.95)]` |
| `p99` | `sorted_vals[floor(n * 0.99)]` |
| TPS | `n / duration` |

> ⚠️ **The largest sample may have a label like `Aggregate Report`** (JMeter's own end-of-run summary). Filter those out — the AI should only report on the application samplers.

### Latency & connect breakdown

Across all rows (not per-sampler):

| | Avg | Min | Max |
|---|---|---|---|
| Response latency (`Latency` column) | `sum / n` | `min` | `max` |
| TCP connect (`Connect` column) | `sum / n` | `min` | `max` |

### Stability window

Sort rows by `timeStamp`. Compare:

| Window | Avg elapsed |
|---|---|
| First 10 samples (by timeStamp) | `sum(elapsed) / 10` |
| Last 10 samples | `sum(elapsed) / 10` |
| **Delta** | `(last - first) / first * 100` (%) |

---

## 4. Pass / Fail Assessment

The report's assessment table uses these default thresholds unless the user specifies otherwise:

| Criterion | Default pass | Source |
|---|---|---|
| Error rate | ≤ 1% | Computed |
| Avg response time | ≤ 500 ms | Computed |
| p95 response time | ≤ 1000 ms | Computed |
| p99 response time | ≤ 2000 ms | Computed |
| Max response time | ≤ 5000 ms | Computed |
| Throughput stability | `|delta| ≤ 20%` | Stability window |
| Peak concurrency | `peak ≥ num_threads * 0.95` | Peak vs configured VUs |

Mark each row with `✅ Pass` or `� Fail`. The conclusion paragraph summarises the overall pass / fail.

---

## 5. Report Template

The AI writes `report.md` (or `5.<N>.md`, etc.) directly with the Write tool. Replace every `<placeholder>` with the real numbers.

```markdown
# Performance Test Report — <workflow name>

## 5.1 Objective

<copy from assignment, e.g. "evaluate behavior under expected sustained workload — throughput, response time, error rate, CPU, memory">

## 5.2 Configuration

| Parameter | Value |
|---|---|
| **Test plan** | `<absolute path to .jmx>` |
| **Target** | `<base URL>` |
| **VUs** | <N> concurrent threads |
| **Ramp-up** | <S> seconds |
| **Duration** | <S> seconds (scheduler) |
| **Loop** | Infinite (-1), capped by duration |
| **Think times** | Step 1: N ms · Step 2: N ms · Step 3: N ms · Step 4: N ms |
| **Scenario** | <workflow> |
| **Data source** | `<CSV file>` (N rows, recycled with shareMode=currentThread) |
| **Listeners** | `<name>` (`<absolute path to .jtl>`) |

## 5.3 Result

**Test run date:** YYYY-MM-DD

| Metric | Value | Threshold / Pass Criterion |
|---|---|---|
| Total requests | <N> | — |
| Test duration | <S> s | ~<expected> s |
| Success rate | <X>% (N/N) | ≥ 99% |
| Failure count | <N> | 0 |
| Throughput (TPS) | <N> req/s | — |
| Throughput (TPM) | <N> req/min | — |
| Peak concurrent threads | <N> | Full ramp-up achieved |

**Per-sampler response time summary**

| Endpoint | Count | Avg | p50 | p95 | p99 | Max | TPS |
|---|---|---|---|---|---|---|---|
| 01 POST /api/forgot-password | … | … | … | … | … | … | … |
| 02 POST /api/reset-password  | … | … | … | … | … | … | … |
| 03 POST /api/login           | … | … | … | … | … | … | … |
| 04 GET /api/products?search= | … | … | … | … | … | … | … |
| 05 POST /api/cart            | … | … | … | … | … | … | … |

**Latency breakdown**

| | Avg | Min | Max |
|---|---|---|---|
| Response latency | … ms | … ms | … ms |
| TCP connect time | … ms | … ms | … ms |

**Stability window — first 10 vs last 10 samples**

| Window | Avg elapsed |
|---|---|
| First 10 samples | … ms |
| Last 10 samples  | … ms |
| **Delta** | … ms (±…%) |

**Assessment**

| Criterion | Result | Status |
|---|---|---|
| Error rate | <X>% | ✅ Pass / ❌ Fail |
| Avg response time | <X> ms | ✅ / ❌ |
| p95 response time | <X> ms | ✅ / ❌ |
| p99 response time | <X> ms | ✅ / ❌ |
| Max response time | <X> ms | ✅ / ❌ |
| Throughput stability | <desc> | ✅ / ❌ |
| Peak concurrency | <N> VUs achieved | ✅ / ❌ |

Close with a one-paragraph conclusion.
```

---

## 6. Listener Choice Reminder

| Listener | guiclass | When to use | Filename |
|---|---|---|---|
| Summary Report | `SummaryReport` | Quick sanity check, live-only | `""` (empty) |
| **Aggregate Report** | `StatVisualizer` | **Always** — this is the actual report | **absolute** path to JTL |
| View Results Tree | `ViewResultsFullVisualizer` | Debug only — never for long runs (RAM killer) | `""` (empty) |

For long runs (10 min+), set `<boolProp name="ResultCollector.error_logging">true</boolProp>` on View Results Tree so it only captures errors. Otherwise the JVM heap will explode.

> ⚠️ **`AggregateReportGui` does NOT exist in JMeter 5.6.3.** Use `StatVisualizer`. If the JMX uses `AggregateReportGui`, the JTL is still valid for analysis — the AI should proceed but flag the listener bug in the report's notes.

---

## 7. Analysis Procedure

1. **Locate the JTL.** The user provides the absolute path. If only a relative path is given, prepend the workspace root (`D:\Testing\HW5\`).
2. **Read the JTL with the Read tool.** It is a CSV — the AI parses the column header line and every data row directly from the read output.
3. **Filter out non-application samples.** Drop rows whose `label` is `Aggregate Report`, `Summary Report`, or any View Results Tree internal marker. Only keep the sampler labels (e.g. `01 POST /api/forgot-password`).
4. **Compute overall stats** per §3.
5. **Group by `label`** and compute per-sampler stats per §3.
6. **Compute latency / connect breakdown** across all rows per §3.
7. **Compute the stability window** (first 10 vs last 10 by `timeStamp`) per §3.
8. **Assess pass / fail** per §4.
9. **Write the report** as a single markdown file using the §5 template. Every `<placeholder>` is replaced with the real number. The conclusion paragraph ties the assessment together in one sentence.

---

## 8. Common JTL Pitfalls

- **Empty `responseCode`** — usually means the sampler never connected. Flag as a separate failure type, not as an HTTP error.
- **`success=false` with `responseCode=200`** — the `ResponseAssertion` failed (e.g. expected 200 but the SUT returned 400 with the assertion still passing on code). Read `failureMessage`.
- **`success=false` with `responseCode=4xx`** — genuine client error (e.g. duplicate email on `register`). Distinguish from server errors.
- **Single huge outlier** — usually the first sample (JVM warm-up) or a connection-pool exhaustion. Report it in `Max` but don't let it dominate the assessment — note it in the conclusion.
- **Total TPS doesn't match the per-sampler sum** — JMeter records subresults separately when assertions are present. Filter subresults before summing (`SampleCount > 0` and `subresults` handling).

---

## 9. What the AI Does NOT Do in This Skill

- ❌ Run JMeter
- ❌ Start or stop the SUT
- ❌ Generate the JMX (covered by `testplan-skill.md`)
- ❌ Write Python scripts to compute statistics
- ❌ Use shell helpers (`awk`, `cut`, `csvkit`, …) to process the JTL

The AI reads the JTL CSV with the Read tool, computes everything in its own reasoning, and writes the markdown report directly.

---

## 10. Versioning & Provenance

This skill was distilled from the three JMX test plans in `test-plans/` (Load, Stress, Spike) and the lessons learned while iterating on them. Every ⚠️ callout in this document is a real bug that was hit during the project — do not skip them.
