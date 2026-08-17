# Soak Test Run: 200 Virtual Users (VUs)

This report details the execution and results of the soak test with **200 concurrent virtual users**.

---

## 1. Execution Summary
*   **Target Load**: 200 VUs
*   **Ramp-up Time**: 15 seconds
*   **Test Duration**: 60 seconds (Actual: 57.80s)
*   **Workload Pacing / Think Time**: 0 ms (fire-hose style, zero pacing)
*   **Target File**: `load.jtl` (Run 4)
*   **Execution Time**: 2026-08-17 16:15:26 to 16:16:24 local time
*   **Total Transactions**: 2,835 samples
*   **Overall Error Rate**: 0.00%
*   **Overall Throughput**: 49.04 RPS
*   **Average Response Time**: 11.78 ms

---

## 2. Performance Metrics Table

| Label | # Samples | Average (ms) | Min (ms) | Max (ms) | Median (ms) | 90% Line (ms) | 95% Line (ms) | 99% Line (ms) | Error % | Throughput (RPS) |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| 01 POST /api/forgot-password | 666 | 18.46 | 6 | 99 | 15.0 | 29.0 | 35.0 | 50.1 | 0.00% | 11.52 |
| 02 POST /api/reset-password | 602 | 17.31 | 6 | 61 | 15.5 | 25.0 | 29.0 | 42.0 | 0.00% | 10.41 |
| 03 POST /api/login | 573 | 7.22 | 1 | 41 | 6.0 | 11.8 | 15.0 | 28.3 | 0.00% | 9.91 |
| 04 GET /api/products?search= | 502 | 9.13 | 2 | 50 | 7.0 | 17.0 | 23.9 | 36.0 | 0.00% | 8.68 |
| 05 POST /api/cart | 492 | 3.98 | 1 | 16 | 3.0 | 6.0 | 8.0 | 13.0 | 0.00% | 8.51 |
| **TOTAL** | **2835** | **11.78** | **1** | **99** | **10.0** | **22.0** | **28.0** | **39.7** | **0.00%** | **49.04** |

---

## 3. Observations & Analysis
*   **Linear Scaling of Throughput**: Throughput scaled almost perfectly linearly from the 100 VU run (from **24.72 RPS** to **49.04 RPS**, a 98.4% increase).
*   **Low Latency Profile**: Average latency only increased marginally from **9.87 ms** to **11.78 ms**, while the 95th percentile remained extremely low at **28.0 ms** (max response time was **99 ms**).
*   **Error-free Execution**: The error rate remained **0.00%**, showing the system is highly stable at 200 VUs under this workload.
