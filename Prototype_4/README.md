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
    - [Load Balancers](#load-balancers)
    - [Caching](#caching)
  - [Reliability](#reliability)
    - [Service Discovery](#service-discovery)
  - [Interoperability](#performance-and-scalability)
  
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

## Reliability

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
