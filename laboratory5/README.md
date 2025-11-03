# Rootly
## Laboratory 5: Network Segmentation Pattern Implementation

### Organization Link:
[https://github.com/swarch-2f-rootly](https://github.com/swarch-2f-rootly)

## Team 2F
- Carlos Santiago Sandoval Casallas
- Cristian Santiago Tovar Bejarano
- Danny Marcelo Yaluzan Acosta
- Esteban Rodriguez Muñoz
- Santiago Restrepo Rojas
- Gabriela Guzmán Rivera
- Gabriela Gallegos Rubio
- Andrés Camilo Orduz Lunar

---

## 1. Objective and Architectural Pattern Selection

The objective of this laboratory is to select, analyze, and implement a set of architectural patterns oriented toward system security.

In this case, we will implement the **Network Segmentation Pattern**, which was assigned to our team. This pattern implements the **Limit Access** security tactic (part of the **Resist Attacks** tactics) by dividing the system's infrastructure into isolated network zones. The purpose is to minimize the attack surface and restrict lateral movement of threats within the system.

---

## 2. Technical Guide and Foundational Concepts

To successfully implement the Network Segmentation Pattern using Docker, it is essential to understand the networking concepts and underlying technologies that will make this architecture possible.

### What is a Network?
A **network** is a system of interconnected computing devices (nodes) capable of exchanging data and sharing resources through defined communication protocols (such as TCP/IP). Networks are logically divided into **subnets**, which are identifiable segments defined by a specific range of IP addresses. In a secure architecture, networks function not only as communication paths but also as **security perimeters**. Proper design determines which segments should be exposed (public) and which must remain isolated (private), thereby establishing layers of defense in depth.

### What is Docker?
**Docker** is a platform that uses operating-system-level virtualization to deliver software in packages called **containers**. Containers are isolated and highly efficient environments that package an application along with all its dependencies (code, runtime, system tools, libraries). Docker provides critical isolation for processes and resources, including networking, making it an ideal tool for implementing granular architectural security patterns such as network segmentation. This isolation capability allows the creation of well-defined security boundaries between different system components.

### What Types of Networks Exist in Docker?
Docker offers several network drivers, but for single-host, multi-service deployments, the **User-Defined Bridge Network** is the recommended option for implementing segmentation.

* **User-Defined Bridge Network:** When a network is explicitly created using Docker Compose (or `docker network create`), a user-defined bridge network is obtained. These networks provide stronger isolation than the default bridge and, fundamentally, enable **automatic DNS resolution**. This means containers can communicate using their **service names** (for example, `api_gateway`) instead of relying on ephemeral IP addresses that may change. This functionality is key to our segmentation strategy, as it allows defining access policies based on logical names rather than volatile IP addresses.

---

## 3. Security Scenario Analysis: CIA Triad

The implementation of the Network Segmentation Pattern is a direct response to a high-risk scenario that threatens the **Confidentiality** and **Integrity** of the application's data.

### CIA Triad Affected
This scenario addresses security through direct mitigation of risks that could compromise the system's **Confidentiality** and **Integrity**.
* **Confidentiality:** There is a risk that sensitive data (user records, plant information, sensor data) could be exposed if an attacker gains direct access to the database or internal APIs without passing through the frontend's authentication and authorization controls.
* **Integrity:** There is a risk of unauthorized modification, deletion, or corruption of data if an attacker can bypass the application's business logic and invoke backend services directly, thereby altering the correct state of the system.

### Six Key Security Concepts in the Scenario

| Concept | Definition | Description in Rootly's Scenario (Pre-Segmentation) |
| :--- | :--- | :--- |
| **Weakness** | A design flaw or inherent system susceptibility that can be exploited. | **Flat Network Architecture:** All services (Frontend, API Gateway, Database) are deployed on a single shared network segment. This lack of logical separation facilitates unauthorized discovery and lateral movement within the network, allowing an attacker who compromises one entry point to access other critical services. |
| **Vulnerability** | The specific path or condition that allows a threat to materialize or exploit a weakness. | **Direct Backend Port Exposure:** Since all services are on the same host, an attacker who discovers the host's public IP can perform a port scan and potentially discover unauthenticated ports of sensitive backend services (e.g., API Gateway on port 8080) that were not designed for direct public consumption. |
| **Threat** | The agent or motivation that executes the attack. | **External Malicious Actor/Automated Bot:** An individual or script originating from the public internet, actively probing the host's public IP to find accessible services and exploit vulnerabilities. This type of threat seeks breaches in the security perimeter. |
| **Attack** | The sequence of actions performed by the threat to exploit the vulnerability. | **Network Reconnaissance and Direct Service Access:** The attacker executes an **Nmap scan** on the host's public IP to list all open ports. Subsequently, they attempt a direct connection or unauthorized API call (e.g., using `curl`) to a backend port, completely bypassing the Frontend's security checks. |
| **Risk** | The potential for loss or damage if the threat successfully exploits the vulnerability. | **Data Leakage and Manipulation:** The primary risk is a high-impact **Data Breach**, resulting in loss of **Confidentiality** (data exposed) and **Integrity** (data modified or corrupted), which can lead to operational shutdown of the system and cause severe reputational damage to the plant monitoring platform. |
| **Countermeasure** | The architectural or implementation action taken to mitigate the risk. | **Network Segmentation Pattern:** Implementation of a **`rootly-public-network`** and a **`rootly-private-network`**. By removing the API Gateway from any network accessible via host ports, the direct access vulnerability is eliminated, as the attacker can only reach the Frontend, which acts as a controlled single point of entry. |

---

## 4. Simulated Attack Guide (Linux/Bash Environment)

This guide illustrates the attack process in a **Pre-Segmentation** environment and how the **Post-Segmentation** countermeasure defeats it. We assume the host IP is `192.168.1.10` and that the frontend service is running on port 3001. To allow users to interact with our frontend, we must expose it with a public IP (in this case `192.168.1.10:3001`). This is a necessary step; however, malicious users could leverage their networking knowledge to scan our network and attempt unauthorized access.

### Prerequisites

Before starting the simulation, ensure the following tools are installed on your Linux (Arch Linux for this case) system:

```bash
# Update system packages
sudo pacman -Syu

# Install nmap for network scanning
sudo pacman -S --noconfirm nmap

# Install curl for HTTP requests (usually pre-installed on Arch)
sudo pacman -S --noconfirm curl

# Verify Docker and Docker Compose are installed
docker --version
docker compose version
```

### Phase 1: Preparation and Discovery (Attacker Steps)

#### Step 1.1: Obtain Host IP Address

The attacker needs to identify the target host's IP address. In a real scenario, this could be the public IP of the server. For local testing, we identify the host machine's local network IP.

```bash
# Command to find host IP address
ip addr show | grep -E 'inet .* scope global'
```

**Example Output:**
```
    inet 192.168.1.10/24 brd 192.168.1.255 scope global dynamic noprefixroute wlp2s0
```

From this output, we extract the host IP: **192.168.1.10**

#### Step 1.2: Network Reconnaissance (Nmap Port Scan)

The attacker performs a port scan to discover exposed services on the target host. For efficiency, we scan specific ports used by Rootly services.

```bash
# Quick scan of specific ports used by Rootly services
sudo nmap -p 3001,5432,8000-8003,8005,8080,8082 192.168.1.10
```

**Pre-Segmentation Result (Vulnerable System):**

```
Starting Nmap 7.98 ( https://nmap.org ) at 2025-11-02 15:58 -0500
Nmap scan report for 192.168.1.10
Host is up.

PORT     STATE    SERVICE
3001/tcp filtered nessus
5432/tcp filtered postgresql
8000/tcp filtered http-alt
8001/tcp filtered vcom-tunnel
8002/tcp filtered teradataordbms
8003/tcp filtered mcreport
8005/tcp filtered mxi
8080/tcp filtered http-proxy
8082/tcp filtered blackice-alerts

Nmap done: 1 IP address (1 host up) scanned in 3.60 seconds
```

**Critical Understanding:** The scan shows all ports as "filtered", which typically indicates firewall protection. However, this is **misleading** - Docker configures iptables in a way that makes ports appear filtered to nmap while still allowing actual connections. The "filtered" state does NOT guarantee security.

**Verification - The Real Vulnerability:**

Let's verify if these ports are actually accessible despite showing as "filtered":

```bash
# Test API Gateway accessibility (Port 8080)
curl -v http://192.168.1.10:8080/api/v1/health --max-time 5
```

**Result:**
```
*   Trying 192.168.1.10:8080...
* Connected to 192.168.1.10 (192.168.1.10) port 8080
> GET /api/v1/health HTTP/1.1
> Host: 192.168.1.10:8080
...
< HTTP/1.1 200 OK
{"status":"healthy","service":"rootly-apigateway"}
* Connection #0 to host 192.168.1.10 left intact
```

**Analysis:** Despite nmap reporting ports as "filtered", the services ARE accessible! This demonstrates:
- Port 3001: Frontend - **Intended public access** ✓
- Port 8080: API Gateway - **SECURITY VULNERABILITY** - Accessible without authentication
- Port 8000-8003, 8005: Backend services - **CRITICAL VULNERABILITIES** - Should be internal only
- Port 5432: PostgreSQL Database - **CRITICAL VULNERABILITY** - Should never be exposed
- Port 8082: Kafka UI - **SECURITY VULNERABILITY** - Administrative interface exposed

The Docker firewall creates a false sense of security. All mapped ports are actually accessible.

**Post-Segmentation Result (Secured System):**

After implementing network segmentation (removing port mappings from backend services in docker-compose.yml), the same scan would show:

```
Starting Nmap 7.98 ( https://nmap.org ) at 2025-11-02 16:05 -0500
Nmap scan report for 192.168.1.10
Host is up.

PORT     STATE    SERVICE
3001/tcp filtered nessus
5432/tcp filtered postgresql
8000/tcp filtered http-alt
8001/tcp filtered vcom-tunnel
8002/tcp filtered teradataordbms
8003/tcp filtered mcreport
8005/tcp filtered mxi
8080/tcp filtered http-proxy
8082/tcp filtered blackice-alerts

Nmap done: 1 IP address (1 host up) scanned in 3.55 seconds
```

### Phase 2: The Attack - Direct Service Access Attempt

The attacker attempts to exploit the discovered backend services by bypassing the frontend authentication layer. 

#### Scenario A: PRE-SEGMENTATION (Attack Succeeds)

In the vulnerable configuration, the attacker can directly access backend services. Let's demonstrate attacks on different services:

**Attack 1: API Gateway Direct Access**

```bash
# Attempt to access API Gateway directly (bypassing frontend)
curl -v http://192.168.1.10:8080/api/v1/health --max-time 5
```

**Output:**
```
*   Trying 192.168.1.10:8080...
* Connected to 192.168.1.10 (192.168.1.10) port 8080
> GET /api/v1/health HTTP/1.1
> Host: 192.168.1.10:8080
> User-Agent: curl/8.9.1
> Accept: */*
> 
< HTTP/1.1 404 Not Found
< Content-Type: application/json; charset=utf-8
< Date: Sun, 02 Nov 2025 20:57:09 GMT
< Content-Length: 27
< 
{"error":"Route not found"}
* Connection #0 to host 192.168.1.10 left intact
```

**Attack 2: Backend Service Direct Access**

```bash
# Attempt to access backend analytics service directly
curl -v http://192.168.1.10:8000/health --max-time 5
```

**Attack 3: PostgreSQL Database Exposure**

```bash
# Attempt to connect to exposed PostgreSQL database
psql -h 192.168.1.10 -p 5432 -U postgres
```

**Result:** All connections succeed! The attacker has:
- **Direct API Gateway access** - Can potentially exploit unauthenticated endpoints
- **Direct backend service access** - Can bypass business logic and access controls
- **Database exposure** - Could attempt password brute-force or exploit vulnerabilities

This represents a **complete breach of the security perimeter**.

---

## 5. Implementing the Network Segmentation Pattern

After identifying the vulnerabilities in the pre-segmentation architecture, we implement the **Network Segmentation Pattern** to isolate backend services and minimize the attack surface.

### 5.1. Core Principle: Separation of Concerns

The Network Segmentation Pattern is a defense-in-depth strategy that reduces the **blast radius** of an attack by dividing the infrastructure into isolated network zones:

1. **Public Zone (rootly-public-network):** Contains only components that absolutely require external access
2. **Private Zone (rootly-private-network):** Contains all sensitive backend services, databases, and internal infrastructure

By enforcing this separation, even if an attacker compromises a public-facing component, they cannot directly access backend services, as those services exist on an entirely separate, non-routable network segment.

### 5.2. Implementation Strategy

The implementation involves three key changes to the Docker Compose configuration:

#### Change 1: Define Two Isolated Networks

We create two user-defined bridge networks with different security properties:

```yaml
networks:
  rootly-public-network:
    driver: bridge
    name: rootly-public-network
    # Standard bridge network - allows external connectivity
  
  rootly-private-network:
    driver: bridge
    name: rootly-private-network
    # Critical: 'internal: true' prevents ANY external connectivity
    internal: true
```

**Key Security Feature:** The `internal: true` flag on the private network ensures maximum isolation by preventing traffic from routing outside the Docker host, even for outgoing connections. This creates a truly isolated internal network.

#### Change 2: Remove Port Mappings from Backend Services

In the vulnerable configuration, all services exposed ports to the host:

**PRE-SEGMENTATION (Vulnerable):**
```yaml
# API Gateway - EXPOSED to host network
api-gateway:
  ports:
    - "8080:8080"  # VULNERABLE: Port mapped to host
  networks:
    - rootly-network

# Database - EXPOSED to host network
db-authentication-and-roles:
  ports:
    - "5432:5432"  # CRITICAL VULNERABILITY: Database exposed
  networks:
    - rootly-network

# Backend Service - EXPOSED to host network
be-analytics:
  ports:
    - "8000:8000"  # VULNERABLE: Backend directly accessible
  networks:
    - rootly-network
```

**POST-SEGMENTATION (Secured):**
```yaml
# API Gateway - NO port mapping, isolated in private network only
api-gateway:
  # ports: removed!  # SECURE: No host port exposure
  networks:
    - rootly-private-network  # Can communicate with backends only

# Database - NO port mapping, isolated in private network
db-authentication-and-roles:
  # ports: removed!  # SECURE: Database not accessible from host
  networks:
    - rootly-private-network  # Only internal communication

# Backend Service - NO port mapping, isolated in private network
be-analytics:
  # ports: removed!  # SECURE: Backend not accessible from host
  networks:
    - rootly-private-network  # Only internal communication
```

#### Change 3: Configure Frontend as the Gateway

The frontend is the **only** service that bridges both networks and exposes a port to the host:

```yaml
frontend-ssr:
  ports:
    - "3001:3001"  # ONLY public-facing port
  networks:
    - rootly-public-network   # Accessible from host via port mapping
    - rootly-private-network  # Can communicate with backend services
  environment:
    - API_GATEWAY_URL=http://api-gateway:8080  # Uses internal DNS
```

**Critical Understanding:** The frontend receives **two IP addresses**:
- **Public Network IP** (example: `172.19.0.2`): Used for external connections from the host
- **Private Network IP** (example: `172.20.0.2`): Used for internal communication with backend services

Only the public network IP is exposed via port mapping. The private network IP remains internal, enabling secure backend communication without external exposure.

### 5.3. Complete Configuration Comparison

#### Before: Single Flat Network (Vulnerable)

```yaml
services:
  frontend-ssr:
    ports:
      - "3001:3001"
    networks:
      - rootly-network  # Single shared network
  
  api-gateway:
    ports:
      - "8080:8080"  # Exposed to host
    networks:
      - rootly-network  # Same network as everything else
  
  be-authentication-and-roles:
    ports:
      - "8001:8000"  # Exposed to host
    networks:
      - rootly-network
  
  db-authentication-and-roles:
    ports:
      - "5432:5432"  # Database exposed to host
    networks:
      - rootly-network

networks:
  rootly-network:
    driver: bridge
```

**Problem:** All services share the same network and have ports mapped to the host, making them directly accessible from outside.

#### After: Segmented Networks (Secured)

```yaml
services:
  frontend-ssr:
    ports:
      - "3001:3001"  # Only public-facing service
    networks:
      - rootly-public-network   # For external access
      - rootly-private-network  # For backend communication
  
  api-gateway:
    # NO ports mapping!
    networks:
      - rootly-private-network  # Only on private network
  
  be-authentication-and-roles:
    # NO ports mapping!
    networks:
      - rootly-private-network  # Isolated
  
  be-user-plant-management:
    # NO ports mapping!
    networks:
      - rootly-private-network  # Isolated
  
  be-analytics:
    # NO ports mapping!
    networks:
      - rootly-private-network  # Isolated
  
  db-authentication-and-roles:
    # NO ports mapping!
    networks:
      - rootly-private-network  # Database fully isolated

networks:
  rootly-public-network:
    driver: bridge
    name: rootly-public-network
  
  rootly-private-network:
    driver: bridge
    name: rootly-private-network
    internal: true  # Prevents external connectivity
```

**Solution:** 
- Backend services have **no port mappings** and exist only on the private network
- API Gateway is **only** on the private network, with no host port mapping
- Frontend is the **only** service in both networks and the **only** service with external access
- Private network is marked as `internal: true` for maximum isolation

### 5.4. Network Communication Flow

The following diagrams illustrate how traffic flows through the segmented architecture. Note that IP addresses shown are examples for illustration purposes.

**External User → Frontend:**
```
External User (example: 192.168.1.10:3001) 
    → Host Network Interface
    → Docker Port Mapping (3001:3001)
    → Frontend Container (rootly-public-network IP example: 172.19.0.2)
```

**Frontend → API Gateway (Internal):**
```
Frontend Container (rootly-private-network IP example: 172.20.0.2)
    → Docker Internal DNS (api-gateway)
    → API Gateway Container (rootly-private-network IP example: 172.20.0.3)
```

**API Gateway → Backend Services:**
```
API Gateway (rootly-private-network IP example: 172.20.0.3)
    → Docker Internal DNS (be-authentication-and-roles)
    → Backend Service (rootly-private-network IP example: 172.20.0.4)
```

**External Attacker → API Gateway (BLOCKED):**
```
External Attacker (example: 192.168.1.10:8080)
    → Host Network Interface
    → NO PORT MAPPING EXISTS
    → Connection Refused
```

### 5.5. Security Benefits Achieved

| Aspect | Before Segmentation | After Segmentation |
|--------|--------------------|--------------------|
| **Attack Surface** | All services exposed (8+ ports) | Only frontend exposed (1 port) |
| **Direct Backend Access** | Possible via port 8080, 8000, etc. | Impossible - no port mappings |
| **Database Exposure** | PostgreSQL accessible on port 5432 | Completely isolated, no external access |
| **Lateral Movement** | Easy - all services on same network | Restricted - services isolated by network |
| **Blast Radius** | Compromise of any service affects all | Compromise limited to network segment |
| **Defense in Depth** | Single layer (application auth) | Multiple layers (network + application) |

---

## 6. Post-Segmentation Attack Results

After implementing network segmentation by removing port mappings and isolating services on the private network, we verify that the previous attack vectors are now completely blocked. The attacker has **no way** to access backend services.

### 6.1. Network Reconnaissance - Post Segmentation

The attacker repeats the same reconnaissance scan to identify exposed services:

```bash
# Attacker performs the same nmap scan after segmentation
sudo nmap -p 3001,5432,8000-8003,8005,8080,8082 192.168.1.10
```

**Post-Segmentation Scan Result:**
```
Starting Nmap 7.98 ( https://nmap.org ) at 2025-11-03 16:15 -0500
Nmap scan report for 192.168.1.10
Host is up.

PORT     STATE    SERVICE
3001/tcp filtered nessus
5432/tcp filtered postgresql
8000/tcp filtered http-alt
8001/tcp filtered vcom-tunnel
8002/tcp filtered teradataordbms
8003/tcp filtered mcreport
8005/tcp filtered mxi
8080/tcp filtered http-proxy
8082/tcp filtered blackice-alerts

Nmap done: 1 IP address (1 host up) scanned in 3.55 seconds
```

**Critical Observation:** The nmap output looks identical to the pre-segmentation scan (all ports show as "filtered"). However, the crucial difference is that now these ports are **genuinely inaccessible**, not just filtered by Docker's iptables. The port mappings have been completely removed.

### 6.2. Attack Attempts - All Vectors Blocked

The attacker now attempts to exploit the previously vulnerable services. **Every single attack fails.**

#### Attack 1: API Gateway Direct Access (BLOCKED)

```bash
# Attacker attempts to access API Gateway health endpoint
curl -v http://192.168.1.10:8080/health --max-time 5
```

**Output:**
```
*   Trying 192.168.1.10:8080...
* connect to 192.168.1.10 port 8080 from 192.168.1.10 port 47810 failed: Conexión rehusada
* Failed to connect to 192.168.1.10 port 8080 after 0 ms: Could not connect to server
* closing connection #0
curl: (7) Failed to connect to 192.168.1.10 port 8080 after 0 ms: Could not connect to server
```

**Result: BLOCKED** - No process is listening on the host's port 8080. The API Gateway exists only on the private network.

#### Attack 2: Backend Analytics Service (BLOCKED)

```bash
# Attacker attempts to access backend analytics service
curl -v http://192.168.1.10:8000/health --max-time 5
```

**Output:**
```
*   Trying 192.168.1.10:8000...
* connect to 192.168.1.10 port 8000 from 192.168.1.10 port 50468 failed: Conexión rehusada
* Failed to connect to 192.168.1.10 port 8000 after 0 ms: Could not connect to server
* closing connection #0
curl: (7) Failed to connect to 192.168.1.10 port 8000 after 0 ms: Could not connect to server
```

**Result: BLOCKED** - The analytics backend is completely isolated on the private network.

#### Attack 3: Authentication Backend (BLOCKED)

```bash
# Attacker attempts to access authentication service
curl -v http://192.168.1.10:8001/health --max-time 5
```

**Output:**
```
*   Trying 192.168.1.10:8001...
* connect to 192.168.1.10 port 8001 from 192.168.1.10 port 46962 failed: Conexión rehusada
* Failed to connect to 192.168.1.10 port 8001 after 0 ms: Could not connect to server
* closing connection #0
curl: (7) Failed to connect to 192.168.1.10 port 8001 after 0 ms: Could not connect to server
```

**Result: BLOCKED** - The authentication service cannot be reached from outside.

#### Attack 4: PostgreSQL Database Direct Access (BLOCKED)

```bash
# Attacker attempts to connect to PostgreSQL database
psql -h 192.168.1.10 -p 5432 -U postgres --command="\l" 2>&1 | head -3
```

**Output:**
```
psql: error: connection to server at "192.168.1.10", port 5432 failed: Connection refused
	Is the server running on that host and accepting TCP/IP connections?
```

**Result: BLOCKED** - The database is completely inaccessible from the external network.

#### Attack 5: Kafka UI Admin Interface (BLOCKED)

```bash
# Attacker attempts to access Kafka UI administrative interface
curl -v http://192.168.1.10:8082/ --max-time 5
```

**Output:**
```
*   Trying 192.168.1.10:8082...
* connect to 192.168.1.10 port 8082 from 192.168.1.10 port 33988 failed: Conexión rehusada
* Failed to connect to 192.168.1.10 port 8082 after 0 ms: Could not connect to server
* closing connection #0
curl: (7) Failed to connect to 192.168.1.10 port 8082 after 0 ms: Could not connect to server
```

**Result: BLOCKED** - Administrative interfaces are no longer exposed.

### 6.3. Why the Attacker Cannot Access the Services

The network segmentation pattern eliminates attack vectors through three key mechanisms:

1. **No Port Mappings on Backend Services:**
   - Before: `ports: - "8080:8080"` exposed the API Gateway to the host
   - After: The `ports` section is removed entirely
   - Result: There is **no listening socket** on the host's network interface for port 8080

2. **Private Network Isolation:**
   - All backend services exist only on `rootly-private-network`
   - This network is marked as `internal: true`
   - Docker **does not route** traffic from the host network to internal networks
   - Result: Even if the attacker knew the internal IP, they cannot reach it

3. **Single Point of Entry:**
   - Only the frontend has a port mapping: `3001:3001`
   - The frontend acts as a controlled gateway
   - All backend requests must go through the frontend's authentication/authorization logic
   - Result: Attackers **must** authenticate through the frontend to access any backend functionality

### 6.4. Attack Surface Comparison

| Attack Vector | Pre-Segmentation | Post-Segmentation |
|---------------|------------------|-------------------|
| Direct API Gateway access (port 8080) | **VULNERABLE** - Connection succeeds | **BLOCKED** - Connection refused |
| Direct backend service access (ports 8000-8003) | **VULNERABLE** - Connection succeeds | **BLOCKED** - Connection refused |
| Direct database access (port 5432) | **CRITICAL VULNERABILITY** - Connection possible | **BLOCKED** - Connection refused |
| Administrative interfaces (port 8082) | **VULNERABLE** - Web UI accessible | **BLOCKED** - Connection refused |
| Number of exposed attack surfaces | **8+ ports exposed** | **1 port exposed (frontend only)** |
| Bypass authentication possible? | **YES** - Direct backend access | **NO** - Must go through frontend |

**Conclusion:** The attacker has **absolutely no way** to directly access any backend service, database, or administrative interface. The attack surface has been reduced from 8+ vulnerable entry points to a single, controlled entry point (the frontend) where proper authentication and authorization are enforced.

### Phase 3: Verification of Internal Communication (Countermeasure success without affecting internal opperations)

This phase verifies that legitimate communication between frontend and backend services still functions correctly over the private network, proving that segmentation enhances security without breaking functionality.

#### Step 3.1: Identify Running Containers

```bash
# List all running containers
docker ps --format "table {{.ID}}\t{{.Names}}\t{{.Ports}}"
```

**Example Output:**
```
CONTAINER ID   NAMES                              PORTS
7a3f9b2c1e5d   rootly-ssr-frontend                0.0.0.0:3001->3000/tcp
4d8e2a1f9c3b   rootly-apigateway                  
9c5f7e3a2b1d   rootly-user-plant-backend          
1f4c8b6e3a7d   rootly-sensor-data-backend         
2e9d5f1c4a8b   rootly-postgres                    
```

Notice that only the frontend has a port mapping to the host (`0.0.0.0:3001->3000/tcp`), while other services have no exposed ports.

#### Step 3.2: Test Internal Communication Using Service Names

Since the frontend container already has wget available, we can test connectivity directly from the host:

```bash
# Test connection to API Gateway health endpoint using Docker DNS resolution
docker exec frontend-ssr wget -O- http://api-gateway:8080/health --timeout=5
```

Alternatively, if we want to access the container shell interactively:

```bash
# Execute an interactive shell in the frontend container (requires root for package installation)
docker exec -it -u root frontend-ssr sh

# Once inside, if curl is needed, install it
apk add --no-cache curl

# Test connection to API Gateway health endpoint
curl http://api-gateway:8080/health
```

**Example Output (Successful Internal Communication):**

```
Connecting to api-gateway:8080 (172.20.0.2:8080)
writing to stdout
-                    100% |********************************|   391  0:00:00 ETA
written to stdout
{
  "services": {
    "analytics": {"status": "unknown", "url": "http://be-analytics:8000"},
    "auth": {"status": "unknown", "url": "http://be-authentication-and-roles:8000"},
    "data_management": {"status": "unknown", "url": "http://be-data-processing:8000"},
    "plant_management": {"status": "unknown", "url": "http://be-user-plant-management:8000"}
  },
  "status": "healthy",
  "timestamp": "2025-11-03T00:05:19Z",
  "version": "1.0.0"
}
```

The connection succeeds and the API Gateway returns its health status along with information about all configured backend services.

**Analysis:** 
- The connection succeeds using the service name `api-gateway`
- Docker's internal DNS resolves this to a private IP (example: `172.20.0.3`)
- Communication occurs entirely within the `rootly-private-network`
- The API Gateway is accessible to authorized services (frontend) but invisible to external attackers

#### Step 3.3: Verify Other Backend Services Connectivity

```bash
# Test connectivity to analytics backend service
docker exec frontend-ssr wget -O- http://be-analytics:8000/health --timeout=5
```

**Example Output:**
```
Connecting to be-analytics:8000 (172.19.0.14:8000)
writing to stdout
-                    100% |********************************|   152  0:00:00 ETA
written to stdout
{"status":"healthy","service":"analytics","influxdb":"healthy"}
```

```bash
# Test connectivity to authentication backend service
docker exec frontend-ssr wget -O- http://be-authentication-and-roles:8000/health --timeout=5
```

**Example Output:**
```
Connecting to be-authentication-and-roles:8000 (172.19.0.12:8000)
writing to stdout
-                    100% |********************************|   172  0:00:00 ETA
written to stdout
{"status":"healthy","service":"authentication","version":"1.0.0"}
```

**Result:** All connections succeed, confirming full internal network functionality is preserved across all services in the private network. The frontend can communicate with all backend services using Docker's internal DNS.

### Summary of Results

| Test Scenario | Pre-Segmentation | Post-Segmentation |
|---------------|------------------|-------------------|
| External scan reveals API Gateway | Port 8080 visible (filtered) | Port 8080 appears filtered |
| External scan reveals Database | Port 5432 visible (filtered) | Port 5432 appears filtered |
| Direct external access to API Gateway | Connection succeeds (VULNERABLE) | Connection refused (SECURE) |
| Frontend to API Gateway communication | Works | Works (via private network) |
| Functionality preserved | Yes | Yes |
| Security posture | Weak - Direct access possible | Strong - Only frontend exposed |

---

## 7. Conclusion

The Network Segmentation Pattern successfully isolates backend services while maintaining full application functionality. Through this laboratory, we have demonstrated:

### Key Achievements

1. **Reduced Attack Surface:** From 8+ exposed ports to a single public-facing port (frontend only)
2. **Eliminated Direct Backend Access:** All backend services are now unreachable from external networks
3. **Database Protection:** Databases are completely isolated within the private network
4. **Preserved Functionality:** Internal communication between services works seamlessly via Docker DNS
5. **Defense in Depth:** Added network-level security on top of application-level controls

### Security Principles Applied

- **Least Privilege:** Services only have access to the networks they absolutely need
- **Separation of Concerns:** Public and private services are logically and physically separated
- **Defense in Depth:** Multiple security layers (network isolation + application authentication)
- **Minimal Attack Surface:** Only necessary services are exposed to external networks

### Lessons Learned

1. **Docker Port Mappings Are Critical:** Simply having services on Docker networks doesn't protect them; port mappings create direct vulnerabilities
2. **"Filtered" ≠ "Secure":** Nmap showing filtered ports doesn't guarantee security; actual accessibility must be verified
3. **Network Segmentation Is Essential:** Internal services should never be directly accessible from external networks
4. **Frontend as Gateway:** The frontend acts as the controlled entry point, enforcing authentication before allowing backend access