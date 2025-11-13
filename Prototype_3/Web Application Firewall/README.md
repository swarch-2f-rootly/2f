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

In prototype 3, after implementing the reverse proxy pattern, the reverse proxy is exposed to the internet as the sole entry point. The API gateway and internal microservices are relocated to a private network segment, preventing any direct external access. However, while the reverse proxy provides basic rate-limiting protection, it lacks advanced Web Application Firewall (WAF) capabilities—such as ModSecurity with the OWASP Core Rule Set (CRS)—leaving the system vulnerable to sophisticated Layer 7 attacks, including SQL injection, cross-site scripting (XSS), and distributed denial-of-service (DDoS) attacks.

iimagencitaaa

### Core Weakness: Lack of Application-Layer Shielding

1. **Reverse proxy without deep inspection**  
   NGINX acts as a reverse proxy but only forwards traffic.It does not provide deep packet inspection or payload analysis capabilities.

2. **No centralized rate limiting**  
   The API Gateway or microservices would need to enforce limits, yet a distributed attack can coordinate thousands of IPs (bots) to bypass per-IP throttling.

3. **Full exposure of the entry point**  
   The architecture exposes a single public endpoint —the reverse proxy— as the system’s only ingress.
   Without additional layers such as a WAF, this design introduces a single point of failure (SPOF).

### Security Implications

| Vulnerability | Description | Security Impact |
|---------------|-------------|-----------------|
| **Gateway resource exhaustion** | “Low & slow” or burst-style layer-7 DDoS targeting `/graphql` or `/auth/login` | **Availability**: the gateway crashes or severely degrades |
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
- **Proposed WAF service** (`rootly-waf`) acting as the pre-gateway inspection layer.  
- **Supporting assets**: WAF rule sets, IP lists (allow/deny), centralized metrics and logging pipelines used to trigger automated countermeasures.

### 2. Source

**Sophisticated External Botnet** (human-driven or automated) originating from the public internet:

- **Knowledge level**: advanced—aware of anti-bot heuristics and ways to bypass basic filters.  
- **Tooling**: distributed attack frameworks (modified Slowloris/LOIC, custom scripts rotating IPs and User-Agent strings).  
- **Intent**: degrade gateway availability and exhaust backend resources using valid-looking but abusive requests.  
- **Origin**: multiple public networks and proxies outside the trusted perimeter.

### 3. Stimulus

The source executes a coordinated layer-7 attack consisting of:

1. **Discovery and fingerprinting** of exposed endpoints (`/graphql`, `/api/v1/auth/login`).  
2. **Generating bursts** of 10,000 requests per minute, alternating GET/POST methods with realistic headers and JSON payloads.  
3. **Rotating IP addresses and User-Agent strings** to evade naive rate limiting.  
4. **Launching “low & slow” waves** that keep connections open to exhaust worker pools.

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
- Integration with LocalStack WAFv2 for automated testing.

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
| **Threat** | Adversary or malicious driver. | **Distributed botnet focused on denial of service:** Coordinated bot networks capable of rotating IPs, headers, and traffic shapes to bypass basic checks, aiming to exhaust resources and trigger downtime. |
| **Attack** | Sequence of actions exploiting the vulnerability. | **Multi-vector layer-7 DDoS:** 1) Endpoint discovery, 2) bursts of tens of thousands of requests per minute with valid payloads, 3) “low & slow” requests to tie up workers, 4) continuous rotation of User-Agent/IP to evade simple blocks. |
| **Risk** | Probability of exploitation × impact. | **Gateway crash and SLA degradation:** High likelihood that the botnet maxes out gateway CPU/threads, leading to 502/503 errors, lost availability for legitimate users, and potential loss of customer trust. |
| **Countermeasure** | Architectural/implementation action mitigating the risk. | **Web Application Firewall pattern:** Introduce `rootly-waf` ahead of the gateway with CRS rules, adaptive rate limiting, temporary blacklists, payload inspection, and centralized telemetry to detect and block denial attempts before they reach backend services. |

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

## Validation Results (Post-WAF) *(Por ejecutar)*

- Objetivo: validar en entorno local con LocalStack que el WAF detecta y bloquea >95% de solicitudes maliciosas y mantiene la disponibilidad.  
- Acciones pendientes:
  1. Capturar métricas de carga base (`wrk`) sin WAF.  
  2. Repetir ensayos tras habilitar `waf-rootly` y documentar latencias, throughput y ratio de bloqueos.  
  3. Registrar auditorías (logs ModSecurity) como evidencia de la táctica `Detect Service Denial`.

grafiquitas

permitidas
<img width="1500" height="750" alt="waf_allowed_percentage" src="https://github.com/user-attachments/assets/0fbb1ab4-c919-4e17-b5af-c2af66332101" />

bloqueadas
<img width="1500" height="750" alt="waf_blocked_percentage" src="https://github.com/user-attachments/assets/5a1931cd-bb77-4adb-83c3-9ec9eb109eea" />

distribucion
<img width="900" height="900" alt="waf_traffic_distribution" src="https://github.com/user-attachments/assets/6b46c9f6-41f5-454d-9135-19eeaffd8252" />

---

## Response to Quality Scenario *(En construcción)*

- Los elementos del escenario, métricas y evidencias se completarán una vez finalizada la validación post-WAF.  
- Se reutilizará la métrica principal (`RQ ≥ 0.8`) y se incluirán capturas de las pruebas con LocalStack cuando estén disponibles.

---

**Estado actual**: documentación preliminar alineada con el patrón WAF y la táctica `Detect Service Denial`. Próximos pasos: concluir implementación, ejecutar pruebas con LocalStack y actualizar las secciones pendientes con resultados cuantitativos.
