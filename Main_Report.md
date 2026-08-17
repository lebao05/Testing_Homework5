**HW05 – Performance Testing**

Student: Le Gia Bao  
Student ID: 23127325  
Course: Software Testing – AI-First Edition  
Exercise: HW05-AI  
SUT: EShop  
Testing Tool: Apache JMeter 5.6.3  
Date: 2026-08-18   
**1\. Executive Summary**  
This assignment evaluates the performance of the EShop backend API using three performance-testing techniques: Load Testing, Stress Testing, and Spike Testing.

The three scenarios use the same end-to-end workflow:  
Forgot Password  
     ↓  
Reset Password  
     ↓  
Login  
     ↓  
Search Products  
     ↓  
Add Product to Cart

The workflow covers the three required endpoint groups:

The tests were designed with AI assistance and subsequently reviewed and corrected manually. CSV-driven test data was used to provide different user credentials, search keywords, and passwords.

The main objectives were:

1. Evaluate the SUT under normal sustained load.  
2. Identify the point where the system becomes unstable under increasing load.  
3. Observe the system's behaviour under sudden traffic spikes.  
4. Determine an empirical endurance threshold on the test hardware.  
5. Use AI to analyse the raw JMeter results and identify incorrect AI interpretations.  
6. Propose a continuous performance-testing pipeline

**2\. System Under Test**

**2.1 SUT Description**

The System Under Test is the **EShop Vietnamese e-commerce demo application**.

The backend exposes REST APIs consumed by the web application.

**2.2 Selected Endpoint Groups**

Three endpoint groups were selected according to the HW05 requirements.

**Auth-heavy**

POST /api/forgot-password  
POST /api/reset-password

These endpoints involve authentication-related processing and therefore represent the authentication workload.

Read-heavy  
GET /api/products?search={keyword}

This endpoint represents product searching and database-heavy read operations.

Transactional  
POST /api/cart

This endpoint modifies application state by adding a product to a user's cart.

3\. Hardware Specification

![][image1]

**4\. End-to-End Workflow**

Each virtual user executes the following workflow:

Workflow Steps

1. **Forgot Password**  
   * POST /api/forgot-password  
   * Input: email  
   * Extract resetToken from response.  
2. **Reset Password**  
   * POST /api/reset-password  
   * Input: email, resetToken, newPassword  
   * Verify HTTP 200\.  
3. **Login**  
   * POST /api/login  
   * Input: email, newPassword  
   * Extract JWT token.  
   * Verify HTTP 200\.  
4. **Search Products**  
   * GET /api/products?search={search\_keyword}  
   * Input: search\_keyword from CSV.  
   * Extract product\_id from the response.  
5. **Add Product to Cart**  
   * POST /api/cart  
   * Input: product\_id, product information, quantity.  
   * Authorization: Bearer {token}.

**Note:** Although login isn’t selected, it is added to ensure the completeness of the workflow and to get access\_token for using Add-To-Cart API.

**5\. Load Test**

**5.1 Objective**  
The Load Test evaluates the behaviour of the EShop API under an expected sustained workload. The objective is to determine whether the system can maintain acceptable:

* **Throughput:** Verify the transaction rate of the system under normal user volumes.  
* **Response Time:** Ensure average and percentile latencies are well within SLA limits.  
* **Error Rate:** Confirm that the API maintains a 0% error rate under expected load.  
* **CPU Utilization:** Verify that the single-threaded Node.js server does not saturate the host's CPU.  
* **Memory Utilization:** Confirm that the system's memory remains stable without signs of leaks.

**5.2 Configuration**  
The test plan is designed to simulate a normal user flow: forgot password, reset password, login, search for a product, and add to the cart. Pacing and think times are implemented to simulate realistic human behavior.

| Parameter | Configuration |
| :---- | :---- |
| Test Plan File | 23127325\_Load\_20260816.jmx |
| Virtual Users (VUs) | 50 concurrent threads. |
| Ramp-up Period | 5 seconds. |
| Test Duration | 30 seconds. |
| Think Times (Pacing) | thinkStep1: 2000 ms; thinkStep2: 5000 ms; thinkStep3: 3000 ms; thinkStep4: 5000 ms. |
| Pacing Jitter | Uniform Random Timer with a delay range of 0 to 1000 ms. |
| Data-Driven Configuration | load\_users.csv containing 50 unique user credentials (email, new\_password) and search keywords. CSV Share Mode is configured to currentThread with recycling enabled. |
| Listener | Summary Report logging to test-plans/load/results/load.jtl. |

Workflow steps:

1. POST /api/forgot-password (Auth-heavy) → extracts resetToken.  
2. POST /api/reset-password (Auth-heavy) → resets the password using the token.  
3. POST /api/login (Auth-heavy) → logs in with the new password and extracts JWT token.  
4. GET /api/products?search= (Read-heavy) → searches products using keyword (guarded by token \!= 'NOT\_FOUND').  
5. POST /api/cart (Transactional) → adds product to cart (guarded by token \!= 'NOT\_FOUND').

**5.3 Result analysis with AI**

| Label | \# Samples | Average (ms) | Min | Max | Median | 90% Line | 95% Line | 99% Line | Error % | Throughput (RPS) |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- |
| 01 POST /api/forgot-password | 100 | 5.90 | 4 | 12 | 6.0 | 8.0 | 10.0 | 11.0 | 0.00% | 3.62 |
| 02 POST /api/reset-password | 95 | 5.55 | 4 | 14 | 5.0 | 6.0 | 7.0 | 14.0 | 0.00% | 3.44 |
| 03 POST /api/login | 59 | 1.02 | 1 | 2 | 1.0 | 1.0 | 1.0 | 1.4 | 0.00% | 2.14 |
| 04 GET /api/products?search= | 50 | 1.34 | 1 | 3 | 1.0 | 2.0 | 2.0 | 2.5 | 0.00% | 1.81 |
| 05 POST /api/cart | 50 | 0.90 | 0 | 1 | 1.0 | 1.0 | 1.0 | 1.0 | 0.00% | 1.81 |
| TOTAL | 354 | 3.64 | 0 | 14 | 4.0 | 6.0 | 7.0 | 10.5 | 0.00% | 12.81 |

**Total Duration:** 27.63 seconds

**Load Analysis:** Under the normal workload (50 concurrent users with pacing), the SUT responded extremely fast. The average response time was only 3.64 ms, and the maximum response time across all samples was 14 ms. The error rate was 0.00%, and the system handled a throughput of 12.81 RPS comfortably without any degradation of resources.

**6\. Stress Test**

**6.1 Objective**  
The Stress Test progressively increases the workload beyond the expected operating level to identify the point at which the system begins to degrade. The objective is to:

* Identify the maximum throughput capacity of the backend API.  
* Locate the system bottlenecks (e.g., SQLite database lock contention, Node.js event loop saturation).  
* Evaluate how the system handles sustained high concurrency with zero think-time.  
* Verify stability and check if error rates or resource leaks occur under stress.

**6.2 Stress Configuration**

| Parameter | Configuration |
| :---- | :---- |
| Test Plan File | 23127325\_Stress\_20260816.jmx |
| Virtual Users (VUs) | 500 concurrent threads (10x normal load). |
| Ramp-up Period | 15 seconds. |
| Test Duration | 30 seconds. |
| Think Times (Pacing) | 0 ms (no think times to apply maximum pressure). |
| Data-Driven Configuration | stress\_users.csv containing 500 unique user accounts (perf-user-001..500@load.com) and search keywords. |
| Workflow steps | Same 5-step flow as Load, executed back-to-back. |
| Listener | StatVisualizer (Aggregate Report) logging to test-plans/stress/results/stress.jtl \+ View Results Tree (live only). |

**6.3 Result analysis with AI**

| Label | \# Samples | Average (ms) | Min | Max | Median | 90% Line | 95% Line | 99% Line | Error % | Throughput (RPS) |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- |
| 01 POST /api/forgot-password | 1814 | 2257.99 | 15 | 4183 | 2491.5 | 3517.7 | 3594.0 | 3994.9 | 0.00% | 60.32 |
| 02 POST /api/reset-password | 1679 | 1597.48 | 8 | 2521 | 1816.0 | 2324.0 | 2430.0 | 2476.2 | 0.00% | 55.83 |
| 03 POST /api/login | 1502 | 1358.37 | 4 | 2852 | 1369.0 | 2216.9 | 2277.7 | 2486.0 | 0.00% | 49.95 |
| 04 GET /api/products?search= | 1430 | 1617.26 | 4 | 2844 | 1803.0 | 2457.0 | 2470.1 | 2797.7 | 0.00% | 47.55 |
| 05 POST /api/cart | 1325 | 717.43 | 4 | 1613 | 591.0 | 1406.0 | 1484.8 | 1582.8 | 0.00% | 44.06 |
| TOTAL | 7750 | 1558.93 | 4 | 4183 | 1555.0 | 2725.0 | 3243.5 | 3623.0 | 0.00% | 257.71 |

**Total Duration:** 30.07 seconds

**Stress Analysis:** Under the heavy stress of 500 VUs with zero pacing, the system maintained a 0.00% error rate but suffered from significant latency degradation. The total average response time surged to 1558.93 ms (a massive jump from 3.64 ms under normal load), and the 95th percentile reached 3243.50 ms. The most degraded endpoint was 01 POST /api/forgot-password with an average latency of 2257.99 ms and a max latency of 4183 ms. This endpoint performs database writes (UPDATE users SET reset\_token \= ?) which triggers SQLite lock contention, blocking subsequent requests.

**6.4 Stress Threshold**  
**Maximum Throughput:** The system's peak throughput reached 257.71 RPS.  
**Breaking/Degradation Point:** The acceptable threshold for response times (e.g., 500 ms) was violated, as the total median response time climbed to 1555.0 ms and the 90% Line reached 2725.0 ms.

**7\. Spike Test** 

**7.1 Objective**  
The Spike Test evaluates the system's response to sudden increases in traffic. The test consists of two high-volume bursts separated by a cooldown period. The objectives are to:

* Measure the elasticity of the SUT when subjected to sudden, intense bursts.  
* Determine if response times recover to normal levels during the cooldown.  
* Observe whether Burst A causes lingering issues (like connection pool starvation or DB locks) that degrade performance during Burst B.

**7.2 Test Plan Configuration**

| Parameter | Configuration |
| :---- | :---- |
| Test Plan File | 23127325\_Spike\_20260816.jmx |
| Virtual Users (VUs) | Burst A: 200 concurrent threads; Burst B: 300 concurrent threads. |
| Ramp-up Period | Burst A: 2 seconds; Burst B: 3 seconds. |
| Duration | Burst A: 5 seconds; Burst B: 10 seconds. |
| Cooldown Period | 3 seconds. Implemented via a startup delay on Spike-B set to 2s (ramp\_A) \+ 5s (dur\_A) \+ 3s (cooldown) \= 10 seconds. |
| Think Times (Pacing) | 0 ms (no think times, fire-hose style). |
| Data-Driven Configuration | strike\_users.csv containing 500 unique user accounts. |
| Listeners | Independent StatVisualizers writing to results/spike-A.jtl and results/spike-B.jtl. |

**7.3 Result analysis with AI**

Burst A (200 VUs, 5s duration)

| Label | \# Samples | Average (ms) | Min | Max | Median | 90% Line | 95% Line | 99% Line | Error % | Throughput (RPS) |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- |
| 01 POST /api/forgot-password | 548 | 511.50 | 10 | 888 | 548.0 | 725.3 | 749.9 | 830.7 | 0.00% | 111.16 |
| 02 POST /api/reset-password | 496 | 374.94 | 5 | 606 | 433.0 | 530.5 | 583.0 | 585.1 | 0.00% | 100.61 |
| 03 POST /api/login | 438 | 337.57 | 4 | 647 | 333.5 | 492.3 | 526.0 | 565.0 | 0.00% | 88.84 |
| 04 GET /api/products?search= | 395 | 355.64 | 5 | 573 | 349.0 | 522.4 | 558.3 | 571.0 | 0.00% | 80.12 |
| 05 POST /api/cart | 348 | 147.64 | 1 | 321 | 138.0 | 228.0 | 308.0 | 320.0 | 0.00% | 70.59 |
| TOTAL | 2225 | 362.24 | 1 | 888 | 369.0 | 610.6 | 698.0 | 760.8 | 0.00% | 451.32 |

**Total Duration:** 4.93 seconds

Burst B (300 VUs, 10s duration)

| Label | \# Samples | Average (ms) | Min | Max | Median | 90% Line | 95% Line | 99% Line | Error % | Throughput (RPS) |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- |
| 01 POST /api/forgot-password | 1102 | 835.32 | 5 | 1399 | 900.0 | 1147.9 | 1360.0 | 1369.0 | 0.00% | 112.08 |
| 02 POST /api/reset-password | 998 | 525.53 | 4 | 916 | 536.0 | 761.0 | 902.0 | 909.0 | 0.00% | 101.51 |
| 03 POST /api/login | 935 | 493.35 | 1 | 1087 | 514.0 | 658.0 | 906.3 | 936.0 | 0.00% | 95.10 |
| 04 GET /api/products?search= | 882 | 601.97 | 1 | 1088 | 622.0 | 866.9 | 983.0 | 1076.0 | 0.00% | 89.71 |
| 05 POST /api/cart | 843 | 193.08 | 1 | 430 | 193.0 | 287.0 | 370.9 | 398.6 | 0.00% | 85.74 |
| TOTAL | 4760 | 546.22 | 1 | 1399 | 535.0 | 982.0 | 1114.1 | 1364.0 | 0.00% | 484.13 |

**Total Duration:** 9.83 seconds

Spike Analysis  
The SUT proved to be resilient under sudden spikes, sustaining 0.00% errors in both bursts. Under Burst A (200 VUs), the average response time was 362.24 ms and the total throughput reached 451.32 RPS. Under Burst B (300 VUs), throughput scaled to 484.13 RPS (+7.3%). However, the increased traffic pushed the API closer to saturation: average response time climbed to 546.22 ms (+50.8% compared to Burst A) and the 95th percentile rose to 1114.10 ms. This indicates that the 3-second cooldown allowed the system to clear the backlog from Burst A, but Burst B's increased load generated higher queuing latencies on the single-threaded Node.js server.

**8\. Endurance test.**

To empirically determine the SUT's hardware threshold and endurance limits, a series of short soak tests were conducted with a 15-second ramp-up and 60-second test duration. Four increasing concurrency levels were evaluated: 100 VUs, 200 VUs, 300 VUs, and 500 VUs.

Endurance Comparison Table

| Metric | 100 VU Soak Test | 200 VU Soak Test | 300 VU Soak Test | 500 VU Soak Test |
| :---- | :---- | :---- | :---- | :---- |
| Virtual Users (VUs) | 100 | 200 | 300 | 500 |
| Ramp-up Time (s) | 15.00 | 15.00 | 15.00 | 15.00 |
| Throughput (RPS) | 24.72 | 49.04 | 73.75 | 122.09 |
| Average Response Time (ms) | 9.87 | 11.78 | 12.26 | 34.10 |
| Median Response Time (ms) | 8.0 | 10.0 | 9.0 | 19.0 |
| 95% Percentile (ms) | 20.6 | 28.0 | 33.0 | 118.0 |
| 99% Percentile (ms) | 31.0 | 39.7 | 65.0 | 217.0 |
| Maximum Response Time (ms) | 75 | 99 | 153 | 410 |
| Error Rate (%) | 0.00% | 0.00% | 0.00% | 0.00% |

**8.1. Excellent Throughput Scaling**  
The system demonstrates near-linear throughput scaling as concurrency increases under paced workload conditions:

* 100 VUs → 24.72 RPS  
* 200 VUs → 49.04 RPS (approximately 2.0× the 100-VU throughput)  
* 300 VUs → 73.75 RPS (approximately 3.0× the 100-VU throughput)  
* 500 VUs → 122.09 RPS (approximately 4.9× the 100-VU throughput)

This indicates that the SUT can utilize additional concurrency effectively when requests are distributed over time by pacing. At 500 VUs, the system achieved 122.09 RPS with a 0.00% error rate.

**8.2. Stable Latency Under Paced Workloads**  
At the highest tested concurrency of 500 VUs:

* Average response time: 34.10 ms  
* Median response time: 19.0 ms  
* P95: 118.0 ms  
* P99: 217.0 ms  
* Maximum: 410 ms  
* Error rate: 0.00%

The maximum observed latency remained below the assumed 500 ms SLA.

This contrasts with the separate 500-VU Stress Test, where average response time reached 1558.93 ms and overall P95 reached 3243.50 ms. The primary difference is workload intensity: the soak tests use pacing, while the stress test applies requests with zero think time. Pacing distributes database operations over time and reduces accumulated SQLite write-lock contention.

**8.3. Hardware & Endurance Threshold Verdict**

**Maximum Stable RPS under Paced Load**

The SUT successfully sustained 122.09 RPS at 500 VUs with 0.00% errors. No significant latency degradation or error-rate increase was observed within the tested 60-second soak period.

**Memory Utilization**

The Node.js server process remained approximately within 60–75 MB, substantially below the approximate V8 default heap limit of \~1.4 GB. JMeter memory usage also remained stable at approximately 150–200 MB. No evidence of memory leakage was observed during these short soak tests.

**Endurance Threshold**

For normal paced operations, the tested threshold is 500 VUs / 122.09 RPS or higher. Because 500 VUs was the highest tested level and still remained within the latency and error targets, the actual upper limit was not reached. Therefore, the hardware threshold should be described as above 500 VUs under the tested paced workload, rather than as an exact maximum.

For unpaced operations, the observed threshold is substantially lower. The separate Stress Test demonstrated severe degradation at 500 VUs, with SQLite write contention causing response times to exceed the 500 ms SLA.

**9\. AI Misinterpretation Hunt**

An AI tool was prompted to analyze the raw JTL results. After checking the raw .jtl data, the interpretation was found to be incorrect. The detailed corrections are listed below.

**Misinterpretation 1 — Load Test Duration**

* AI claimed:  
   *The Load Test ran successfully for its full scheduled 5-minute duration (300 seconds) at a sustained throughput of 12.81 RPS.*  
* Actual value:  
   Total Duration \= 27.63 seconds (with only 354 samples recorded).  
* Source:  
   load.jtl  
* Correction:  
   The AI incorrectly read the target test plan filename and Thread Group title (Load-50VU-5min) instead of checking the actual timestamps of the first and last samples in the raw log. The test actually executed for only 27.63 seconds. Therefore, the throughput of 12.81 RPS is validated only for this brief period, and long-term stability has not been demonstrated.

**10\. Evaluation of AI Optimization Recommendations**

This assessment evaluates the validity of proposed performance optimizations against the actual technology stack of the System Under Test (SUT), which consists of a single-process Node.js backend using SQLite as an in-process database engine.

| Proposed Optimization | Classification | Rationale & Feasibility Analysis |
| :---- | :---- | :---- |
| 1\. Enable SQLite WAL Mode | Feasible | SQLite normally uses a rollback journal, which can cause readers and writers to block each other. Our EShop workflow has several database writes, so concurrent users may experience higher latency.Enabling WAL (Write-Ahead Logging) allows reads to continue while a write is happening. This can reduce database locking and improve performance under concurrent load.  |
| 2\. Add Index on users.email | Feasible | High Feasibility. The users table is frequently searched by email during login, forgot-password, and reset-password. Without an index, SQLite may need to scan the entire table, which becomes slower as the number of users grows. Adding a unique index on email can make these lookups much faster and reduce database work under concurrent requests. This is a small and low-risk change that is easy to implement.  |
| 3\. Add Index on products.name | Hallucinated / Ineffective | Infeasible for the current implementation. The product search endpoint executes a query similar to SELECT \* FROM products WHERE name LIKE '%keyword%'. Standard SQLite B-Tree indexes cannot efficiently accelerate searches beginning with a leading wildcard (%).  |
| 4\. Database Connection Pool | Hallucinated | Infeasible for the current architecture. Connection pooling is commonly used with server-based databases such as PostgreSQL and MySQL. SQLite operates as an embedded library inside the Node.js process rather than as a separate database server. Multiple SQLite connections can increase write contention and SQLITE\_BUSY errors because SQLite permits only one writer at a time. A single shared connection with appropriate busy\_timeout configuration is more appropriate for this SUT. |
| 5\. Introduce Redis Cache | Feasible but Over-engineered | Low Feasibility for the current SUT. Redis can technically cache product search results, but it introduces additional infrastructure, deployment, monitoring, and synchronization complexity for a small single-node SQLite application. A lightweight in-memory cache such as node-cache, an LRU cache, or a JavaScript Map could provide similar benefits without an external service dependency. |

**11\. Task 3 — Continuous Performance Testing Proposal**

**11.1 Proposed Model**  
The continuous performance-testing pipeline consists of six stages:

**1\. Developer pushes a commit**

* Developer pushes code to the GitHub repository.  
* The CI pipeline is triggered.

**2\. Detect the type of change**

* Analyze changed files.  
* If the change affects performance-sensitive components such as API controllers/routes, database queries, authentication, product search, cart/checkout, or database schema, trigger a performance test.  
* Otherwise, skip the expensive performance test.

**3\. Build and deploy the SUT**

* Install dependencies.  
* Start the backend.  
* Prepare/reset the test database.  
* Load the required CSV test data.

**4\. Run a short performance test**

* Execute JMeter in non-GUI mode.  
* Use a fixed workload so that results can be compared between commits.  
* Generate a .jtl result file and HTML dashboard.

**5\. Compare p95 with the baseline**

* Extract the p95 latency from the current run.  
* Compare it with the baseline from the previous stable commit.  
* Example policy: Regression threshold \= 10%.

**6\. Report the result**

* Store the JMeter HTML report as a CI artifact.  
* Add the result to the pull request.  
* If p95 regression exceeds the threshold, notify the developer.

**Regression policy:** current\_p95 \> baseline\_p95 × 1.10 → Performance regression → Fail / flag CI pipeline

**Otherwise:** current\_p95 ≤ baseline\_p95 × 1.10 → Performance acceptable → Continue pipeline

**11.2 Flow Chart**  
GitHub Commit / PR  
        |  
        v  
Analyze Changed Code  
        |  
        v  
Performance-sensitive change?  
       / \\  
     No   Yes  
     |     |  
     v     v  
   Skip   Build \+ Deploy SUT  
   Perf        |  
   Test        v  
          Run JMeter  
          Fixed Workload  
               |  
               v  
          Extract p95  
               |  
               v  
       Compare with Baseline  
               |  
               v  
          Regression?  
          /        \\  
        Yes        No  
         |          |  
         v          v  
    Flag / Fail    Pass  
    CI Pipeline    CI  
         |          |  
         \+----------+  
              |  
              v  
     Store Report \+ Notify  
          Developer

**11.3 Example Threshold**  
Assume the baseline performance is:

| Metric | Baseline |
| :---- | :---- |
| p95 latency | 1,046 ms |
| Throughput | 437.8 req/s |
| Error rate | 0% |

**Maximum acceptable p95 \= 1,046 × 1.10 \= 1,150.6 ms**

* p95 ≤ 1,150 ms → pass  
* p95 \> 1,150 ms → flag potential regression

The threshold should be calibrated using repeated baseline runs because p95 naturally fluctuates between executions.

**11.4 Cost vs. False Alarm Trade-offs**

Cost  
Running the full Load, Stress, and Spike tests on every commit would be expensive and slow. Therefore, the proposed pipeline uses change-based triggering.

| Change | Performance Test |
| :---- | :---- |
| README/documentation | No |
| CSS/UI only | No |
| Backend API | Yes |
| Database query | Yes |
| Database schema | Yes |
| Authentication logic | Yes |
| Product search | Yes |
| CI configuration | Optional |

A short performance test can run on every relevant PR, while the complete Stress/Spike/Endurance tests can run periodically, such as nightly or before release.

False Alarms  
A single p95 measurement can be noisy because of:

* CPU background processes  
* Database cache state  
* Network variation  
* JVM/JMeter overhead  
* Operating-system scheduling  
* Other processes running on the machine

Therefore, automatically failing a build based on a very small difference can produce false alarms. A practical implementation should use repeated baseline runs, a sufficiently large regression threshold, and consistent test environments to distinguish genuine performance regressions from normal measurement noise.

**13\. AI Critique**

The AI was helpful in designing the performance tests and analysing the JMeter results, but some of its recommendations were incomplete or not suitable for the current SUT. For example, it initially suggested using database connection pooling as a possible optimization. This recommendation is reasonable for server-based databases such as PostgreSQL or MySQL, but the EShop uses SQLite embedded directly in the Node.js process. Multiple connections could increase write contention instead of improving performance. The AI also suggested adding a normal B-Tree index to the product search query, but the query uses `LIKE '%keyword%'`, so a standard index cannot efficiently improve this type of search. The AI did not catch these issues initially because it relied on common database optimization patterns without fully considering the specific architecture and query implementation. I had to review the actual SUT code, database configuration, JMeter results, and raw `.jtl` data to verify its suggestions. Another important limitation was that AI could interpret metrics correctly but could not independently confirm whether the test environment, hardware usage, or workload was representative. From this assignment, I learned that AI should be treated as an assistant rather than the final authority. A good performance-testing process requires giving AI accurate context, checking its assumptions against real evidence, and correcting its conclusions when they do not match the actual system. 

## **14.Conclusion**

This assignment provided practical experience in designing and executing Load, Stress, Spike, and Endurance tests for the EShop backend using JMeter. The tests covered the complete workflow from password reset and login to product search and adding products to the cart. The results showed how system performance changed as the number of concurrent users increased, particularly in terms of throughput, p95 latency, error rate, CPU usage, and memory usage.

The AI analysis was useful for identifying possible performance bottlenecks and optimization ideas, but some recommendations were not suitable for the current SQLite-based architecture. Human review was therefore necessary to verify the results against the raw .jtl data and distinguish feasible improvements from incorrect assumptions.

The endurance test also helped identify the approximate sustained-load limit of the current hardware and SUT. Based on the measured throughput, latency, error rate, and resource usage, the identified endurance threshold provides a practical baseline for future performance testing.

Finally, a continuous performance-testing pipeline was proposed to automatically run performance tests for performance-sensitive changes and detect p95 regressions. This approach can help identify performance problems earlier while controlling testing cost and reducing false alarms. Overall, the assignment demonstrated that performance testing should combine real execution data, resource monitoring, AI-assisted analysis, and human validation rather than relying on a single performance metric.

**15\. Assessment**

| No. | Criteria | Grade | Self-Assessed Grade  |
| :---- | :---- | :---- | :---- |
| 1 | Task 1 — Load testing | 20 | 20 |
| 2 | Task 1 — Stress testing | 20 | 20 |
| 3 | Task 1 — Spike testing | 20 | 20 |
| 4 | Task 2 — AI analysis \+ misinterpretation hunt (with correct values from raw logs) | 10 | 10 |
| 5 | Task 3 — Continuous Performance Testing proposal (G9.6) | 10 | 10 |
| 6 | Task 3 — Continuous Performance Testing proposal (G9.6) | 10 | 10 |
|  | Total | 100 | 100 |

[image1]: <data:image/png;base64,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>