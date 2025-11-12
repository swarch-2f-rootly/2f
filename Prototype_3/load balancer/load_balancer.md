# **Performance & Scalability Quality Attribute Scenario: Load Balancing Pattern Implementation and Validation**

---

## **1. System Weakness and Performance Context**

During the performance assessment of the **Analytics Backend**, the `graphql_analytics` endpoint exhibited signs of **scalability limitations** under increasing concurrent user loads.  
While the service maintained stable error rates (0%), response latency and throughput trends revealed **non-linear degradation** as user concurrency exceeded approximately **3000 users**.  
This indicated a performance saturation point caused by **uneven request distribution** and **CPU-bound request processing** within a single backend instance.

To mitigate this bottleneck and improve horizontal scalability, the **Load Balancer pattern** was selected and implemented to distribute traffic evenly across multiple backend containers.

---

## **2. Quality Attribute Scenario**

![Scenario](../images/scLoadBalancerP3.png)

| **Element** | **Description** |
|--------------|-----------------|
| **Artifact** | Analytics Backend — GraphQL analytics endpoint |
| **Source** | Multiple external users concurrently sending analytics requests |
| **Stimulus** | 4000 HTTP requests generated within a 30-second interval |
| **Environment** | Normal operation under synthetic load testing |
| **Response** | System processes all requests, logging latency and HTTP status outcomes |
| **Response Measure** | Primary metrics: Response time variance (%) and failed request rate per test period |

---

## **3. Baseline Load Test (Before Load Balancer)**

![baseline-performance](../images/sin_lb_-_graphql_analytics_performance_20251111_110940.png)

| **Metric** | **Best (1 user)** | **Knee (3101 users)** | **Max Load (4000 users)** |
|-------------|------------------|-----------------------|----------------------------|
| **Avg Response Time (ms)** | 87.30 | 4485.68 | 6840.71 |
| **P95 (ms)** | — | 8139.73 | — |
| **P99 (ms)** | — | 12685.69 | — |
| **Error Rate (%)** | 0.00 | 0.00 | 0.00 |
| **Throughput (req/s)** | 5.57 | 94.50 | 52.90 |

**Analysis:**  
The system showed **acceptable performance up to ~3000 concurrent users**. Beyond this point, **response times increased exponentially** (from ~4.5s to ~6.8s), while throughput declined by almost **44%**, indicating **resource contention** and lack of horizontal scaling capabilities.

---

## **4. Countermeasure Implementation: Load Balancer Pattern**

A **Load Balancer** was introduced in front of the analytics backend cluster to enable **request distribution** across multiple instances.  
The configuration applied included:

- Round-robin routing strategy  
- Health checks and failover logic  
- Disabled session persistence to prevent node saturation  
- Continuous metric collection via Prometheus and Grafana  

**Expected Outcome:**  
- Reduced **average and percentile response times**  
- Increased **throughput** due to balanced concurrency  
- Extended **knee point**, reflecting improved scalability

---

## **5. Validation Results (After Load Balancer)**

![post-lb-performance](../images/con_lbGraphql_analytics_performance_avg_3iter.png)

| **Metric** | **Best (1 user)** | **Knee (3401 users)** | **Max Load (4000 users)** |
|-------------|------------------|-----------------------|----------------------------|
| **Avg Response Time (ms)** | 235.67 | 3520.17 | 3520.19 |
| **P95 (ms)** | — | 5272.00 | — |
| **P99 (ms)** | — | 7829.25 | — |
| **Error Rate (%)** | 0.00 | 0.00 | 0.00 |
| **Throughput (req/s)** | 5.86 | 113.96 | 118.30 |

**Analysis:**  
After deploying the Load Balancer:
- The **knee point increased** from **3101 → 3401 concurrent users**, reflecting a **10% scalability improvement**.  
- **Average response time** at the knee decreased by **~21% (4486 → 3520 ms)**.  
- **P95/P99 latencies improved** by roughly **35–40%**.  
- **Throughput increased** significantly (**94.5 → 113.9 req/s at knee**, and **118.3 req/s at max load**).  
- **Error rate remained 0%**, confirming **stability under higher load**.

---

## **6. Response to Quality Scenario**

| **Attribute** | **Observation** |
|----------------|-----------------|
| **Performance** | The Load Balancer reduced latency and improved throughput, maintaining stable service quality under higher concurrency. |
| **Scalability** | The system achieved a higher concurrency threshold before saturation (knee shifted +300 users), validating horizontal scalability. |
| **Reliability** | Error-free operation under 4000 concurrent requests demonstrates consistent backend availability and effective load distribution. |

---

## **7. Conclusion**

The **Load Balancer pattern** successfully mitigated the initial performance bottleneck by distributing incoming traffic evenly across multiple backend instances.  
Post-deployment metrics confirm measurable improvements in **response time**, **throughput**, and **scalability tolerance**, fulfilling the **Performance and Scalability** quality objectives for the Analytics Backend.

---

