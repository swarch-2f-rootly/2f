# **Performance & Scalability Quality Attribute Scenario: Caching Pattern Implementation and Validation**

---

## **1. System Weakness and Performance Context**

During the performance evaluation of the **Analytics Backend**, it was observed that under heavy concurrent access, multiple clients repeatedly requested identical analytical data, resulting in **redundant database queries** and **increased response latency**.  
This behavior indicated inefficiencies in **resource utilization** and **response time scalability**, as each request triggered a full computation and database retrieval, regardless of data duplication.

To address this bottleneck, the **Caching Pattern (Cache-Aside)** was selected.  
The main objective was to **maintain multiple copies of computed results** in memory, reducing repeated database hits and improving latency under high concurrency.

---

## **2. Quality Attribute Scenario**

![Scenario](../images/scCachingP3.png)

| **Element** | **Description** |
|--------------|-----------------|
| **Architectural Pattern** | Caching – Aside |
| **Architectural Tactic** | Maintain Multiple Copies of Computations (Manage resources) |
| **Artifact** | Analytics Backend |
| **Source** | Concurrent web or mobile clients repeatedly requesting the same data |
| **Stimulus** | A burst of 4000 GET requests within 20 seconds, all targeting an identical resource |
| **Environment** | Normal operations |
| **Response** | Processes each request (each triggering a full database query), records latency and query statistics in monitoring logs |
| **Response Measure** | Average request latency (ms) |

---

## **3. Baseline Load Test (Before Caching Implementation)**

![post-lb-performance](../images/con_lbGraphql_analytics_performance_avg_3iter.png)

| **Metric** | **Best (1 user)** | **Knee (users)** | **Max Load (4000 users)** |
|-------------|------------------|------------------|----------------------------|
| **Avg Response Time (ms)** |  |  |  |
| **P95 (ms)** |  |  |  |
| **P99 (ms)** |  |  |  |
| **Error Rate (%)** |  |  |  |
| **Throughput (req/s)** |  |  |  |

**Analysis:**  
Before applying caching, the system experienced **repeated database queries for identical data**, causing **high latency** and **CPU-bound query processing** as concurrency increased.

---

## **4. Countermeasure Implementation: Caching Pattern**

The **Cache-Aside pattern** was implemented within the analytics backend to store frequently accessed query results in memory.  
The main configuration included:

- In-memory cache layer (e.g., Redis or local caching module).  
- TTL (Time-To-Live) policy to ensure freshness of cached data.  
- Cache invalidation rules for data updates.  
- Integration with backend metrics for cache hit/miss analysis.  

**Expected Outcome:**  
- Reduction in average and percentile response times.  
- Decrease in redundant database queries.  
- Increased throughput under concurrent identical requests.  
- Stable response time variance due to memory-based retrieval.

---

## **5. Validation Results (After Caching Implementation)**

![post-cache-performance](../images/con_cache_performance.png)

| **Metric** | **Best (1 user)** | **Knee (users)** | **Max Load (4000 users)** |
|-------------|------------------|------------------|----------------------------|
| **Avg Response Time (ms)** |  |  |  |
| **P95 (ms)** |  |  |  |
| **P99 (ms)** |  |  |  |
| **Error Rate (%)** |  |  |  |
| **Throughput (req/s)** |  |  |  |

**Analysis:**  


---

## **6. Performance Metrics Comparison**

| **Metric** | **Before Caching** | **After Caching** | **Observation / Technical Impact** |
|-------------|--------------------|-------------------|------------------------------------|
| **Average Response Time (ms)** |  |  |  |
| **Response Time Variance (%)** |  |  |  |
| **Throughput (req/sec)** |  |  |  |
| **Database Query Load (%)** |  |  | |
| **CPU Utilization (avg)** |  |  |  |
| **Cache Hit Ratio (%)** | — |  | |

---

## **7. Technical Insights**

---

## **8. Response to Quality Scenario**

| **Attribute** | **Observation** |
|----------------|-----------------|
| **Performance** |  |
| **Scalability** ||
| **Resource Efficiency** |  |
| **Reliability** |  |

---

## **9. Conclusion**




