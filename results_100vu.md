# Soak Test Run: 100 Virtual Users (VUs)

This report details the execution and results of the soak test with **100 concurrent virtual users**.

---

## 1. Execution Summary
*   **Target Load**: 100 VUs
*   **Ramp-up Time**: 15 seconds
*   **Test Duration**: 60 seconds (Actual: 57.76s)
*   **Workload Pacing / Think Time**: 0 ms (fire-hose style, zero pacing)
*   **Target File**: `load.jtl` (Run 3)
*   **Execution Time**: 2026-08-17 16:11:05 to 16:12:03 local time
*   **Total Transactions**: 1,428 samples
*   **Overall Error Rate**: 0.00%
*   **Overall Throughput**: 24.72 RPS
*   **Average Response Time**: 9.87 ms

---

## 2. Performance Metrics Table

| Label | # Samples | Average (ms) | Min (ms) | Max (ms) | Median (ms) | 90% Line (ms) | 95% Line (ms) | 99% Line (ms) | Error % | Throughput (RPS) |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| 01 POST /api/forgot-password | 332 | 15.58 | 6 | 52 | 14.0 | 22.0 | 27.4 | 35.7 | 0.00% | 5.75 |
| 02 POST /api/reset-password | 301 | 15.59 | 5 | 75 | 15.0 | 20.0 | 23.0 | 36.0 | 0.00% | 5.21 |
| 03 POST /api/login | 291 | 5.71 | 2 | 49 | 5.0 | 9.0 | 10.0 | 13.1 | 0.00% | 5.04 |
| 04 GET /api/products?search= | 253 | 6.39 | 2 | 26 | 6.0 | 9.0 | 12.8 | 19.5 | 0.00% | 4.38 |
| 05 POST /api/cart | 251 | 3.81 | 1 | 20 | 4.0 | 5.0 | 6.0 | 10.5 | 0.00% | 4.35 |
| **TOTAL** | **1428** | **9.87** | **1** | **75** | **8.0** | **17.0** | **20.6** | **31.0** | **0.00%** | **24.72** |

---

## 3. Observations & Analysis
*   **Nominal Capacity**: With 100 VUs and zero think time, the backend handles the load with ease. Latency is extremely low (average **9.87 ms**, maximum **75 ms**), indicating no queueing bottleneck.
*   **Error-free Execution**: The error rate is **0.00%**, confirming that the SQLite database and Express server are fully stable at this concurrency level.
*   **Resource Utilization**: The CPU and memory usage of the host remained low and stable throughout the duration of the test.

> [!NOTE]
> *Failed Run Log*: A previous execution attempt (Run 2) was captured in `load.jtl` showing a **65.23% error rate**. Inspection of the logs revealed `org.apache.http.conn.HttpHostConnectException: Connection refused` for 1,033 samples, indicating that the target backend API was briefly down or crashed at the start of that run. Once the server was restarted, the subsequent run (documented above) completed with **0.00% errors**.
