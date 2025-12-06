# Rootly 
## Prototype 4 - Quality Attributes, Part 2

## Team 2F
- Carlos Santiago Sandoval Casallas
- Cristian Santiago Tovar Bejarano
- Danny Marcelo Yaluzan Acosta
- Esteban Rodriguez Muñoz
- Santiago Restrepo Rojas
- Gabriela Guzmán Rivera
- Gabriela Gallegos Rubio
- Andrés Camilo Orduz Lunar

## Table of Contents

- [Software System](#software-system)
- [Architectural Structures](#architectural-structures)
  - [Components and Connector Structure](#components-and-connector-view)
  - [Deployment Structure](#deployment-view)
  - [Layered Structure](#layered-view)
  - [Decomposition Structure](#decomposition-view)
- [Quality Attributes](#quality-attributes)
  - [Security](#security)
    - [Network Segmentation](#network-segmentation)
    - [Secure Channel](#secure-channel)
    - [Reverse Proxy](#reverse-proxy)
    - [Web Application Firewall](#web-application-firewall)
  - [Performance and Scalability](#performance-and-scalability)
    - [Load Balancer](#load-balancer)
    - [Caching](#caching)
  - [Reliability](#reliability)
    - [Replication pattern lb analytics](#replication-pattern-lb-analytics)
    - [Cluster pattern](#cluster-pattern)
    - [Replication pattern db caching](#replication-pattern-db-caching)
    - [Service Discovery](#service-discovery)
  - [Interoperability](#interoperability)
  
- [Prototype – Deployment Instructions](#deployment-instructions)


## Software System
- **Name:** Rootly  
- **Logo:**  
<img src="images/Rootly.png" alt="Rootly Logo" width="100"/>

**Description:**
  
**ROOTLY** is an agricultural monitoring system designed to support significant decision-making in the agricultural environment. It leverages a microservices-based architecture to integrate field devices, process data, and deliver real-time analytics to users through a web and mobile application.

The system operates by capturing environmental and soil data—such as humidity and temperature—directly from the field using microcontroller devices. This information is then sent to a central platform where it is processed, validated, and analyzed. The platform's architecture combines robust databases for storing both transactional information (like user profiles and configurations) and large volumes of time-series sensor data.

Finally, users can access all this information through an intuitive interface, available on both web and mobile. They can view real-time metrics, explore historical data, manage their crops, and receive analytical insights to optimize their agricultural practices.
  
---

  
## Quality Attributes

## Security
### Network Segmentation

![network-segmentation](images/scNetworkSegmentationP3.png)

#### 1. Artifact

**Critical Backend Components:** All system services and data stores that must remain internal and protected. This includes:
- **API Gateway** (`api-gateway`): Central routing and orchestration service
- **All Microservices** (`be-*`): Backend business logic services
- **All Databases** (`db-*`): PostgreSQL and InfluxDB instances
- **Message Broker**: Kafka queue system (`queue-data-ingestion`)
- **Storage Layers** (`stg-*`): MinIO object storage

#### 2. Source

An **External Malicious Actor** (individual or automated bot) originating from the **public Internet** (i.e., outside the defined internal network boundary).

**Actor Characteristics:**
- **Knowledge Level**: Network scanning expertise
- **Tools**: Port scanners (nmap), HTTP clients (curl), database clients (psql)
- **Intent**: Unauthorized access to internal services and data
- **Origin**: External public network, outside the trusted perimeter

#### 3. Stimulus

The actor executes a series of network probes: **Direct External Connection Attempts** to specific, known internal service ports. The attempts (via tools like `cURL`, `psql`) target host ports potentially mapped to backend components (e.g., 8080, 5432, 8000-8003). 

**Specific Attack Actions:**
1. **Network Reconnaissance**: Port scanning to discover exposed services
2. **Service Fingerprinting**: Identifying service types and versions
3. **Direct Connection Attempts**: Bypassing the frontend to access backend services directly
4. **Database Access Attempts**: Attempting to connect to exposed database ports

The focus is on **any traffic originating from outside the trusted internal network segment**.

#### 4. Environment

The system is under **Normal Operation** with its **Current Network Configuration**. This configuration can be either:

#### Pre-Segmentation (Baseline)
The network configuration (e.g., single, flat Docker network) results in unintended port exposures to the host interface. In this state:
- All services share a single Docker network
- Multiple backend services have host port mappings
- External attackers can potentially discover and access internal services
- No network-level isolation between public and private components

#### Post-Segmentation (Validation)
The architecture utilizes the **Network Segmentation Pattern** (Public/Private isolated networks) where:
- Services are distributed across two isolated networks
- **No host port mappings** exist for backend services
- Only the frontend has external access
- Backend services communicate exclusively via the private network

#### 5. Response

The network infrastructure must **Process the External Connection Request** directed at the internal component. The network layer's response will be one of three outcomes:

1. **Connection Granted (Success)**: A TCP/IP connection is successfully established
   - Attacker gains direct access to the internal service
   - Can potentially bypass application-level authentication
   - Represents a breach of the network perimeter

2. **Connection Refused (Denial)**: An immediate rejection occurs due to no process listening or firewall rules
   - No service is listening on the requested port
   - Port mapping does not exist
   - Represents successful network isolation

3. **Timeout (Denial)**: No response is received within the standard time limit
   - Traffic is dropped by firewall or routing rules
   - Represents partial security measure

The system/network tools must accurately **Log the Network Access Outcome** for every external attempt, enabling measurement and validation.

#### 6. Response Measure

The system's security is validated by the total count of established connections, which represents a successful breach of the network perimeter:

**Primary Metric:**
$$\text{Total Successful External Connections} = \sum_{i=1}^{n} \text{Granted Connection}_i$$

Where:
- $n$ = total number of connection attempts to internal services
- Each $\text{Granted Connection}_i \in \{0, 1\}$ (0 = denied, 1 = granted)

**Measurement Details:**
- **Testing Period**: One complete test suite run targeting all internal service ports
- **Target Ports**: All ports used by backend services (8080, 8000-8003, 5432, 8082, etc.)
- **Success Criteria**: Connection establishment (HTTP 200 response, database connection, etc.)

**Performance Target:**

| Environment | Target Value | Interpretation |
|-------------|--------------|----------------|
| Pre-Segmentation (Baseline) | > 0 (Expected: 5-8) | Vulnerability present - attack surface exposed |
| Post-Segmentation (Goal) | **= 0** | **Security objective achieved** - complete network isolation |

**Interpretation:**
- **Count = 0**: Network segmentation is effective - no external access to internal services
- **Count > 0**: Security failure - at least one internal service is accessible from external networks, regardless of application-level authorization status

This metric directly measures the effectiveness of the network perimeter and validates whether the Network Segmentation Pattern successfully implements the **Limit Access** security tactic.

#### Countermeasure: Network Segmentation Pattern

The Network Segmentation Pattern mitigates this security scenario by implementing a defense-in-depth strategy that isolates backend services from external access through:

- Public/Private Network Isolation: Creating separate rootly-public-network and rootly-private-network

- Port Mapping Elimination: Removing all host port mappings from backend services

- Single Entry Point: Only frontend service remains publicly accessible

- Internal Network Communication: Backend services communicate exclusively via private network

This approach implements the "Limit Access" security tactic by ensuring external attackers cannot directly reach internal services, forcing all traffic through the authenticated frontend gateway.

#### Comparative Security Assessment


| Metric | Pre-Segmentation | Post-Segmentation | Improvement |
|--------|------------------|-------------------|-------------|
| **Total Successful External Connections** | 5+ (Vulnerable) | 0 (Target Achieved) | 100% |
| **Exposed Attack Surfaces** | 8+ ports | 1 port (frontend only) | 87.5% reduction |
| **API Gateway Direct Access** | ✓ Connection successful | ✗ Connection refused | Blocked |
| **Backend Services Access** | ✓ Connection successful | ✗ Connection refused | Blocked |
| **Database Direct Access** | ✓ Connection possible | ✗ Connection refused | Critical vulnerability eliminated |
| **Admin Interfaces Exposure** | ✓ Accessible | ✗ Connection refused | Blocked |
| **Authentication Bypass Possible** | Yes | No | Security control enforced |


#### Summary

The Network Segmentation Pattern addresses a critical architectural weakness in Rootly's initial deployment: a flat network architecture where all components shared a single Docker network with multiple host port mappings, exposing internal services (API Gateway, databases, message brokers) directly to external networks.

**Security Scenario:** An external malicious actor attempts direct connections to internal services, bypassing frontend authentication and authorization mechanisms. The quality scenario measures the **total count of successful external connections** to internal components as the primary security metric.

**Countermeasure Implementation:** The system was restructured into two isolated network zones:
- **Public Network (`rootly-public-network`):** Contains only the frontend with controlled external access
- **Private Network (`rootly-private-network`):** Isolated internal network (marked as `internal: true`) containing all backend services, databases, and infrastructure with no port mappings to the host

This implements the **Limit Access** security tactic (Resist Attacks category), establishing network-level perimeter defense as a complementary layer to application-level security.

#### Verification – Comparative Analysis

| Aspect | **Before Network Segmentation** | **After Network Segmentation** |
|--------|----------------------------------|--------------------------------|
| **Attack Surface** | 8+ services exposed through host port mappings | Only frontend exposed (single entry point) |
| **External Connection Attempts** | API Gateway, backend services, PostgreSQL, Kafka UI | All internal services tested |
| **Response** | Connection granted to API Gateway, backend services, and databases | Connection refused for all internal services |
| **Response Measure** | **5+ successful external connections** to internal services | **0 successful external connections** to internal services |
| **CIA Impact** | Confidentiality, Integrity, and Availability at risk | All three properties protected by network isolation |

**Validation Outcome:** The Network Segmentation Pattern successfully eliminated the architectural vulnerability, achieving a **100% reduction in external attack surface**. All 5+ previously vulnerable services are now completely inaccessible from external networks, while internal service communication remains fully functional through Docker's private network DNS resolution. The security objective of **zero successful external connections** was achieved and validated through simulated attack scenarios.


---
## Secure Channel

![Secure Channel Scenario](images/secure_channel.png)

#### 1. Artifact

**Frontend Web Service (`fe-web`)**: The main entry point for user interaction, responsible for transmitting sensitive data (credentials, tokens, sensor payloads) between the browser and backend services.

#### 2. Source

**Network Attacker**: Any internal or external actor with access to the network (Wi-Fi, LAN, Docker bridge, etc.) capable of intercepting traffic between the browser and frontend.

####  3. Stimulus

The attacker uses packet capture tools (Wireshark, tcpdump) to intercept HTTP traffic, aiming to read credentials, tokens, and user data in plaintext.

####  4. Environment

#### Pre–Secure Channel (Baseline)
- Communication between browser and frontend occurs over unencrypted HTTP.
- All requests and responses are readable in transit.
- Sensitive data is exposed to anyone with network access.

#####  Post–Secure Channel (Validation)
- HTTPS/TLS is enforced for all browser–frontend communication.
- All packets are encrypted; only TLS handshake and encrypted application data are visible.

####  5. Response

The system must ensure that all sensitive data in transit is protected from interception and tampering:
- **Encryption:** All traffic is encrypted using TLS.
- **Integrity:** Data cannot be modified without detection.
- **Authentication:** Only trusted endpoints are accessible.

####  6. Response Measure

The effectiveness of the Secure Channel is measured by the number of readable packets containing sensitive data:

| Metric                      | Pre–Secure Channel | Post–Secure Channel | Target      |
|-----------------------------|--------------------|---------------------|-------------|
| Readable packets/session    | ≥ 5                | 0                   | 0           |
| Credentials exposed         | Yes                | No                  | No          |
| Data tampering possible     | Yes                | No                  | No          |

####  Security Concepts Overview

| Concept         | Description in Secure Channel Scenario                  |
|-----------------|--------------------------------------------------------|
| **Weakness**    | No encryption between browser and frontend             |
| **Threat**      | Network attacker intercepts HTTP traffic               |
| **Attack**      | Packet capture and analysis of sensitive data          |
| **Risk**        | Data breach, credential theft, session hijacking       |
| **Vulnerability**| Plaintext transmission of sensitive information       |
| **Countermeasure**| Enforce HTTPS/TLS (Secure Channel Pattern)           |

####  Countermeasure: Secure Channel Pattern

The Secure Channel Pattern mitigates the risk by:
- Enforcing HTTPS/TLS for all browser–frontend communication.
- Generating and installing TLS certificates for the frontend service.
- Updating all endpoints and environment variables to use `https://`.
- Validating that intercepted packets are encrypted and unreadable.

####  Validation – Before vs. After

| Aspect                | Before Secure Channel           | After Secure Channel            |
|-----------------------|---------------------------------|---------------------------------|
| **Response**          | Attacker intercepts and reads all HTTP traffic | All traffic is encrypted; attacker cannot read data |
| **Response Measure**  | Credentials, tokens, payloads exposed | No sensitive data exposed; only encrypted packets |
| **Confidentiality**   | Compromised                     | Protected                       |
| **Integrity**         | Vulnerable to tampering         | Protected by TLS                |
| **Functionality**     | Normal operation                | Normal operation (no impact)    |

####  Summary

Implementing the Secure Channel Pattern eliminates the exposure of sensitive data in transit, blocks credential theft and session hijacking, and preserves system functionality. All browser–frontend communication is now encrypted, fulfilling the security objective for data-in-transit protection.

---
###  Reverse Proxy

Rootly implements the reverse proxy pattern inside the Web Application Firewall (WAF). The WAF is therefore the only public-facing component for frontend traffic, simultaneously enforcing inspection rules and the proxy features described below.

![Reverse proxy flood scenario](images/reverse_proxy_sceneryP3.png)

####  1. Artifact

**Ingress Path for HTTP/REST Traffic:** Public-facing connector that carries requests from `fe-mobile`, `fe-web` (through the WAF), and automation clients toward the `api-gateway` and all downstream microservices.

- **WAF Reverse Proxy / Edge Layer:** The WAF component hosts the reverse proxy logic that all external HTTP requests traverse.
- **API Gateway (`api-gateway`):** Central orchestrator whose overload cascades to the rest of the platform.
- **Backend Microservices:** Analytics, authentication, plant management, and processing services that depend on a healthy gateway.

####  2. Source

**Botnet or Automated Scraper.** Distributed actors, or a single runaway integration, capable of generating sustained HTTP floods from the public internet.

**Characteristics:**
- Tooling: `hey`, `wrk`, `k6`, or custom scripts.
- Knowledge: Public `/api/*` routes exposed to clients.
- Objective: Deny service by starving shared resources rather than stealing data.

####  3. Stimulus

The threat launches **high-concurrency bursts** against popular REST endpoints:
1. Baseline probe verifies available routes (e.g., `/api/v1/metrics`).
2. Flood workload issues hundreds/thousands of requests per second with aggressive concurrency.
3. Payloads mix valid and malformed data to force parsing, authentication, and routing on every request.

### 4. Environment

Scenario executed under normal operations while comparing two configurations.

#### Pre–Reverse Proxy (Baseline)
- `api-gateway` is mapped directly to a host port that mobile/web clients use.
- No edge rate-limiting, caching, or inspection exists.
- Every flood packet reaches the gateway and propagates to the microservices it fronts.

#### Post–Reverse Proxy (Validation)
- The WAF reverse proxy is the only public HTTP/REST connector; all clients traverse `web-browser/fe-mobile → WAF (reverse proxy) → api-gateway`.
- `api-gateway` and downstream services are isolated on a private network without host port mappings.
- The WAF enforces per-IP/per-route throttling, optional caching, and centralized logging to monitor ingress.

### 5. Response

When the WAF reverse proxy is **not** yet implemented, the system responds to floods in a brittle way:

1. **Edge Throttling:** Does **not** exist; every request hits `api-gateway`, so there is no HTTP `429` shedding at the perimeter.
2. **Burst Absorption:** Short spikes are still directed straight to the gateway, which has limited ability to buffer traffic and therefore starves worker pools.
3. **Selective Forwarding:** All traffic—legitimate and abusive—travels through the exposed HTTP/REST connector, so the gateway forwards overload to backends.
4. **Visibility:** Observability is fragmented across services, making it harder to identify the dominant attacker IPs or routes in real time.

This lack of an edge reverse proxy means `api-gateway` must process the entire flood, leading to CPU saturation, increased latency, and cascading 5xx errors. The remainder of this section explains how embedding the reverse proxy inside the WAF fixes those gaps.

####  6. Response Measure

Validation focuses on runtime metrics collected during the flood test:

- **P95 Latency (Legitimate Traffic):** Must remain within SLA even when floods are active.
- **Forwarded RPS to `api-gateway`:** Bounded regardless of incoming attack volume.
- **HTTP 429 Count:** Demonstrates abusive traffic is rejected before reaching backends.
- **Backend Error Rate (5xx):** Should stay near baseline with protection enabled.

| Metric | Pre–Reverse Proxy | Post–Reverse Proxy | Interpretation |
| --- | --- | --- | --- |
| **P95 latency (legit traffic)** | >3–5 s | <200–300 ms | Availability only preserved with proxy in place. |
| **Gateway-observed RPS** | Mirrors attack (~1000 RPS) | Capped (~200–300 RPS) | Proxy keeps downstream load bounded. |
| **HTTP 5xx rate** | 20–40% | <2% | Failures drop because services avoid overload. |
| **HTTP 429 rate** | 0 | High (shed traffic) | Edge now rejects abusive bursts immediately. |

####  Security Concepts Overview

| Concept | Description in the Reverse Proxy Scenario |
| --- | --- |
| **Weakness** | Direct exposure of `api-gateway` without edge filtering or caching. |
| **Threat** | Botnet, scraper, or runaway integration capable of high-RPS floods. |
| **Attack** | Massive concurrent REST calls hammer `/api/*` endpoints to exhaust resources. |
| **Risk** | Farmers and operators lose analytics/telemetry access due to timeouts and 5xx responses. |
| **Vulnerability** | Unbounded ingress path allows every attack packet to reach internal services. |
| **Countermeasure** | Implement the reverse proxy inside the WAF between clients and `api-gateway`, enforcing throttling, caching hot responses, and centralizing inspection. |

####  Countermeasure: Reverse Proxy Pattern

The **Reverse Proxy Pattern** establishes a guarded ingress path:

- The WAF reverse proxy is the only service exposed publicly; `api-gateway` resides on a private network and is reachable solely through the WAF.
- The HTTP/REST connector between `fe-mobile` and the backend now includes rate limiting, burst controls, and optional caching to keep forwarded RPS within safe envelopes.
- Observability improves because every external HTTP request is logged in one place, accelerating detection and response.

####  Validation – Before vs. After

| State | Response | Response Metrics |
| --- | --- | --- |
| **Before reverse proxy** | `api-gateway` processes every spike, saturates CPU, and propagates latency/timeouts to clients. | P95 latency >3 s, backend RPS ≈ attack RPS (~1000), 20–40% 5xx, no `429` shedding. |
| **After reverse proxy** | The WAF reverse proxy sheds overflow (HTTP 429) and forwards only bounded traffic through the HTTP/REST connector, keeping services responsive. | P95 latency <300 ms, forwarded RPS capped (~200–300), <2% 5xx, high `429` count evidencing throttling. |

####  Comparative Security Assessment

| Metric | Pre–Reverse Proxy | Post–Reverse Proxy | Improvement |
| --- | --- | --- | --- |
| **Protected entry points** | None – gateway exposed | Proxy between clients and gateway | Single hardened ingress |
| **Attack traffic reaching backends** | 100% of flood | <30% (rate-limited) | >70% reduced |
| **Client experience during flood** | Timeouts and failures | SLA respected | Availability restored |
| **Detection & observability** | Distributed per-service logs | Centralized at proxy | Faster triage |

#### Summary

Adopting the WAF-hosted reverse proxy converted an unbounded ingress path into a controlled choke point that enforces the **Limit Access** tactic. Legitimate `fe-mobile` sessions maintain service quality even when hostile traffic is present, because overload is absorbed and rejected at the edge.

###  Verification – Comparative Analysis

| Aspect | **Before Reverse Proxy** | **After Reverse Proxy** |
| --- | --- | --- |
| **Response** | API gateway and downstream services absorb the entire flood, leading to saturation, restarts, and user-visible downtime. | The WAF reverse proxy throttles the attack, forwards only safe RPS to the gateway, and keeps legitimate sessions responsive. |
| **Response Measure** | High latency, backend RPS mirrors attack volume, zero 429 shedding, elevated 5xx. | Latency within SLA, bounded backend RPS, significant 429 shedding, minimal 5xx. |

---
###  Web Application Firewall
![Web Application Firewall scenario](images/WAFPatternScenario.png)

####  Scenario Snapshot
In this scenario, the system faces a sustained **Layer-7 DoS attack** initiated by a single malicious client repeatedly sending legitimate-looking HTTP requests to its public API endpoints.  
Initially, a plain NGINX reverse proxy acts as the only public entry point, forwarding all traffic to the API Gateway without deep inspection or rate limiting.  
This design exposes the system to resource exhaustion, high latency, and loss of availability.  
To mitigate this weakness, the **Web Application Firewall (WAF) pattern** is applied by adding a WAF as an additional edge component in front of the existing reverse proxy, introducing intelligent traffic inspection and rate control mechanisms that detect and block malicious requests while maintaining service performance for legitimate users.

- **Weakness:** The system exposes a single public entry point through an NGINX reverse proxy that only forwards traffic to the API Gateway without deep inspection or centralized rate limiting.
- **Threat:** A malicious external client or automated script capable of generating a sustained stream of HTTP(S) requests that simulated legitimate traffic.
- **Attack:**  A **Layer-7 DoS** attack that repeatedly targets exposed endpoints (e.g., `/graphql`, `/auth/login`), using continuous or “low-and-slow” requests to exhaust gateway worker pools and degrade performance.
- **Risk:** The API Gateway becomes overloaded, producing 502/503 responses and blocking legitimate users. System availability and user experience deteriorate severely.
- **Vulnerability:** Absence of application-layer protection and global throttling. The reverse proxy lacks mechanisms to identify and block abusive request patterns.
- **Countermeasure:** Integrate a **Web Application Firewall (WAF)** with the reverse proxy/API Gateway edge so that every request is inspected, filtered, and throttled before it reaches backend services.

####  Explanation of the Countermeasure

The **Web Application Firewall (WAF) pattern** introduces a dedicated layer for **application-layer inspection and traffic control**.  
It applies the *Detect Service Denial* and *Limit Resource Demand* architectural tactics to strengthen the system’s availability and resilience.

By adding a WAF as an additional edge component in front of the existing reverse proxy (rootly-waf), the system gains the ability to:

- **Analyze and classify** HTTP requests using the OWASP Core Rule Set (CRS).  
- **Detect anomalies and block malicious traffic** before it reaches backend services.  
- **Apply dynamic rate limiting** by IP, route, or payload size.  
- **Maintain service availability** with minimal latency degradation during sustained request floods.
- **Centralize telemetry** (blocked IPs, triggered rules, anomaly scores) for auditing and adaptive security tuning.

This countermeasure mitigates the initial weakness by placing an **intelligent inspection and throttling layer** at the network edge, ensuring that the previously passive reverse proxy receives only validated and rate-controlled traffic, which prevents API Gateway overload under Layer-7 DoS conditions.

####  Summary

#### Verification 

| Aspect | **Before WAF (Reverse Proxy Only)** | **After WAF (WAF Pattern Applied)** |
|--------|-------------------------------------|--------------------------------------|
| **Response** | Reverse proxy forwards all requests directly to the API Gateway, allowing a single malicious client to overload the service and cause unresponsiveness. | WAF inspects and throttles abusive traffic, forwarding only traffic classified as legitimate to the reverse proxy and then to the API Gateway. |
| **Response Measure** | API Gateway latency > 5 s, 502/503 errors after ~30 s, availability < 60% under sustained Layer-7 attack. | **99.2%–99.8% of hostile traffic blocked** before reaching the reverse proxy and API Gateway; no HTTP 502/503 errors were observed during the post-mitigation attack runs, and allowed requests maintained **p75 latencies below 50 ms**, consistent with normal operation. |

The implementation of the WAF pattern effectively mitigates application-layer DoS attacks by inspecting and blocking malicious request patterns before they reach the reverse proxy and API Gateway. Quantitative validation shows that the WAF filters **more than 99% of hostile traffic** during the attack scenario while keeping latency for legitimate requests low and avoiding gateway errors.

These results confirm a substantial improvement in **availability**, **performance stability**, and **observability**: the system remains responsive under hostile traffic, legitimate clients experience stable response times, and WAF logs provide an auditable trail of triggered rules and detected attacks.

---

##  Performance and Scalability
###  Load Balancer

![Scenario](images/scLoadBalancerP3.png)
During peak usage, approximately **4,000 HTTP requests were sent within 1 or 2 seconds** (to simulate concurrency) from multiple external clients accessing the `/graphql_analytics` endpoint. Forwarded all requests directly to a single backend instance, causing **increased response times, uneven workload distribution, and CPU saturation**.  Although the system remained functional, **response time variance and throughput degradation** became evident as concurrency grew beyond ~3,000 users, exposing limitations in scalability and responsiveness.


| **Element** | **Description** |
|--------------|-----------------|
| **Artifact** | Analytics Backend — GraphQL analytics endpoint |
| **Source** | Multiple external users concurrently sending analytics requests |
| **Stimulus** | 4000 HTTP requests generated within a 2-second interval |
| **Environment** | Normal operation under synthetic load testing |
| **Response** | System processes all requests, logging latency and HTTP status outcomes |
| **Response Measure** | Primary metrics: Response time variance (%) and failed request rate per test period |

####  Baseline Load Test (Before Load Balancer)

![baseline-performance](images/sin_lbGraphql_analytics_performance.png)

####  Countermeasure Implementation: Load Balancer Pattern

**Load Balancer** was introduced in front of the analytics backend cluster to enable **request distribution** across multiple instances.  
The configuration applied included:
- Round-robin routing strategy  
- Health checks and failover logic  
- Disabled session persistence to prevent node saturation  
- Continuous metric collection via Prometheus and Grafana

####   Implementation Load Balancer Results**

![post-lb-performance](images/con_lbGraphql_analytics_performance_avg_3iter.png)

####  Performance Metrics Comparison

| **Metric** | **Before Load Balancer** | **After Load Balancer** | **Observation / Technical Impact** |
|-------------|---------------------------|---------------------------|------------------------------------|
| **Average Response Time (ms)** | 620 ms | 285 ms | ↓ Response time reduced by ~54%, indicating improved distribution and reduced queueing latency. |
| **Response Time Variance (%)** | 42% | 11% | ↓ Variance significantly stabilized, meaning more predictable latency under load. |
| **Throughput (req/sec)** | 133 req/s | 260 req/s | ↑ System throughput nearly doubled due to concurrent backend processing. |
| **Failed Requests (%)** | 6.5% | 0.3% | ↓ Error rate almost eliminated after introducing traffic balancing. |
| **CPU Utilization (per instance)** | ~95–100% (single node) | ~55–65% (per node, 2 replicas) | ↓ Load evenly distributed across replicas, avoiding CPU saturation. |
| **Network Latency (avg)** | 78 ms | 43 ms | ↓ Reduced network wait times between client and backend. |
| **Scalability Behavior** | Linear degradation under stress | Stable performance across replicas | ↑ Horizontal scalability achieved via load distribution. |
| **System Availability** | Degraded under concurrent load | Sustained at 99%+ | ↑ Improved reliability and uptime under concurrent access. |

####  Summary

The **Load Balancer pattern** successfully mitigated the initial performance bottleneck by distributing incoming traffic evenly across multiple backend instances.  
Post-deployment metrics confirm measurable improvements in **response time**, **throughput**, and **scalability tolerance**, fulfilling the **Performance and Scalability** quality objectives for the Analytics Backend.

###  Caching

![Scenario](images/scCachingP3.png)

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

###  Baseline Load Test (Before Caching Implementation)

![post-lb-performance](images/con_lbGraphql_analytics_performance_avg_3iter.png)

| **Metric** | **Best (1 user)** | **Knee (3401 users)** | **Max Load (4000 users)** |
|-------------|------------------|-----------------------|----------------------------|
| **Avg Response Time (ms)** | 235.67 | 3520.17 | 3520.19 |
| **P95 (ms)** | — | 5272.00 | — |
| **P99 (ms)** | — | 7829.25 | — |
| **Error Rate (%)** | 0.00 | 0.00 | 0.00 |
| **Throughput (req/s)** | 5.86 | 113.96 | 118.30 |

---
###  Countermeasure Implementation: Caching Pattern

The **Cache-Aside pattern** was implemented within the analytics backend to store frequently accessed query results in memory.  
The main configuration included:

- In-memory cache layer (Redis).  
- TTL (Time-To-Live) policy to ensure freshness of cached data.  
- Cache invalidation rules for data updates.  
- Integration with backend metrics for cache hit/miss analysis.  

---

## Reliability

### Replication pattern lb analytics

### Cluster pattern

### Replication pattern db caching

### Service Discovery

Service Discovery ensures that internal callers (e.g., `api-gateway`, `ms-user-plant-management`) always find the authentication service even as replicas are restarted or rescheduled. Observability relies on **Docker logs** to detect registration or resolution issues early and drive remediation.

![Service discovery scenario](images/Service-Discovery-Pattern.png)

#### 1. Artifact

**Service Lookup Path for Authentication:** Runtime resolution of the `be-authentication-and-roles` service name to healthy container instances.

- **Service Registry / Docker DNS:** Built-in Docker resolver providing name-to-IP mapping for containers on `rootly-network`.
- **be-authentication-and-roles:** Auth/RBAC microservice whose availability is critical to all request flows.
- **Internal Clients:** `api-gateway`, `ms-user-plant-management`, and background jobs that call authentication APIs.
- **Observability:** Centralized `docker compose logs` stream used to detect lookup errors (`no such host`, `connection refused`) and container health changes.

#### 2. Source

**Internal service callers** (gateway and backend microservices) performing REST calls against `http://be-authentication-and-roles:8000`. They depend on name resolution and healthy targets to proceed.

#### 3. Stimulus

- A new replica of `be-authentication-and-roles` is started or restarted.
- A caller issues authentication or role checks immediately after the change.
- DNS propagation or container health takes a few seconds to stabilize.

#### 4. Environment

Normal operations with Docker Compose on `rootly-network`, dynamic container restarts/scaling, and aggregated logs collected via `docker compose logs -f`.

#### 5. Response

- **Before Service Discovery discipline:** Callers experience intermittent `502/503` or `no such host be-authentication-and-roles` while the new container comes up; failures surface only when user flows break.
- **After Service Discovery discipline:** Docker DNS provides up-to-date mappings; log-based observers watch for resolution errors and unhealthy containers, triggering a fast restart or alert when needed. Callers automatically pick a healthy instance once available.

#### 6. Response Measure

Validation driven by Docker log telemetry and HTTP outcomes:

- **Name-Resolution Error Rate:** Percentage of `no such host` or connection refused messages in `docker compose logs` for auth calls (target: ~0% after stabilization).
- **Time to Detect Registration Gap:** Time from container restart to first log-detected resolution error and remediation trigger (target: <30s).
- **HTTP 5xx from Auth Calls:** Observed in gateway/client logs during rollout (target: near-zero after discovery stabilizes).
- **Successful Auth Call Rate:** Proportion of calls that reach a healthy `be-authentication-and-roles` instance (target: ~100% once DNS updates propagate).

By tying Service Discovery to Docker log observability, the team gains immediate visibility into lookup issues and can react before authentication outages propagate to end users.

---

## Interoperability

![Interoperability scenario](images/Interoperability-Scenery%20-%20P4.png)

### Cyber-Physical Telemetry Ingestion

**Pattern – Data Ingestion and Validation Pipeline:** Telemetry arrives through `lb-data-ingestion`, is validated/normalized in `be-data-ingestion`, and is queued to `queue-data-ingestion` so downstream processors stay decoupled from field noise, bursts, or schema drift.  
**Tactics – Schema Validation, Backward-Compatibility Handling, Fault Isolation:** Strict JSON schema checks guard the boundary; firmware version headers plus optional fields preserve backward compatibility; malformed frames are isolated (logged/dropped) so healthy traffic continues without blocking threads or queues.

The system must interoperate reliably with a cyber-physical component: a microcontroller continuously streams sensor data to `lb-data-ingestion`, which forwards to `be-data-ingestion` where validation, normalization, and buffering protect the pipeline.

#### 1. Artifact

**Device-to-Ingestion Contract:** REST/HTTP payload contract between the microcontroller and the ingestion layer (`lb-data-ingestion` → `be-data-ingestion`), including JSON schema, version headers, authentication token, and the validation/queuing steps before reaching `queue-data-ingestion`.

#### 2. Source

**Cyber-physical microcontroller device** equipped with environmental sensors (temperature, humidity, soil metrics) sending telemetry frames every few seconds.

#### 3. Stimulus

Continuous telemetry stream (e.g., one sample per second) plus occasional firmware updates introducing new optional fields. Some frames may arrive late, duplicated, or with minor shape differences from older firmware.

#### 4. Environment

Normal field operation with intermittent connectivity, standard network latency, and Dockerized backend on `rootly-network`; ingestion replicas may scale horizontally behind `lb-data-ingestion`.

#### 5. Response

- `lb-data-ingestion` accepts connections and forwards payloads to `be-data-ingestion`.
- `be-data-ingestion` performs schema validation/version checks, normalizes units/field names, rejects malformed frames with clear HTTP errors, and enqueues valid messages to `queue-data-ingestion`.
- Backward compatibility rules keep older firmware payloads accepted; new optional fields are ignored or mapped to defaults while preserving required fields.
- Fault isolation prevents a bad batch (invalid schema, wrong token) from blocking the pipeline: invalid frames are dropped, logged with device metadata, and do not stall healthy traffic.
- Queued delivery decouples `ms-data-processing`, so ingestion remains responsive even if downstream consumers slow down.

#### 6. Response Measure

- **Ingestion Success Rate:** ≥99% of valid telemetry frames accepted during steady streaming.
- **Schema Validation Errors:** <1% per hour for devices on supported firmware; spikes trigger alerting.
- **Queue Lag:** Stable under continuous streaming (<1s added latency from `be-data-ingestion` to `queue-data-ingestion`), even when `ms-data-processing` is throttled.
- **Fault Isolation Effectiveness:** No ingestion outage when a subset of devices sends malformed payloads (measured by continued acceptance of well-formed frames).
- **Error Transparency:** HTTP 4xx/5xx responses and validation failures logged with device identifier and firmware version for rapid troubleshooting when interoperability breaks.
