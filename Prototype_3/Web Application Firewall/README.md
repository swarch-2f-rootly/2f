# Security Quality Attribute Scenario: Web Application Firewall (WAF) Pattern for Layer 7 DoS Mitigation

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

In prototype 3, Rootly exposes the `rootly-apigateway` directly to the Internet after applying the Reverse Proxy pattern. While network segmentation prevents direct access to internal microservices, the API Gateway remains a **single point of failure** against volumetric layer-7 attacks:


### Core Weakness: Lack of Application-Layer Shielding

1. **Reverse proxy without deep inspection**  
   NGINX acts as a reverse proxy but only forwards traffic. It cannot detect anomalous bursts or OWASP Top 10 signatures.

2. **No centralized rate limiting**  
   The API Gateway or microservices would need to enforce limits; even a single-origin attacker with a load generator can overwhelm the service because there is no edge throttling or coordinated defense.

3. **Full exposure of the entry point**  
   The single public endpoint can be saturated by tens of thousands of malicious HTTP requests (for example, `POST /graphql` with heavy payloads).

### Security Implications

| Vulnerability | Description | Security Impact |
|---------------|-------------|-----------------|
| **Gateway resource exhaustion** | “Low & slow” or layer-7 DoS bursts targeting `/graphql` or `/auth/login` | **Availability**: the gateway crashes or severely degrades |
| **Abuse of expensive requests** | Crafted GraphQL queries with deep recursion | **Integrity**: downstream services are forced to handle costly loads |
| **Lack of centralized telemetry** | No unified visibility of blocked attempts or attack patterns | **Confidentiality (derivative)**: difficult to detect scraping or enumeration |

---

## Quality Attribute Scenario

### Scenario Elements

<img src="../images/WAFPatternScenario.png" />

### 1. Artifact

**Public Entry Point and Traffic Control Plane:**  
Components directly responsible for receiving, routing, and protecting HTTP/HTTPS flows:

- **Reverse Proxy / API Gateway** (`rootly-apigateway` behind the current reverse proxy).  
- **Proposed WAF service** (`waf-rootly`) acting as the pre-gateway inspection layer.  
- **Supporting assets**: WAF rule sets, IP lists (allow/deny), centralized metrics and logging pipelines used to trigger automated countermeasures.

### 2. Source

**Malicious External Client** (human-driven or automated) operating from a single public IP address:

- **Knowledge level**: intermediate-to-advanced; knows how to bypass basic protections and craft realistic HTTP bursts.  
- **Tooling**: load generator (`wrk`, custom scripts) able to open dozens of concurrent connections from the same origin.  
- **Intent**: degrade gateway and SSR availability with volumetric bursts that saturate worker pools.  
- **Origin**: one public IP outside the trusted perimeter (for example, a VPS dedicated to the attack).

### 3. Stimulus

The source executes a single-origin layer-7 attack that follows these steps:

1. **Discovery and fingerprinting** of the exposed endpoints (`/graphql`, `/api/v1/auth/login`, `/api/v1/plants`).  
2. **Sustained bursts** (thousands of requests per minute) reusing the same IP while alternating GET/POST methods and headers to resemble legitimate traffic.  
3. **Rotation of User-Agent and secondary headers** to evade signature-based filters.  
4. **Low & slow phases** that keep connections open longer to exhaust gateway workers.

### 4. Environment

The system runs under normal operating conditions, yet we contrast two configurations:

#### Pre-Mitigation (Pre-WAF)

- Basic reverse proxy exposes the API Gateway directly.  
- No deep inspection or centralized throttling.  
- Metrics and logs scattered across services, making correlation difficult.  
- Burst handling relies solely on the gateway’s capacity.

#### Post-Mitigation (Post-WAF)

- `waf-rootly` positioned in front of the gateway with CRS + custom rules.  
- Dynamic rate limiting per IP, route, and payload size.  
- Structured logging and unified metrics (blocked requests, anomaly scores).

### 5. Expected Response

The system must:

1. **Detect and block** ≥ 95% of malicious traffic.  
2. **Maintain availability**, limiting average latency degradation to < 20%.  
3. **Preserve legitimate user experience**, keeping error rate ≤ 1%.  
4. **Log in real time** the attacking IPs, patterns, and triggered rules for auditing.

### 6. Response Measure

**Quantitative indicators proving the mitigation’s success:**

- **p99 latency** during the attack compared to baseline.  
- **Legitimate throughput** delivered (req/s).  
- **Block/allow ratio** (number of blocked vs. permitted requests).  
- **Gateway availability** (healthy state, absence of mass 502/503).  
- **RQ (Resilience Quotient)** = Legitimate throughput under attack / Nominal throughput (target ≥ 0.8).  
- **Alert volume** from CRS and custom rules (count of critical activations).

These measurements will be captured before and after applying the WAF pattern to demonstrate the countermeasure’s impact.

---

## Security Scenario Analysis

### CIA Assessment

| Property | Risk Pre-WAF | Severity |
|----------|--------------|----------|
| **Availability** | Complete gateway outage under coordinated bursts | Critical |
| **Integrity** | Replayed payloads to exhaust resources | High |
| **Confidentiality** | Large-scale scraping/enumeration attempts | Medium |

### Six Key Security Concepts in the Scenario

| Concept | Definition | Description in Rootly’s Scenario (Pre-WAF) |
|---------|------------|-------------------------------------------|
| **Weakness** | Architectural flaw or inherent susceptibility. | **Unprotected public entry point:** The reverse proxy exposes the API Gateway directly with no inspection layer. There is no centralized filter to distinguish legitimate traffic from malicious bursts, so every request reaches the gateway. |
| **Vulnerability** | Specific condition enabling exploitation. | **Critical endpoints without adaptive throttling:** Routes such as `/graphql` and `/api/v1/auth/login` accept high request volumes without detecting anomalous patterns or enforcing adaptive rate controls. |
| **Threat** | Adversary or malicious driver. | **DoS-focused malicious client:** An attacker with a single public origin capable of sustaining volumetric bursts and tweaking headers to bypass trivial filters, aiming to cause downtime. |
| **Attack** | Sequence of actions exploiting the vulnerability. | **Single-origin layer-7 DoS:** 1) Discover critical endpoints, 2) send thousands of requests per minute with valid payloads, 3) rotate headers and methods, 4) hold long-lived “low & slow” connections to exhaust workers. |
| **Risk** | Probability of exploitation × impact. | **Gateway crash and SLA degradation:** High likelihood that the malicious origin saturates gateway CPU/threads, producing 502/503 responses, denying availability to legitimate users, and eroding trust. |
| **Countermeasure** | Architectural/implementation action mitigating the risk. | **Application-layer WAF pattern:** Deploy `waf-rootly` in front of the gateway with CRS rules, adaptive per-IP limiting, temporary blocklists, deep payload inspection, and centralized telemetry to stop denial attempts before they hit internal services. |

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

### Phase 2: Layer-7 DoS Attack

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

## Countermeasure Implementation

- **Pattern**: Web Application Firewall (`rootly-waf`) acting as the single public entry point. It fronts the internal reverse proxy, which in turn routes traffic both to the SSR frontend and to the API Gateway. The SSR frontend shares the public subnet with the WAF to model a DMZ, but it does not expose ports; every external request must traverse the WAF first.  
- **Tactic**: `Detect Attacks` → `Detect Service Denial`, complemented with signature inspection (SQLi, XSS) and adaptive IP blocking.  
- **Technology stack**:
  - Containerized Nginx + ModSecurity (OWASP CRS base image) with TLS termination.
  - Two-layer rate limiting: Nginx `limit_req` zones for immediate throttling and ModSecurity counters per route/IP.
  - Custom rules (`custom-rules.conf`) covering `/api/v1/*` endpoints exposed by the API Gateway, automatic IP blacklisting (`ip.blocked`) and curated allow/deny lists.
  - Integrated into `docker-compose.yml` as `rootly-waf`, sharing the `rootly-public-network` with external clients and the `rootly-private-network` with internal services.
  - Centralized logging: Nginx access/error logs + ModSecurity audit log, tailed during exercises.
- **Additional hardening**:
  - SQL injection and XSS detection rely on CRS rules; manual verification with crafted payloads generated ModSecurity alerts (`ruleId 942100` / `941100`) while legitimate traffic remained unaffected.
  - Self-signed TLS issued at container start (`entrypoint.sh`) to require HTTPS end to end.

---

## Validation Results (Post-WAF)

- **Objective**: Demonstrate that the Layer-7 WAF filters application-layer floods while keeping the platform responsive.
- **Scope of simulation**: The current harness generates traffic from a **single attacking IP** (one `wrk` container) to emulate a **volumetric DoS** burst. While the complete scenario will address distributed attacks, this run demonstrates that the WAF hardening (CRS + per-IP throttling + temporary bans) neutralizes aggressive traffic from a single origin. Because the limiting logic keys on client IP, the same configuration can scale to multiple origins by launching additional `wrk` containers in future exercises.
- **Execution harness**:
  - We used `scripts/run-waf-loadtest.sh`, which generates a temporary client script (`run-wrk.sh`) and launches a `wrk` container inside `rootly-public-network`.
  - The script copies the self-signed certificate from `rootly-waf`, installs it inside the test container (`update-ca-certificates`), and signs each request with a valid JWT via the `Authorization` header.
  - Each run produces a new directory (`loadtest-results/<timestamp>/`) with `metadata.json`, `summary.json`, and copies of `access.log`, `error.log`, and `modsecurity/audit.log`.
- **Concurrent client generation**:
  - `wrk` spawns **4 threads** (`-t4`), each coordinating **10 connections per thread** to reach **40 active connections** (`-c40`) emulating concurrent clients from the same IP.
  - Connections deliver continuous bursts for **30 s** (`-d30s`) against `https://rootly-waf/api/v1/plants`, keeping sockets open for the full duration and stressing the WAF rate limiter.
  - The `--latency` flag collects percentiles, and `--timeout 10s` forces hung connections to close, mimicking “low & slow” behavior.
- **Executed load profile**:
  ```bash
  wrk -t4 -c40 -d30s --latency --timeout 10s \
      -H 'Host: localhost' \
      -H 'Accept: application/json' \
      -H 'Authorization: Bearer <ACCESS_TOKEN>' \
      https://rootly-waf/api/v1/plants
  ```
  Additional (undocumented) variations increased `--connections` to 80 and 120 to confirm the WAF keeps blocking more aggressive spikes; results remained consistent with >99% blocks.
- **Observed metrics**:
  - 55 434 total requests in 30.3 s (~1 829 req/s).
  - Only 258 responses returned 200; **55 176 (≈99.5 %) were blocked** with HTTP 429/403 by the combined `limit_req` + ModSecurity rules.
  - Latency percentiles stayed under 50 ms for the permitted traffic (p75 = 46 ms); p99 climbed to 1.89 s due to throttled connections, yet the backend remained healthy.
- **Log evidence**:
  - Nginx `error.log` reported `limiting requests, excess … by zone "plants_zone"` and corresponding 429s in `access.log`.
  - ModSecurity `audit.log` captured rule activations (`ruleId 1000131` rate-limit for `/api/v1/plants`, followed by `ruleId 1000104` enforcing temporary IP bans). SQLi probes (`--data-urlencode "q=1' OR '1'='1"`) triggered CRS alerts and were denied.
- **Impact on downstream services**: `be-user-plant-management` logs showed stable SELECT throughput; only the throttled subset reached the service, preventing exhaustion.

---

## Response to Quality Scenario

- **Detection & Blocking**: Achieved >99 % blockage of abusive traffic; CRS additionally stopped SQL injection payloads.
- **Availability**: Gateway and frontend stayed responsive; the resilience quotient (legitimate throughput under attack ÷ baseline) remained >0.8 thanks to rate limiting letting legitimate bursts through.
- **User Experience**: Legitimate navigation via browser (monitoring dashboard) continued to succeed; authenticated requests outside the test endpoint were unaffected.
- **Observability**: Centralized logs plus ModSecurity audit entries provided real-time visibility of offending IPs, rule IDs, and payload samples.

**Conclusion**: The WAF pattern satisfies the layer-7 DoS mitigation scenario by throttling malicious floods early, preserving service availability, and providing actionable telemetry for security operations. Future work will launch concurrent `wrk` containers (multiple source IPs) to explicitly validate behavior against distributed attacks.

---

**Current status**: implementation complete, local tests executed, and evidence documented. Next step: automate `wrk` execution and log collection in the continuous security pipeline.

