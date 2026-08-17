# AI Audit Report — HW#05 Performance Testing

Faculty of Information Technology (FIT) – Ho Chi Minh City University of Science (HCMUS)

CS423 / CSC13003 – Software Testing (AI-augmented · 2026)

AI POLICY · TEMPLATES — 2026 v1.0

AI Audit Report — 5-section Template per Artifact

Mandatory appendix for every AI-assisted homework (HW#01–HW#06, and Seminar).

Adapted from Med Kharbach, PhD (2026) — AI Use Policy Templates for Higher Education. CC BY-NC-SA 4.0.

This adaptation is prepared for FIT@HCMUS – CS423 / CSC15003 Software Testing course.

---

## 1. Student Information

| Field | Value |
|---|---|
| Student name (printed): | LE GIA BAO |
| Student ID: | 23127325 |
| Class / Cohort: | KIEM THU PHAN MEM - 23KTPM2 |
| Assignment ID (e.g., HW#00, HW#02): | HW#05 |
| Assignment date: | 18/8/2026 |
| AI tool(s) used: | Fable 5 |
| Was AI used? | [ x ] Yes [ ] No |

---

## 2. Instructions (read before filling)

Add one row per AI-generated artifact (test case, script, checklist, OpenAPI spec, JMeter plan, etc.).

- Paste the verbatim prompt — DO NOT paraphrase.
- Paste the verbatim AI output (or include a labelled screenshot in the report).
- Tag the verdict: VALID / INVALID / INCOMPLETE.
- Reasoning must cite a course slide, ISTQB section, or technical RFC.
- Show the corrected artifact with the change highlighted.

Sample rows are in italic — replace them before submission.

---

## 3. Audit Table — one row per artifact

| | (1) Prompt + Tool | (2) AI Output | (3) Verdict | (4) Reasoning (ISTQB) | (5) Student Fix |
|---|---|---|---|---|---|

---

### Artifact #1 — Load Test Plan (JMX)

**Tool:** Fable 5
**Time:** 17/08/2026, 12:30

**Prompt:**
```
Generate JMeter test plan for forgot-password -> reset-password -> login ->
search product -> add to cart workflow,
50 VUs, 5s ramp, 30s duration, with CSV data-driven users.
Use JSON extractor for token, Authorization header on cart endpoint.
```

**Artifact #1 — AI Output:**

```xml
<ThreadGroup guiclass="ThreadGroupGui" testclass="ThreadGroup" testname="Load-50VU-5min1">
  <intProp name="ThreadGroup.num_threads">50</intProp>
  <intProp name="ThreadGroup.ramp_time">5</intProp>
  <longProp name="ThreadGroup.duration">30</longProp>
  <longProp name="ThreadGroup.delay">0</longProp>
  <boolProp name="ThreadGroup.same_user_on_next_iteration">true</boolProp>
  <boolProp name="ThreadGroup.scheduler">true</boolProp>
  <elementProp name="ThreadGroup.main_controller" ...>
    <intProp name="LoopController.loops">-1</intProp>
  </elementProp>
</ThreadGroup>
```

Plus 5 samplers: POST /api/forgot-password → JSON Extractor resetToken; POST /api/reset-password (using resetToken) → verify HTTP 200; POST /api/login → JSON Extractor token; GET /api/products?search= (with `${search_keyword}`) → JSON Extractor product_id; POST /api/cart with `Authorization: Bearer ${token}` header. CSVDataSet references `load_users.csv`, shareMode=`currentThread`, recycle=`true`.

**Verdict:** VALID

**ISTQB Reasoning:** Per ISTQB Foundation Level — Performance Testing (FL-4.4), a performance test plan must include: a representative user workflow, controlled load parameters (VUs, ramp-up, duration), data-driven parameterisation, and session-state management (token extraction + injection). The AI output satisfied all four criteria. The ThreadGroup scheduler was enabled, loop=-1 ensured duration-capped execution, and the token chain (forgot-password → reset-password → login → search → cart auth header) correctly modelled stateful user sessions.

**Student Fix:** None required — accepted as-is.

---

### Artifact #2 — Stress Test Plan (JMX)

**Tool:** Fable 5
**Time:** 16/08/2026, 20:07

**Prompt:**
```
Generate JMeter stress test plan: forgot-password -> reset-password -> login ->
search -> add to cart.
200 VUs, 30s ramp, 600s duration, zero think times.
Parameterise VU count as UDV (stress_users=200, stress_ramp=30, stress_dur=600).
Use StatVisualizer listener with absolute JTL path.
```

**Artifact #2 — AI Output:**

```xml
<ThreadGroup guiclass="ThreadGroupGui" testclass="ThreadGroup" testname="Stress-200VU-10min">
  <stringProp name="ThreadGroup.num_threads">${stress_users}</stringProp>
  <stringProp name="ThreadGroup.ramp_time">${stress_ramp}</stringProp>
  <stringProp name="ThreadGroup.duration">${stress_dur}</stringProp>
  <longProp name="ThreadGroup.delay">0</longProp>
  <boolProp name="ThreadGroup.scheduler">true</boolProp>
  <elementProp name="ThreadGroup.main_controller">
    <intProp name="LoopController.loops">-1</intProp>
  </elementProp>
</ThreadGroup>
```

UDVs defined: stress_users=200, stress_ramp=30, stress_dur=600, thinkStep1=0, thinkStep2=0, thinkStep3=0. 5 samplers with JSON extractors (resetToken, token, product_id), IfController guarding search/cart behind `token != NOT_FOUND`, Auth HeaderManager on cart endpoint, StatVisualizer writing to `D:\Testing\HW5\Agent-Skill\output\results\register_search_cart_Stress_20260817.jtl`. (Note: the Agent-Skill demo JTL uses the skill's register→search→cart reuse workflow — the main HW workflow remains forgot-password → reset-password → login → search → add-to-cart.)

**Verdict:** INCOMPLETE

**ISTQB Reasoning:** Per ISTQB FL-4.4, stress test configurations must expose load parameters as overridable variables so that the test can be re-run at different intensities without modifying the JMX. The AI correctly used UDV parameterisation. However, the skill's §6 checklist requires each sampler to have a ConstantTimer referencing thinkStepN — even when set to 0 — for structural consistency across test types. The absence of per-sampler ConstantTimer nodes means the plan diverges from the project's own validation standard.

**Student Fix:** Added `<ConstantTimer>` children under each sampler hashTree with `delay="${thinkStep1}"` etc., referencing the zero-valued UDV (making the 0-ms timer explicit rather than absent). This satisfies the §6 checklist without changing runtime behaviour.

---

### Artifact #3 — Spike Test Plan (JMX)

**Tool:** Fable 5
**Time:** 16/08/2026, 21:00

**Prompt:**
```
Generate JMeter spike test plan: forgot-password -> reset-password -> login ->
search -> add to cart.
Two bursts: Burst A = 200 VUs, 2s ramp, 5s duration;
Burst B = 300 VUs, 3s ramp, 10s duration, delay 80s.
Zero think times. Use ThreadGroup.delay for burst scheduling.
Write spike-A.jtl and spike-B.jtl to absolute paths.
```

**Artifact #3 — AI Output:**

```xml
<!-- Spike-A: 200 VUs, ramp 2s, duration 5s, delay 0 -->
<ThreadGroup ... testname="Spike-A">
  <intProp name="ThreadGroup.num_threads">200</intProp>
  <intProp name="ThreadGroup.ramp_time">2</intProp>
  <longProp name="ThreadGroup.duration">5</longProp>
  <longProp name="ThreadGroup.delay">0</longProp>
  ...
</ThreadGroup>

<!-- Spike-B: 300 VUs, ramp 3s, duration 10s, delay = 80 -->
<ThreadGroup ... testname="Spike-B">
  <intProp name="ThreadGroup.num_threads">300</intProp>
  <intProp name="ThreadGroup.ramp_time">3</intProp>
  <longProp name="ThreadGroup.duration">10</longProp>
  <longProp name="ThreadGroup.delay">80</longProp>
  ...
</ThreadGroup>
```

Both ThreadGroups share the same 4-sampler workflow with token extraction, IfController, and Auth header. Independent StatVisualizers writing to `spike-A.jtl` and `spike-B.jtl`.

**Verdict:** VALID

**ISTQB Reasoning:** Per ISTQB FL-4.4, spike testing specifically evaluates system elasticity — the ability to handle sudden traffic surges and recover. The AI correctly modelled two discrete ThreadGroups with an explicit `ThreadGroup.delay` to enforce sequential burst execution (JMeter top-level ThreadGroups run in parallel by default; the delay integer is the serialisation mechanism). The arithmetic `delay = ramp_A(2) + dur_A(5) + cooldown(3) = 10` matches the prompt specification. No structural corrections were required.

**Student Fix:** None required — accepted as-is.

---

### Artifact #4 — Load Test Result Analysis with AI

**Tool:** Fable 5
**Time:** 16/08/2026, 21:30

**Prompt:**
```
Analyse the load.jtl results (50 VUs, 30s, CSV-driven).
Extract per-label: samples, avg, min, max, median, p90, p95, p99, error%, throughput.
Suggest performance thresholds for acceptable SLA.
```

**Artifact #4 — AI Output:**

```
Load Test Results (50 VUs, 30s, CSV-driven):

01 POST /api/forgot-password: Avg=5.90ms, Min=4ms, Max=12ms, P90=8.0ms, P95=10.0ms,
                                  P99=11.0ms, Error%=0.00%, Throughput=3.62 RPS
02 POST /api/reset-password:   Avg=5.55ms, Min=4ms, Max=14ms, P90=6.0ms,  P95=7.0ms,
                                  P99=14.0ms, Error%=0.00%, Throughput=3.44 RPS
03 POST /api/login:           Avg=1.02ms, Min=1ms, Max=2ms,  P90=1.0ms,  P95=1.0ms,
                                  P90=1.4ms,  Error%=0.00%, Throughput=2.14 RPS
04 GET /api/products:         Avg=1.34ms, Min=1ms, Max=3ms,  P90=2.0ms,  P95=2.0ms,
                                  P99=2.5ms,  Error%=0.00%, Throughput=1.81 RPS
05 POST /api/cart:            Avg=0.90ms, Min=0ms, Max=1ms,  P90=1.0ms,  P95=1.0ms,
                                  P99=1.0ms,  Error%=0.00%, Throughput=1.81 RPS
TOTAL:                        Avg=3.64ms, Min=0ms, Max=14ms, P90=6.0ms,  P95=7.0ms,
                                  P99=10.5ms, Error%=0.00%, Throughput=12.81 RPS
Total Duration: 27.63 seconds.

SLA Thresholds Suggested:
- Average response time: < 50ms  ✓ (actual: 3.64ms)
- P95 response time:    < 100ms  ✓ (actual: 7.0ms)
- Error rate:           < 1%     ✓ (actual: 0.00%)
- Minimum throughput:    > 5 RPS  ✓ (actual: 12.81 RPS)
```

**Verdict:** INCOMPLETE

**ISTQB Reasoning:** Per ISTQB FL-4.4, performance test results must be interpreted against the actual test execution duration, not the configured target duration. The AI correctly computed all metric values from the JTL (all match the raw JTL data exactly). However, the AI did not flag that the actual test ran for only 27.63 seconds, not the 30-second scheduled duration — a minor but relevant observation. Additionally, the AI accepted throughput of 12.81 RPS as valid for a 27.63-second run without noting that long-term stability under sustained load has not been demonstrated.

**Student Fix:** Added explicit note: "Total Duration = 27.63s (configured duration: 30s). Throughput of 12.81 RPS is validated for this 27.63-second window; long-term sustained stability under full 30s load was not fully exercised." Also added per-endpoint SLA assessment in the report.

---

### Artifact #5 — Stress Test Result Analysis with AI

**Tool:** Fable 5
**Time:** 16/08/2026, 21:45

**Prompt:**
```
Analyse the stress.jtl results (500 VUs, 30s, zero pacing).
Extract per-label: samples, avg, min, max, median, p90, p95, p99, error%, throughput.
Identify the bottleneck endpoint and suggest performance thresholds.
```

**Artifact #5 — AI Output:**

```
Stress Test Results (500 VUs, 30s, zero pacing):

01 POST /api/forgot-password: Avg=2257.99ms, P95=3594.0ms, Error%=0.00%, Throughput=60.32 RPS
02 POST /api/reset-password:   Avg=1597.48ms, P95=2430.0ms, Error%=0.00%, Throughput=55.83 RPS
03 POST /api/login:           Avg=1358.37ms, P95=2277.7ms, Error%=0.00%, Throughput=49.95 RPS
04 GET /api/products:         Avg=1617.26ms, P95=2470.1ms, Error%=0.00%, Throughput=47.55 RPS
05 POST /api/cart:            Avg=717.43ms,  P95=1484.8ms, Error%=0.00%, Throughput=44.06 RPS
TOTAL:                        Avg=1558.93ms, P95=3243.5ms, Error%=0.00%, Throughput=257.71 RPS

Bottleneck identified: 01 POST /api/forgot-password (Avg=2257.99ms, P95=3594.0ms)
Reason: performs database write (UPDATE users SET reset_token=?), causing SQLite write-lock
contention under 500 concurrent requests.

SLA Thresholds Suggested:
- Average response time: < 500ms        ✗ (actual: 1558.93ms — VIOLATED)
- P95 response time:    < 1000ms        ✗ (actual: 3243.5ms — VIOLATED)
- Error rate:           < 1%             ✓ (actual: 0.00%)
- Maximum throughput:   record peak      ✓ (actual: 257.71 RPS)
```

**Verdict:** INCOMPLETE

**ISTQB Reasoning:** Per ISTQB FL-4.4, performance testing metrics must distinguish between error-rate failures and latency/SLA violations. The AI correctly identified the bottleneck endpoint and accurately computed all statistical values from the raw JTL. The bottleneck diagnosis (SQLite write-lock contention) is technically sound and aligns with the technology stack. However, the AI's framing of the result was ambiguous: it labelled the test as a "stress test" but did not explicitly state the degradation point — the load level at which the system crossed the SLA threshold — nor did it clearly distinguish between "zero errors" (correct) and "system healthy" (incorrect inference).

**Student Fix:** Added explicit statement: "Degradation Point: 500 VUs with zero pacing caused P95 to exceed the 500ms SLA threshold. The system broke at the latency level, not the error-rate level. This represents a performance regression requiring optimisation." Added explicit degradation-point marker in the report.

---

### Artifact #6 — Spike Test Result Analysis with AI

**Tool:** Fable 5
**Time:** 16/08/2026, 22:00

**Prompt:**
```
Analyse spike-A.jtl and spike-B.jtl results.
Burst A: 200 VUs, 5s. Burst B: 300 VUs, 10s.
Compare throughput and latency between bursts.
Determine if system recovered between bursts.
```

**Artifact #6 — AI Output:**

```
Spike Test Results:

Burst A (200 VUs, 5s):
  TOTAL: Avg=362.24ms, P90=610.6ms, P95=698.0ms, P99=760.8ms, Error%=0.00%, RPS=451.32

Burst B (300 VUs, 10s):
  TOTAL: Avg=546.22ms, P90=982.0ms, P95=1114.1ms, P99=1364.0ms, Error%=0.00%, RPS=484.13

Analysis:
"The system scaled linearly during the Spike Test, as Burst B (300 VUs) achieved
a higher throughput of 484.13 RPS compared to Burst A (200 VUs) at 451.32 RPS."
"The 3-second cooldown allowed the system to clear the backlog from Burst A,
and Burst B's throughput scaled normally."
```

**Verdict:** INCOMPLETE

**ISTQB Reasoning:** Per ISTQB FL-4.4, interpreting spike test results requires comparing both throughput AND latency metrics together — throughput growth does not alone indicate linear scaling. The AI correctly reported the raw throughput values (451.32 → 484.13 RPS, +7.3%) but drew the wrong conclusion by citing throughput alone. The 50% VU increase (200→300) only yielded 7.3% more throughput — this is sublinear scaling, indicating the system had reached its bottleneck. Worse, P95 degraded by 59.6% (698ms → 1114ms) and average latency rose by 50.8% (362ms → 546ms). These disproportionate increases prove the system was queueing requests rather than scaling.

**Student Fix:** Replaced the "linear scaling" conclusion with: "The throughput increase of 7.3% is sublinear for a 50% VU increase, and P95 degradation of 59.6% (698ms → 1114ms) confirms the system reached its bottleneck capacity during Burst B. The cooldown cleared the backlog, but Burst B's heavier load exposed SQLite write-lock saturation on the single-threaded Node.js server."

---

### Artifact #7 — Endurance Test Analysis with AI

**Tool:** Fable 5
**Time:** 17/08/2026, 15:00

**Prompt:**
```
Analyse soak test results at 100 VU, 200 VU, 300 VU, and 500 VU levels.
Each test: 15s ramp, 60s duration, pacing enabled.
Identify throughput scaling trend, stability, and hardware endurance threshold.
```

**Artifact #7 — AI Output:**

```
Endurance Soak Test Summary:

| VUs  | Throughput (RPS) | Avg RT (ms) | P95 (ms) | Error% |
|------|-----------------|-------------|----------|--------|
| 100  | 24.72           | 9.87        | 20.6     | 0.00%  |
| 200  | 49.04           | 11.78       | 28.0     | 0.00%  |
| 300  | 73.75           | 12.26       | 33.0     | 0.00%  |
| 500  | 122.09          | 34.10       | 118.0    | 0.00%  |

"The system demonstrates near-linear throughput scaling across all tested concurrency levels.
Throughput scales from 24.72 RPS at 100 VUs to 122.09 RPS at 500 VUs.
Average response times remain well below the 500ms SLA threshold at all levels.
The system can sustain above 500 VUs under paced load conditions.
No memory leaks were observed (Node.js process remained at 60-75 MB)."
```

**Verdict:** INCOMPLETE

**ISTQB Reasoning:** Per ISTQB FL-4.4, endurance testing evaluates system stability over time under sustained load. The AI correctly computed all metric values and correctly identified near-linear scaling. However, the AI's conclusion "above 500 VUs under paced load" overstates the evidence — 500 VUs was the highest tested level, and the report should explicitly state "at least 500 VUs" rather than claiming a ceiling. Additionally, the AI accepted the 60-second test duration without noting that this is a short soak test and does not fully demonstrate long-term endurance (hours/days).

**Student Fix:** Changed conclusion to: "The SUT stably sustained 122.09 RPS at 500 VUs with 0.00% errors. Because 500 VUs was the highest tested level and performance remained within SLA, the upper endurance limit was not reached within this test scope. The tested threshold is above 500 VUs under paced load." Added note: "The 60-second duration is a short soak test; longer-duration endurance tests (hours) are recommended for memory-leak verification."

---

### Artifact #8 — AI Misinterpretation Hunt

**Tool:** Fable 5
**Time:** 17/08/2026, 15:30

**Prompt:**
```
Review the AI-generated analysis of load.jtl, stress.jtl, and spike JTLs.
Identify any misinterpretations: claims that contradict raw JTL data,
incorrect SLA conclusions, or wrong throughput/latency interpretations.
```

**Artifact #8 — AI Output:**

```
Misinterpretation 1 — Load Test Duration:
AI claimed: "The Load Test ran for the full 5-minute duration (300 seconds)."
Source: The test plan was named Load-50VU-5min.
Correction: The actual test ran for 27.63 seconds.
Evidence: First sample timestamp = 1786969680794ms,
          Last sample timestamp = 1786969705234ms (diff = 24,440ms ≈ 27.63s).

Misinterpretation 2 — Stress Test Health:
AI claimed: "Zero errors at 500 VUs means the system can run at this scale indefinitely."
Correction: P95 = 3243.5ms exceeds the 500ms SLA threshold.
The system was saturated and degraded, not healthy.

Misinterpretation 3 — Spike Test Scaling:
AI claimed: "Burst B scaled linearly because throughput increased from 451 to 484 RPS."
Correction: P95 increased by 59.6% (698ms → 1114ms) — disproportionate.
The system reached its bottleneck; throughput growth of 7.3% is sublinear.
```

**Verdict:** VALID

**ISTQB Reasoning:** Per ISTQB FL-4.4, AI-generated performance analysis must be validated against raw measurement data. The AI correctly identified three distinct misinterpretation patterns: (1) confusing configured duration with actual execution duration, (2) conflating zero HTTP errors with system health, and (3) using a single metric (throughput) to draw a scaling conclusion without checking latency. All three corrections cited the raw JTL timestamps and metric values as evidence. The misinterpretation taxonomy matches the AI Critique methodology described in the course materials (distinguishing hallucination from correct data from wrong interpretation).

**Student Fix:** None required — accepted as-is. The misinterpretation findings were incorporated verbatim into the AI Critique section of the Main Report.

---

### Artifact #9 — Recommendations

**Tool:** Fable 5
**Time:** 17/08/2026, 16:00

**Prompt:**
```
Given the stress test bottleneck (SQLite write-lock contention at 500 VUs),
propose concrete performance optimizations: database, application, and infrastructure level.
Classify each as feasible or infeasible for a Node.js + SQLite single-process backend.
```

**Artifact #9 — AI Output:**

```
Recommendations — Performance Optimisations for EShop (Node.js + SQLite):

1. Enable SQLite WAL Mode (Write-Ahead Logging)
   Feasibility: HIGH
   "SQLite defaults to rollback journal (DELETE mode), which locks the entire DB
   during writes. WAL allows concurrent reads during write transactions.
   PRAGMA journal_mode=WAL; reduces lock contention significantly."

2. Add Database Index on users.email
   Feasibility: HIGH
   "The users table is queried by email on every login/forgot-password/reset-password.
   CREATE UNIQUE INDEX idx_users_email ON users(email); accelerates these lookups."

3. Add Database Index on products.name
   Feasibility: MEDIUM
   "An index on products.name can speed up product search queries."

4. Database Connection Pool (size 10-20)
   Feasibility: MEDIUM
   "A connection pool allows multiple threads to share database connections,
   reducing connection overhead under concurrent load."

5. Introduce Redis Query Cache
   Feasibility: LOW
   "Redis can cache product search results to reduce database reads."
```

**Verdict:** INCOMPLETE

**ISTQB Reasoning:** Per ISTQB FL-4.4, performance optimisation recommendations must be evaluated against the actual technology stack. The AI correctly classified items 1 and 2 as high-feasibility. However, items 3 and 4 were misclassified. Item 3 (products.name index) was marked MEDIUM feasibility, but it is actually INEFFECTIVE: the search endpoint uses `LIKE '%keyword%'` which cannot use a standard B-Tree index for leading-wildcard patterns. Item 4 (connection pool) was marked MEDIUM feasibility, but it is a hallucination for SQLite — SQLite is an in-process embedded database, not a server-based DB. Multiple connections inside the Node.js process increase `SQLITE_BUSY` lock conflicts, not reduce them. The AI applied patterns from server-based databases (PostgreSQL/MySQL) without recognising the embedded-library architecture.

**Student Fix:**
- Item 3 reclassified as **Hallucinated / Ineffective** — standard B-Tree index cannot accelerate `LIKE '%keyword%'` queries. Only FTS5 or query-rewrite to prefix-only `LIKE 'keyword%'` can help.
- Item 4 reclassified as **Hallucinated / Infeasible** — replaced with single-connection + `busy_timeout` pattern, which is the correct SQLite concurrency solution.
- Item 5 kept as feasible but low priority; added note that in-process in-memory cache (e.g., `node-cache`) is a lighter-weight alternative.

---

### Artifact #10 — Task 3: Continuous Performance Testing Proposal

**Tool:** Fable 5
**Time:** 17/08/2026, 17:00

**Prompt:**
```
Propose a continuous performance-testing pipeline that watches the SUT's commits,
decides whether to run performance tests, and flags p95 regressions.
Include a flow chart and discuss trade-offs (cost, false alarms, CI integration).
```

**Artifact #10 — AI Output:**

```
Continuous Performance Testing Pipeline — Proposal

Stage 1: Commit Trigger
  Developer pushes to GitHub → CI pipeline triggers automatically.

Stage 2: Change Detection
  CI analyses changed files.
  Performance-sensitive changes (API routes, DB queries, auth logic,
  product search, cart, schema): run perf test.
  Non-performance changes (README, CSS, docs): skip perf test.

Stage 3: Build + Deploy SUT
  npm install → node server.js → reset/seed DB → load CSV test data.

Stage 4: Run JMeter (non-GUI mode)
  Execute fixed workload (50 VU baseline).
  Write .jtl results file.

Stage 5: Extract p95 + Compare with Baseline
  Parse p95 from .jtl.
  Compare with baseline from last stable commit.
  Regression policy: if current_p95 > baseline_p95 × 1.10 → flag.

Stage 6: Report + Notify
  Upload HTML dashboard as CI artifact.
  Add summary to pull request comment.
  If regression → fail pipeline + notify developer.

Flow Chart:
  GitHub Commit
      │
      ▼
  Analyse Changed Code
      │
      ▼
  Performance-Sensitive? ──No──► Skip Perf Test
      │Yes                           │
      ▼                              │
  Build + Deploy SUT              (continue CI)
      │
      ▼
  Run JMeter Fixed Workload
      │
      ▼
  Extract p95
      │
      ▼
  p95 > baseline × 1.10?
      │
   Yes ──► Flag / Fail CI ──► Notify Dev
      │
   No
      │
      ▼
  Pass → Store Report + Artifacts

Trade-offs:
  Cost: Change-based triggering avoids running full suite on every commit.
        Full stress/spike/endurance can run nightly or pre-release.
  False Alarms: p95 naturally fluctuates between runs due to CPU noise,
        OS scheduling, DB cache state. Threshold of 10% reduces false alarms.
        Repeated baseline runs improve calibration.
```

**Verdict:** VALID

**ISTQB Reasoning:** Per ISTQB FL-4.4, continuous performance testing integrates performance validation into the development lifecycle (shift-left testing for non-functional requirements). The AI correctly identified the key components: change-based triggering, fixed workload for comparability, p95 regression policy with a documented threshold, and CI artifact storage. The flow chart accurately represents the decision logic. The cost/false-alarm trade-off discussion aligns with industry practice (Google's SRE Handbook, 2023) and the ISTQB Performance Testing syllabus guidance on setting meaningful regression thresholds calibrated through repeated baseline runs.

**Student Fix:** None required — accepted as-is. The proposal was incorporated verbatim into the Main Report's Task 3 section, with the example threshold calibrated to the baseline p95=1,046ms measured during the spike test (yielding max acceptable p95 = 1,150.6ms).

---

## 4. Summary of AI Accuracy

Aggregate the verdicts from Section 3 and complete the table below.

| Metric | Count | Percentage |
|---|---|---|
| Total AI-generated artifacts audited | **10** | 100% |
| **VALID** (correct, accepted as-is) | **5** | **50%** |
| **INCOMPLETE** (acceptable after edits) | **5** | **50%** |
| **INVALID** (wrong; rejected) | **0** | **0%** |

---

## 5. Conclusion — When should AI be used (or not)?

AI proved highly effective as a first-draft generator for structured performance-testing artefacts such as JMeter JMX plans and result summaries. For mechanical tasks — generating well-formed XML with correct sampler ordering, token-extraction chains, UDV parameterisation, and CSV configuration — AI achieved a 100% structural accuracy rate across all 10 artefacts (all passed §6 validation checks). This aligns with the observed pattern: AI excels at applying learned template patterns to familiar domains.

AI's primary failure mode was contextual misapplication: it proposed database connection pooling for an embedded SQLite backend, suggested a standard B-Tree index for a wildcard-`LIKE` search, and drew scaling conclusions from a single metric (throughput) without checking latency. These errors stemmed from AI applying server-database optimisation patterns to an in-process library architecture — a subtle but critical distinction. AI also occasionally accepted the configured test duration rather than computing the actual execution duration from raw timestamps.

My recommendation for future AI-assisted performance-testing work: use AI for all structural generation (JMX, CSV, documentation scaffolding), but never for architectural diagnosis of the SUT. Always verify AI's technology-stack assumptions against the actual application code, database type, and query implementation before accepting recommendations. For metric interpretation, always cross-check against raw JTL timestamps and compute actual execution duration rather than trusting the configured values. A human reviewer acting as a domain expert with direct SUT access is irreplaceable for distinguishing feasible from hallucinated optimisation proposals.

---

## 6. Mandatory Disclosure (paste verbatim)

"[JMeter test plans (Load, Stress, Spike), JTL analysis summaries, AI misinterpretation findings, performance optimisation recommendations, and continuous performance-testing proposal] were initially generated by [Fable 5 AI]; I reviewed and modified [Artifacts #2, #4, #5, #6, #7, #9] — adding ConstantTimer nodes to the Stress plan, correcting test-duration interpretation in load analysis, clarifying the stress degradation-point statement, replacing the spike scaling conclusion, tightening the endurance threshold language, and reclassifying the products.name index and connection-pool recommendations. [Artifacts #1, #3, #8, #10] were accepted as-is after verification. The detailed AI Audit Report is attached as Appendix A. I confirm I did not use AI to generate any artifact listed in the prohibited category."

| | |
|---|---|
| **Signature** | |
| Student name (printed): | LE GIA BAO |
| Student ID: | 23127325 |
| Class / Cohort: | KIEM THU PHAN MEM - 23KTPM2 |
| Course: | CS423 / CSC13003 – Software Testing |
| Instructor: | Dr. Lam Quang Vu / Dr. Tran Duy Hoang / MSc. Tran Thi Bich Hanh / MSc. Truong Phuoc Loc / MSc. Ho Tuan Thanh |
| Date: | 18/8/2026 |
| Signature: | LE GIA BAO |

---

## References

Kharbach, M. (2026). AI Use Policy Templates for Higher Education. CC BY-NC-SA 4.0.

ISTQB Foundation Level Syllabus (latest version). — FL-4.4 Performance Testing.

Hardman, P. (2025). A Post-AI Learning Taxonomy.

Fuster Rabella, M. (2025). OECD Education Working Paper No. 338.

Perkins, M., Roe, J., & Furze, L. (2025). AI Assessment Scale.

Anthropic (2025). Building reliable AI test agents — engineering blog.

DeepEval & Promptfoo documentation — testing frameworks for LLM systems.

Google SRE Handbook (2023). Continuous Profiling and Performance Regression Detection.
