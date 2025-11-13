# Security Quality Attribute Scenario: Web Application Firewall (WAF) Pattern for Layer 7 DDoS Mitigation

## Table of Contents

1. [Weakness and Security Context](#weakness-and-security-context)
2. [Quality Attribute Scenario](#quality-attribute-scenario)
3. [Security Scenario Analysis](#security-scenario-analysis)
4. [Vulnerability Demonstration (Pre-WAF)](#vulnerability-demonstration-pre-waf)
5. [Countermeasure Implementation](#countermeasure-implementation)
6. [Validation Results (Post-WAF)](#validation-results-post-waf)
7. [Response to Quality Scenario](#response-to-quality-scenario)

---

## Weakness and Security Context

### System Vulnerability Overview

In prototype 3, after implementing the reverse proxy pattern, the reverse proxy is exposed to the internet as the sole entry point. The API gateway and internal microservices are relocated to a private network segment, preventing any direct external access. However, while the reverse proxy provides basic rate-limiting protection, it lacks advanced Web Application Firewall (WAF) capabilities—such as ModSecurity with the OWASP Core Rule Set (CRS)—leaving the system vulnerable to application-layer (Layer 7) attacks, including SQL injection, cross-site scripting (XSS), and denial-of-service (DoS) attacks.

---

### Core Weakness: Lack of Application-Layer Shielding

1. **Reverse proxy without deep inspection**  
   NGINX acts as a reverse proxy but only forwards traffic. It does not perform deep packet inspection or payload analysis.

2. **No centralized rate limiting**  
   The API Gateway or microservices must enforce their own limits. A single malicious client can repeatedly send high-frequency requests or maintain long-lived connections, eventually exhausting gateway threads or CPU.

3. **Full exposure of the entry point**  
   The architecture exposes a single public endpoint —the reverse proxy— as the system’s only ingress.  
   Without additional protection such as a WAF, this design introduces a single point of failure (SPOF).

### Security Implications

| Vulnerability | Description | Security Impact |
|---------------|-------------|-----------------|
| **Gateway resource exhaustion** | “Low & slow” or burst-style Layer-7 DoS targeting `/graphql` or `/auth/login`. | **Availability**: the gateway becomes unresponsive or crashes. |
| **Abuse of expensive requests** | Crafted GraphQL queries with deep recursion or large payloads. | **Integrity**: downstream services are forced to handle costly loads. |
| **Lack of centralized telemetry** | No unified visibility of blocked attempts or request anomalies. | **Confidentiality (derivative)**: difficult to detect scraping or enumeration. |

---

## Quality Attribute Scenario

### Scenario Elements

<img src="../images/WAFPatternScenario.png" />


### 1. Artifact

**Public Entry Point and Traffic Control Plane:**  
Components directly responsible for receiving, routing, and protecting HTTP/HTTPS flows:

- **Reverse Proxy / API Gateway** (`rootly-apigateway` behind the current reverse proxy).  
- **Proposed WAF service** (`rootly-waf`) acting as the pre-gateway inspection layer.  
- **Supporting assets**: WAF rule sets, IP allow/deny lists, centralized metrics and logging pipelines used to trigger automated countermeasures.

### 2. Source

**Malicious External Client** originating from the public internet:

- **Knowledge level**: moderate to advanced—able to mimic legitimate request headers and payloads.  
- **Tooling**: custom scripts or automated clients generating repetitive Layer-7 traffic (e.g., modified Slowloris, curl loops, load-test tools).  
- **Intent**: degrade gateway availability and exhaust backend resources using valid-looking but abusive requests.  
- **Origin**: single external host or a limited set of IP addresses outside the trusted perimeter.

### 3. Stimulus

The source executes a **Layer-7 DoS attack** consisting of:

1. **Discovery and probing** of exposed endpoints (`/graphql`, `/api/v1/auth/login`).  
2. **Continuous bursts** of HTTP requests using alternating GET/POST methods with realistic headers and JSON payloads.  
3. **Sustained connection holding (“low & slow”)** to exhaust worker pools and keep threads occupied.  

### 4. Environment

The system operates under normal conditions, but we contrast two configurations:

#### Pre-Mitigation (Pre-WAF)

- Basic reverse proxy exposes the API Gateway directly.  
- No deep inspection or centralized throttling.  
- Metrics and logs scattered across services, making correlation difficult.  
- Burst handling relies solely on the gateway’s processing capacity.

#### Post-Mitigation (Post-WAF)

- `rootly-waf` positioned in front of the gateway with CRS + custom rules.  
- Dynamic rate limiting per IP, route, and payload size.  
- Structured logging and unified metrics (blocked requests, anomaly scores).  
- Integration with LocalStack WAFv2 for automated testing.

### 5. Expected Response

The system must:

1. **Detect and block or throttle** ≥ 95% of malicious requests.  
2. **Maintain availability**, limiting average latency degradation to < 20%.  
3. **Preserve legitimate user experience**, keeping error rate ≤ 1%.  
4. **Log in real time** the attacking IPs, request patterns, and triggered rules for auditing.

### 6. Response Measure

**Quantitative indicators proving the mitigation’s success:**

- **p99 latency** during the attack compared to baseline.  
- **Legitimate throughput** delivered (req/s).  
- **Block/allow ratio** (number of blocked vs. permitted requests).  
- **Gateway availability** (healthy state, absence of mass 502/503).  
- **RQ (Resilience Quotient)** = Legitimate throughput under attack / Nominal throughput (target ≥ 0.8).  
- **Alert volume** from CRS and custom rules (count of critical activations).

---

## Security Scenario Analysis

### CIA Assessment

| Property | Risk Pre-WAF | Severity |
|----------|--------------|----------|
| **Availability** | Gateway becomes unresponsive under sustained request bursts | Critical |
| **Integrity** | Replayed payloads to exhaust resources | High |
| **Confidentiality** | Large-scale scraping/enumeration attempts | Medium |

### Six Key Security Concepts in the Scenario

| Concept | Definition | Description in Rootly’s Scenario (Pre-WAF) |
|---------|------------|-------------------------------------------|
| **Weakness** | Architectural flaw or inherent susceptibility. | **Unprotected public entry point:** The reverse proxy exposes the API Gateway directly without inspection or rate control. Every request, malicious or not, is passed downstream. |
| **Vulnerability** | Specific condition enabling exploitation. | **Endpoints lacking adaptive throttling:** Routes such as `/graphql` and `/api/v1/auth/login` accept repeated requests without detecting abnormal frequency or connection patterns. |
| **Threat** | Adversary or malicious driver. | **Single malicious client executing a denial-of-service attack:** A persistent source repeatedly targeting the system with crafted requests that consume server resources. |
| **Attack** | Sequence of actions exploiting the vulnerability. | **Layer-7 DoS sequence:** 1) endpoint probing, 2) repetitive bursts or slow connections holding threads open, 3) resource exhaustion and latency spikes. |
| **Risk** | Probability of exploitation × impact. | **Gateway overload and SLA degradation:** High likelihood that sustained request patterns from one attacker will saturate gateway CPU or worker threads, producing 502/503 errors and reducing availability. |
| **Countermeasure** | Architectural/implementation action mitigating the risk. | **Web Application Firewall pattern:** Introduce `rootly-waf` ahead of the gateway with CRS rules, adaptive rate limiting, and payload inspection to detect and block single-source denial attempts before they reach backend services. |

---

## Vulnerability Demonstration (Pre-WAF)

Goal: show that without a WAF the API Gateway collapses under malicious load.

### Prerequisites (attacker host)

```bash
sudo apt update
sudo apt install -y wrk hping3
```

### Phase 1: Baseline

```bash
wrk -t4 -c50 -d30s http://<PUBLIC_HOST>:8080/health
```

**Expected result**: ~1500 req/s, p99 latency < 120 ms, no errors.

### Phase 2: Layer-7 DDoS Attack

1. **Expensive GraphQL burst**

```bash
wrk -t12 -c1200 -d120s -s stress-graphql.lua http://<PUBLIC_HOST>:8080/graphql
```

> `stress-graphql.lua` issues deep recursive queries (`__typename`, excessive introspection).

2. **Low & slow login flood**

```bash
hping3 --flood --url http://<PUBLIC_HOST>:8080/api/v1/auth/login --data 512
```

### Observed behavior (without WAF)

- Latency spikes above 5 seconds.  
- `rootly-apigateway` returns HTTP 502/503 after ~30 seconds.  
- Downstream microservices stay alive, but the gateway crashes (CPU pegged, possible OOM).  
- NGINX logs indicate saturation with no clear source (no global rate limiting).

---

## Countermeasure Implementation *(Pendiente de ejecución)*

- **Patrón propuesto**: Web Application Firewall (`waf-rootly` delante del `rootly-apigateway`).  
- **Táctica**: `Detect Attacks` → `Detect Service Denial`.  
- **Entorno de pruebas**: LocalStack con soporte de WAFv2 (`awslocal wafv2 create-web-acl …`).  

> **Estado actual**: Scripts de aprovisionamiento en preparación. Hay que integrar el contenedor WAF en `docker-compose.yml`, aplicar reglas CRS + personalizadas y automatizar la asociación del WebACL simulado a los endpoints del prototipo.

---

## Analysis of Results: Web Application Firewall (WAF) Scenario

The graphical results illustrate how the implementation of a Web Application Firewall (WAF) effectively mitigates Layer 7 Denial of Service (DoS) attacks by filtering malicious requests before they reach the web application layer.

The bar chart shows the percentage of traffic allowed by the WAF across eight test runs. Each bar represents one run (Test 1 to Test 8), with allowed traffic ranging from 0.23% to 0.80%. The consistently low percentages indicate that the WAF is blocking the vast majority of requests generated during the tests (over 99%), suggesting that most of the simulated traffic was identified as malicious or suspicious and effectively filtered before reaching the application.

<p align="center">
   <img width="700" height="500" alt="waf_allowed_percentage" src="https://github.com/user-attachments/assets/0fbb1ab4-c919-4e17-b5af-c2af66332101" />
</p>

The bar chart shows the percentage of traffic blocked by the WAF across eight test runs (Test 1–Test 8). In all runs, the WAF blocks between 99.20% and 99.77% of the requests, indicating that nearly all simulated traffic is classified as malicious or suspicious and successfully prevented from reaching the application.

<p align="center">
   <img width="700" height="500" alt="waf_blocked_percentage" src="https://github.com/user-attachments/assets/5a1931cd-bb77-4adb-83c3-9ec9eb109eea" />
</p>

The pie chart shows the global distribution of traffic processed by the WAF during the attack scenario. Overall, **99.67%** of requests are **blocked**, while only **0.33%** are **allowed**. This indicates that almost all incoming traffic was classified as malicious or suspicious and successfully filtered, with only a very small fraction treated as legitimate.

<p align="center">
   <img width="650" height="500" alt="waf_traffic_distribution" src="https://github.com/user-attachments/assets/6b46c9f6-41f5-454d-9135-19eeaffd8252" />
</p>

In summary, the WAF implementation enhances the system’s resilience against application-layer DoS attacks by maintaining stability, optimizing resource usage, and preserving service availability under hostile traffic conditions.

---

## Response to Quality Scenario

Following the deployment and testing of the Web Application Firewall (WAF), the traffic analysis confirms that the protection layer is effectively filtering application-layer attacks.

Across all test runs, the WAF blocked between **99.20% and 99.77%** of incoming requests, with a global distribution of **99.67% blocked** versus **0.33% allowed**. This indicates that almost all traffic generated in the attack scenario was classified as malicious or suspicious and stopped before reaching the application gateway.

While these charts focus on traffic distribution rather than end-to-end latency or throughput, the consistently high blocking rate demonstrates that the WAF strongly supports the system’s **availability** and **performance efficiency** under hostile traffic patterns. Future validation with LocalStack will extend this analysis with additional metrics (latency, successful legitimate throughput, and RQ calculations) to confirm consistent behavior across different deployment environments.


---
