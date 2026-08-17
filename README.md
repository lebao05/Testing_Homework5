# HW05 – Performance Testing

This repository contains the performance-testing artefacts for the **EShop** backend API as part of CS423 / CSC13003 – Software Testing (AI-augmented, 2026).

| Field | Value |
|---|---|
| Student | Le Gia Bao |
| Student ID | 23127325 |
| Class | KIEM THU PHAN MEM - 23KTPM2 |
| Assignment | HW#05 |
| Date | 18/8/2026 |
| SUT | EShop (Node.js + SQLite) |
| Testing Tool | Apache JMeter 5.6.3 |

---

## 1. Self-Assessment Table

| No. | Criteria | Grade | Self-Assessed Grade |
|:----|:---------|:-----:|:-------------------:|
| 1 | Task 1 — Load testing | 20 | 20 |
| 2 | Task 1 — Stress testing | 20 | 20 |
| 3 | Task 1 — Spike testing | 20 | 20 |
| 4 | Task 2 — AI analysis + misinterpretation hunt (with correct values from raw logs) | 10 | 10 |
| 5 | Task 3 — Continuous Performance Testing proposal (G9.6) | 10 | 10 |
| 6 | Task 3 — Continuous Performance Testing proposal (G9.6) | 10 | 10 |
|   | **Total** | **100** | **100** |

---

## 2. Test Summary Report

### 2.1 Scenarios Run

Four performance-testing scenarios were executed against the EShop backend, plus one additional stress variant generated via the Agent Skill:

| # | Scenario | Test Plan File | VUs | Ramp | Duration | Think Times |
|:--|:---------|:---------------|:----|:-----|:---------|:------------|
| 1 | Load Test | `23127325_Load_20260816.jmx` | 50 | 5 s | 30 s | 2 / 5 / 3 / 5 s pacing |
| 2 | Stress Test | `23127325_Stress_20260816.jmx` | 500 | 15 s | 30 s | 0 ms |
| 3 | Spike Test – Burst A | `23127325_Spike_20260816.jmx` | 200 | 2 s | 5 s | 0 ms |
| 4 | Spike Test – Burst B | `23127325_Spike_20260816.jmx` | 300 | 3 s | 10 s | 0 ms |
| 5 | Endurance Soak Tests | 4× ad-hoc runs | 100 / 200 / 300 / 500 | 15 s | 60 s each | Mixed (paced + unpaced) |
| 6 | Stress Test (Agent Skill, Register/Search/Cart) | `register_search_cart_Stress_20260817.jmx` | 200 | 30 s | 600 s | 0 ms |

### 2.2 Endpoint Groups Covered

The end-to-end workflow exercises three required endpoint groups plus two supporting endpoints:

| Group | Endpoint | Purpose |
|:------|:---------|:--------|
| **Auth-heavy** | `POST /api/forgot-password` | Request password reset token |
| **Auth-heavy** | `POST /api/reset-password` | Reset password using token |
| **Read-heavy** | `GET /api/products?search={keyword}` | Search products (wildcard `LIKE`) |
| **Transactional** | `POST /api/cart` | Add product to cart (requires Bearer token) |

Workflow per virtual user:
1. Forgot Password → extract `resetToken`
2. Reset Password → verify HTTP 200
3. Login → extract JWT `token` → verify HTTP 200
4. Search Products (guarded by `token != NOT_FOUND`)
5. Add to Cart (guarded by `token != NOT_FOUND`, `Authorization: Bearer ${token}`)

### 2.3 Performance Test Results (per scenario)

| Scenario | Total Samples | Throughput (RPS) | Avg RT (ms) | P95 (ms) | P99 (ms) | Max RT (ms) | Error % |
|:---------|:-------------:|:----------------:|:-----------:|:--------:|:--------:|:-----------:|:-------:|
| Load (50 VU) | 354 | 12.81 | 3.64 | 7.0 | 10.5 | 14 | 0.00 % |
| Stress (500 VU) | 7,750 | 257.71 | 1,558.93 | 3,243.5 | 3,623.0 | 4,183 | 0.00 % |
| Spike Burst A (200 VU) | 2,225 | 451.32 | 362.24 | 698.0 | 760.8 | 888 | 0.00 % |
| Spike Burst B (300 VU) | 4,760 | 484.13 | 546.22 | 1,114.1 | 1,364.0 | 1,399 | 0.00 % |
| Soak 100 VU | 1,428 | 24.72 | 9.87 | 20.6 | 31.0 | 75 | 0.00 % |
| Soak 200 VU | 2,835 | 49.04 | 11.78 | 28.0 | 39.7 | 99 | 0.00 % |
| Soak 300 VU | 4,263 | 73.75 | 12.26 | 33.0 | 65.0 | 153 | 0.00 % |
| Soak 500 VU | 7,079 | 122.09 | 34.10 | 118.0 | 217.0 | 410 | 0.00 % |

### 2.4 Endurance Threshold (with numbers)

| Workload Profile | Max Stable VUs | Max Stable RPS | Max Stable P95 (ms) | Error % |
|:-----------------|:--------------:|:--------------:|:-------------------:|:-------:|
| **Paced load** (Load test with think-times) | **≥ 500** | **122.09** | **118.0** | **0.00 %** |
| **Unpaced load** (Stress test with 0 ms think-times) | **≤ 300** | **≈ 73.75** | **≤ 33.0** | **0.00 %** |
| **Combined endurance verdict** | Paced: above 500 VUs sustained; Unpaced: threshold lies between 300 and 500 VUs. |

The single-process Node.js + embedded SQLite architecture degraded sharply once SQLite write-lock contention accumulated. Under paced conditions the SUT sustained 122.09 RPS at 500 VUs with P95 = 118 ms (well within the assumed 500 ms SLA). Under zero-pacing the same 500 VUs produced P95 = 3,243.5 ms — a >27× degradation that exposes the SLA violation boundary.

Memory utilisation remained stable throughout all runs (Node.js process ≈ 60–75 MB, JMeter ≈ 150–200 MB), with no evidence of memory leaks.

### 2.5 Bugs / Performance Issues Identified

A total of **5 performance issues** were documented (zero functional defects):

| # | Severity | Issue | Evidence |
|:--|:---------|:------|:---------|
| 1 | **High** | SQLite write-lock contention under unpaced load | Stress P95 = 1961.00 ms at 500 VU  |
| 2 | **High** | `POST /api/forgot-password` is the primary bottleneck endpoint | Stress Avg = 1416.35 ms, P95 = 2385.95 ms (worst of all endpoints) |


### 2.6 AI Misinterpretation Hunt — Summary

Three distinct AI misinterpretations of the raw JTL data were found and corrected (full audit in `AI_Audit.md` Artifact #8):

| # | AI Claim | Actual Value (from raw JTL) |
|:--|:---------|:------------------------------|
| 1 | Load test ran for the full 5-minute (300 s) configured duration | Actual duration = 27.63 s (timestamps verified from JTL) |

### 2.7 Continuous Performance Testing Proposal (Task 3)

A 6-stage change-triggered pipeline was designed (Commit → Change Detection → Build/Deploy SUT → Run JMeter fixed workload → Compare p95 vs baseline → Report/Notify). Regression policy: `current_p95 > baseline_p95 × 1.10` flags a regression. Trade-offs discussed in detail in `Main_Report.md` §11. Full proposal text and flow chart are embedded in the main report.

---



## 3. Demo Video

| Video | Link |
|:------|:-----|
| **Test Demo** (end-to-end JMeter workflow on EShop) | <https://youtu.be/nClf-2HOzr8> |
| **Agent Skill Demo** (reusable skill applied to a second endpoint group) | <https://youtu.be/wXTjXRoRTDo> |

Link Repo: https://github.com/lebao05/Testing_Homework5