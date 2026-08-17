# Soak Test Run: 500 Virtual Users (VUs)

This report details the execution and results of the soak test with **500 concurrent virtual users**.

---

## 1. Execution Summary
*   **Target Load**: 500 VUs
*   **Ramp-up Time**: 15 seconds
*   **Test Duration**: 60 seconds (Actual: 57.98s)
*   **Workload Pacing / Think Time**: Throttled by User-Defined timers (cumulative 15s think time per loop iteration)
*   **Target File**: `load.jtl` (Run 5, filtered for samples after cutoff)
*   **Execution Time**: 2026-08-17 16:19:10 to 16:20:29 local time
*   **Total Transactions**: 7,079 samples
*   **Overall Error Rate**: 0.00%
*   **Overall Throughput**: 122.09 RPS
*   **Average Response Time**: 34.10 ms

---

## 2. Performance Metrics Table

| Label | # Samples | Average (ms) | Min (ms) | Max (ms) | Median (ms) | 90% Line (ms) | 95% Line (ms) | 99% Line (ms) | Error % | Throughput (RPS) |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| 01 POST /api/forgot-password | 1655 | 50.38 | 5 | 410 | 30.0 | 117.0 | 165.3 | 256.8 | 0.00% | 28.54 |
| 02 POST /api/reset-password | 1505 | 43.10 | 6 | 327 | 27.0 | 91.0 | 131.0 | 217.0 | 0.00% | 25.96 |
| 03 POST /api/login | 1434 | 25.50 | 1 | 236 | 12.0 | 63.0 | 88.3 | 154.0 | 0.00% | 24.73 |
| 04 GET /api/products?search= | 1252 | 34.28 | 1 | 306 | 18.0 | 88.0 | 117.0 | 195.0 | 0.00% | 21.59 |
| 05 POST /api/cart | 1233 | 11.11 | 1 | 144 | 5.0 | 27.0 | 42.0 | 85.7 | 0.00% | 21.26 |
| **TOTAL** | **7079** | **34.10** | **1** | **410** | **19.0** | **82.2** | **118.0** | **217.0** | **0.00%** | **122.09** |

---

## 3. Observations & Analysis
*   **Scale and Throughput**: Throughput scaled to **122.09 RPS** with **7,079 total transactions** in under a minute.
*   **Controlled Latency Growth**: Average latency increased to **34.10 ms** (from 12.26 ms at 300 VUs). The 95th percentile remained well within standard SLA limits at **118.0 ms** (with a max transaction duration of **410 ms**).
*   **Timer Buffering Effect**: Even at 500 VUs, response times remain highly stable because the Load test plan enforces a 15-second pacing delay between user actions. This spaces out the API traffic, preventing the database and Node.js event loop from being flooded at the same instant (contrast this with the 500 VU Stress Test with 0ms timers, where average latency degraded to **1558.93 ms** at a peak of **257.71 RPS**).
