# Soak Test Run: 300 Virtual Users (VUs)

This report details the execution and results of the soak test with **300 concurrent virtual users**.

---

## 1. Execution Summary
*   **Target Load**: 300 VUs
*   **Ramp-up Time**: 15 seconds
*   **Test Duration**: 60 seconds (Actual: 57.80s)
*   **Workload Pacing / Think Time**: 0 ms (fire-hose style, zero pacing)
*   **Target File**: `load.jtl` (Run 5)
*   **Execution Time**: 2026-08-17 16:18:11 to 16:19:09 local time
*   **Total Transactions**: 4,263 samples
*   **Overall Error Rate**: 0.00%
*   **Overall Throughput**: 73.75 RPS
*   **Average Response Time**: 12.26 ms

---

## 2. Performance Metrics Table

| Label | # Samples | Average (ms) | Min (ms) | Max (ms) | Median (ms) | 90% Line (ms) | 95% Line (ms) | 99% Line (ms) | Error % | Throughput (RPS) |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| 01 POST /api/forgot-password | 992 | 18.30 | 6 | 153 | 14.5 | 32.0 | 42.0 | 75.0 | 0.00% | 17.16 |
| 02 POST /api/reset-password | 902 | 17.25 | 6 | 124 | 15.0 | 28.0 | 36.0 | 60.0 | 0.00% | 15.60 |
| 03 POST /api/login | 871 | 8.77 | 1 | 108 | 6.0 | 16.0 | 24.5 | 65.9 | 0.00% | 15.07 |
| 04 GET /api/products?search= | 752 | 10.30 | 1 | 142 | 6.0 | 20.0 | 33.4 | 74.5 | 0.00% | 13.01 |
| 05 POST /api/cart | 746 | 4.25 | 0 | 85 | 3.0 | 6.0 | 10.0 | 44.9 | 0.00% | 12.91 |
| **TOTAL** | **4263** | **12.26** | **0** | **153** | **9.0** | **24.0** | **33.0** | **65.0** | **0.00%** | **73.75** |

---

## 3. Observations & Analysis
*   **Excellent Scaling**: Throughput increased to **73.75 RPS** (a 50.4% increase from 200 VUs). The system scales cleanly from 100 VUs (24.72 RPS) through 200 VUs (49.04 RPS) to 300 VUs (73.75 RPS) with 0.00% errors.
*   **Marginal Latency Growth**: The total average response time only rose slightly to **12.26 ms** (from 11.78 ms at 200 VUs). The P95 response time was still very low at **33.0 ms** (max response time was **153 ms**).
*   **System Threshold**: At 300 VUs, the system shows no signs of resource exhaustion or database lockups under this duration.
