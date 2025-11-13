
---

# Security Quality Attribute Scenario: Secure Channel Pattern Implementation and Validation

## Table of Contents

1. [Architectural Weakness and Security Context](#architectural-weakness-and-security-context)
2. [Quality Attribute Scenario](#quality-attribute-scenario)
3. [Security Scenario Analysis](#security-scenario-analysis)
4. [Vulnerability Demonstration (Pre-Secure Channel)](#vulnerability-demonstration-pre-secure-channel)
5. [Countermeasure Implementation (Secure Channel)](#countermeasure-implementation-secure-channel)
6. [Validation Results (Post-Secure Channel)](#validation-results-post-secure-channel)
7. [Response to Quality Scenario](#response-to-quality-scenario)
8. [Security Concepts Mapping](#security-concepts-mapping)
9. [CIA Triad Impact Assessment](#cia-triad-impact-assessment)
10. [Preservation of Functionality](#preservation-of-functionality)
11. [Visual Evidence (Wireshark)](#visual-evidence-wireshark)
12. [Security Tactic Implementation Validation](#security-tactic-implementation-validation)
13. [Final Recommendations](#final-recommendations)

---

## Architectural Weakness and Security Context

### System Vulnerability Overview

The **Rootly plant monitoring system** initially lacked a **Secure Channel** between the **web browser** and the **frontend web service (`frontend-ssr`)**. All communication occurred over **unencrypted HTTP**, exposing sensitive data (credentials, JWT tokens, sensor payloads) to interception by any actor on the same network.

#### The Fundamental Weakness: Absence of Secure Channel

* **Plaintext Transmission:** Requests and responses were sent over HTTP, readable by anyone with network access.
* **No Encryption:** No TLS/SSL was configured, so traffic could be intercepted and modified.
* **Attack Surface:** Any attacker with access to the network (Wi-Fi, Docker bridge, etc.) could capture and analyze packets.

#### Security Implications

| Vulnerability         | Description                            | Security Impact          |
| --------------------- | -------------------------------------- | ------------------------ |
| **Data Interception** | Attacker can read all HTTP traffic     | **Confidentiality Loss** |
| **Message Tampering** | Attacker can modify requests/responses | **Integrity Loss**       |
| **Session Hijacking** | Attacker can steal tokens/credentials  | **Critical Risk**        |

**Why This Matters:**
A malicious actor can exploit this weakness by capturing network traffic using tools like Wireshark or tcpdump, gaining access to sensitive data and potentially manipulating requests or hijacking sessions.

---

## Quality Attribute Scenario

### Scenario Elements

<img src="images/secure_channel.png" alt="Quality Attribute Scenario" width="1000"/>

| Element              | Description                                   |
| -------------------- | --------------------------------------------- |
| **Artifact**         | Frontend web service (`frontend-ssr`)         |
| **Source**           | Network attacker (internal/external)          |
| **Stimulus**         | Intercept and analyze traffic                 |
| **Environment**      | Normal operation, local or production network |
| **Response**         | All communication encrypted (TLS 1.2+)        |
| **Response Measure** | Readable HTTP packets = 0                     |

**Scenario Focus:**
This scenario evaluates the effectiveness of the **Secure Channel Pattern** as a countermeasure to the architectural weakness. It quantifies security by measuring the number of **readable network packets** containing sensitive information.

* **Exposure Baseline (Pre-Secure Channel):** Expected ≥ 5 readable packets per session
* **Validation Target (Post-Secure Channel):** **Goal = 0** readable packets

---

## Security Scenario Analysis

### CIA Triad Impact Assessment

The lack of a Secure Channel directly threatens the **Confidentiality**, **Integrity**, and **Availability** of the application's data and services.

#### Confidentiality Risk

Sensitive data (user credentials, plant information, sensor data) could be exposed if an attacker intercepts HTTP traffic.

#### Integrity Risk

Unauthorized modification or corruption of data if an attacker can tamper with requests or responses in transit.

#### Availability Risk

Potential denial of service if an attacker manipulates or floods unencrypted traffic.

| Property            | Pre-Secure Channel       | Impact   | Countermeasure           |
| ------------------- | ------------------------ | -------- | ------------------------ |
| **Confidentiality** | Data readable in transit | Critical | TLS encryption           |
| **Integrity**       | Data can be tampered     | High     | TLS validation           |
| **Availability**    | Minor risk               | Medium   | Proper TLS configuration |

### Six Key Security Concepts in the Scenario

| Concept            | Definition           | Secure Channel Scenario                    |
| ------------------ | -------------------- | ------------------------------------------ |
| **Weakness**       | Design flaw          | No encryption between browser and frontend |
| **Vulnerability**  | Exploitable path     | HTTP traffic readable by attacker          |
| **Threat**         | Agent/motivation     | Network attacker with packet capture tools |
| **Attack**         | Exploit sequence     | Capture and analyze HTTP packets           |
| **Risk**           | Probability × Impact | High risk of data breach                   |
| **Countermeasure** | Mitigation action    | Implement TLS (Secure Channel Pattern)     |

---

## Vulnerability Demonstration (Pre-Secure Channel)

This section demonstrates the security vulnerabilities present in the pre-Secure Channel architecture through a simulated attack scenario.

### Attack Prerequisites

Tools required for the attack simulation:

```bash
sudo apt-get update
sudo apt-get install tcpdump wireshark
```

### Phase 1: Network Reconnaissance

```bash
ip addr show | grep -E 'inet .* scope global'
```

**Example Output:**
`inet 192.168.1.10/24 brd 192.168.1.255 scope global dynamic noprefixroute wlp2s0`

### Phase 2: Packet Capture

```bash
sudo tcpdump -i any -w pre_tls.pcap tcp port 3001 or port 8080
```

**Wireshark Filter:** `http`

**Result:**
All headers, tokens, and JSON bodies are visible.

**Example:**

```
POST /api/v1/login HTTP/1.1
Authorization: Bearer eyJhbGciOiJIUzI1Ni...
{"username":"admin","password":"P@ssw0rd"}
```

### Phase 3: Direct Service Access

```bash
curl -v http://192.168.1.10:8080/api/v1/health --max-time 5
```

**Result:**
Connection succeeds, data exposed.

**Summary Table: Vulnerability Assessment**

| Service          | Port      | Access Result | Security Impact |
| ---------------- | --------- | ------------- | --------------- |
| Frontend         | 3001      | Accessible    | ✓ Intended      |
| API Gateway      | 8080      | Accessible    | ✗ Critical      |
| Backend Services | 8000-8003 | Accessible    | ✗ High          |
| Sensitive Data   | -         | Exposed       | ✗ Critical      |

---

## Countermeasure Implementation (Secure Channel)

After identifying the vulnerabilities, the **Secure Channel Pattern** was implemented to encrypt all communications and minimize the attack surface.

### Core Principle: Encrypt Data in Transit

The Secure Channel Pattern ensures that all data exchanged between the **browser** and **frontend web service** is protected from interception and tampering.

### Implementation Strategy

#### Step 1: TLS Certificate Generation

Automatic creation with OpenSSL in Docker:

```dockerfile
RUN apk add --no-cache openssl && \
    mkdir -p ./certs && \
    openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
      -keyout ./certs/localhost.key \
      -out ./certs/localhost.crt \
      -subj "/C=CO/ST=Bogota/L=Bogota/O=MyLab/OU=Dev/CN=localhost"
```

#### Step 2: Custom HTTPS Server

`server.js` handles HTTPS:

```js
const certs = {
  key: fs.readFileSync('./certs/localhost.key'),
  cert: fs.readFileSync('./certs/localhost.crt')
};

createServer(certs, (req, res) => {
  // Application logic
}).listen(3001, '0.0.0.0', () => {
  console.log('> HTTPS Server ready at https://localhost:3001');
});
```

#### Step 3: Configuration Updates

* Startup command in `package.json`: `node server.js`
* API endpoints updated to `https://`
* Environment variables:

  ```bash
  ENV NODE_ENV=production
  ENV HOSTNAME=0.0.0.0
  ENV PORT=3001
  ```

---

## Validation Results (Post-Secure Channel)

After implementing the Secure Channel Pattern, all previously exposed attack vectors were mitigated.

### Attack Attempts - All Vectors Blocked

#### Phase 1: Capture TLS Traffic

```bash
sudo tcpdump -i any -w post_tls.pcap tcp port 3001
```

#### Phase 2: Wireshark Analysis

* Filter: `tls`
* Result: Only encrypted **Application Data** is visible.

**Example:**

```
TLSv1.3 Client Hello
TLSv1.3 Server Hello, Certificate
TLSv1.3 Application Data (encrypted)
```

#### Phase 3: Functional Verification

* Browser shows secure connection.
* Server logs: `> HTTPS Server ready at https://localhost:3001`

**Summary Table: Attack Surface Comparison**

| Attack Vector              | Pre-Secure Channel | Post-Secure Channel |
| -------------------------- | ------------------ | ------------------- |
| Packet interception        | Vulnerable         | Blocked ✓           |
| Data tampering             | Possible           | Blocked ✓           |
| Credential theft           | Possible           | Blocked ✓           |
| Number of readable packets | 5+                 | 0 ✓                 |

---

## Response to Quality Scenario

| Scenario Element     | Requirement                                        | Implementation Response          |
| -------------------- | -------------------------------------------------- | -------------------------------- |
| **Artifact**         | Protect communication between browser and frontend | ✓ All traffic encrypted          |
| **Source**           | Defend against network attackers                   | ✓ TLS blocks packet inspection   |
| **Stimulus**         | Prevent interception of HTTP packets               | ✓ All packets encrypted          |
| **Environment**      | Maintain normal operation                          | ✓ System functionality preserved |
| **Response**         | All traffic encrypted                              | ✓ Achieved                       |
| **Response Measure** | Plaintext packets = 0                              | ✓ Met                            |

---

## Security Concepts Mapping

| Concept                                      | Status                                                   |
| -------------------------------------------- | -------------------------------------------------------- |
| **Vulnerability** (Unencrypted HTTP traffic) | **MITIGATED** - All traffic encrypted                    |
| **Attack** (Packet capture and analysis)     | **PREVENTED** - Only encrypted data visible              |
| **Countermeasure** (Secure Channel Pattern)  | **SUCCESSFULLY IMPLEMENTED** - Validated through testing |

---

## CIA Triad Impact Assessment

| Security Property   | Risk Status Pre-Secure Channel     | Protection Status Post-Secure Channel |
| ------------------- | ---------------------------------- | ------------------------------------- |
| **Confidentiality** | At Risk - Data readable in transit | **Protected** - All data encrypted    |
| **Integrity**       | At Risk - Data can be tampered     | **Protected** - TLS validation        |
| **Availability**    | Minor risk                         | **Maintained** - No negative impact   |

---

## Preservation of Functionality

### Verification of Internal Communication

* All legitimate communication between browser and frontend continues to function over HTTPS.
* API requests routed securely through the encrypted channel.
* No degradation in system functionality; security enhanced without operational impact.

**Example:**

```bash
curl -k https://localhost:3001/api/v1/health
```

**Output:**

```
{"status":"healthy","service":"rootly-apigateway"}
```

---

## Visual Evidence (Wireshark)

### 🔹 Without Secure Channel (HTTP)

![HTTP traffic without Secure Channel](images/http_no_secure_channel.png)

> All data transmitted in **plaintext**; credentials and payloads are visible.

### 🔹 With Secure Channel (HTTPS)

![HTTPS traffic with Secure Channel](images/https_with_secure_channel.png)

> Traffic is **fully encrypted**; only TLS handshake and “Encrypted Application Data” are visible.

---

## Security Tactic Implementation Validation

**Tactic Category:** Resist Attacks
**Specific Tactic:** Encrypt Data in Transit

**Validation Results:**

1. **Access Control Mechanism:** TLS implemented for all communications.
2. **Perimeter Definition:** All traffic passes through encrypted channels.
3. **Entry Point Control:** Only HTTPS endpoints exposed.
4. **Unauthorized Access Prevention:** All packet inspection attempts blocked.
5. **Legitimate Access Preservation:** System functionality maintained.

**Conclusion:**
The **Secure Channel Pattern** successfully enforces **Encrypt Data in Transit**, eliminating data exposure risks and strengthening Rootly’s security posture.

---


