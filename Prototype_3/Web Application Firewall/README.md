# Security Quality Attribute Scenario: Web Application Firewall (WAF) Pattern for Layer 7 Dos Mitigation

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

### Phase 2: Layer-7 Dos Attack

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
