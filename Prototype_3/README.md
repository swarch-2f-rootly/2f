# Rootly 
## Prototype 3 - Quality Attributes, Part 1

## Team 2F
- Carlos Santiago Sandoval Casallas - Cristian Santiago Tovar Bejarano - Danny Marcelo Yaluzan Acosta - Esteban Rodriguez Muñoz - Santiago Restrepo Rojas - Gabriela Guzmán Rivera - Gabriela Gallegos Rubio - Andrés Camilo Orduz Lunar.

## Table of Contents

- [Software System](#software-system)
- [Architectural Structures](#architectural-structures)
  - [Component-and-Connector Structure](#component-and-connector-view)
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
    - [otro patron](#otro-patron)
- [Prototype – Deployment Instructions](#deployment-instructions)
  
---

## Software System
- **Name:** Rootly  
- **Logo:**  
<img src="images/Rootly.png" alt="Rootly Logo" width="100"/>

- **Description:**  
**ROOTLY** is an agricultural monitoring system designed to support significant decision-making in the agricultural environment. It leverages a microservices-based architecture to integrate field devices, process data, and deliver real-time analytics to users through a web and mobile application.

The system operates by capturing environmental and soil data—such as humidity and temperature—directly from the field using microcontroller devices. This information is then sent to a central platform where it is processed, validated, and analyzed. The platform's architecture combines robust databases for storing both transactional information (like user profiles and configurations) and large volumes of time-series sensor data.

Finally, users can access all this information through an intuitive interface, available on both web and mobile. They can view real-time metrics, explore historical data, manage their crops, and receive analytical insights to optimize their agricultural practices.

---

# Architectural Structures
## Components and Connector Structure
### Components and Connector view
![Component-and-connector-view](images/C&C_View_Prototype2.png)


### External Components (Out of System Scope)
- **Web Browser**
  - Type: External client
  - Responsibility: User interface for web applications
  - Protocols: HTTP
  - Relations: Consumes `FrontEnd` via HTTP
- **Microcontroller Device (different implementations)**
  - Type: External IoT/embedded device
  - Responsibility: Collects and sends sensor and plant data
  - Protocols: HTTP/REST
  - Relations: Consumes `BackEnd data-ingestion` via HTTP/REST

### Internal Components

**Frontend Service**  
  - Type: Web client application
  - Responsibility: Provides the user interface for the web platform; handles presentation logic, user interactions, and requests to backend services.
  - Role in connections: Consumer
  - Relations:
    - Serves the "web-browser"
    - Consumes `API Gateway` (HTTP/GraphQL)

- **Frontend Mobile**  
  - Type: Mobile client application
  - Responsibility: Provides a mobile-optimized interface consuming backend APIs directly via REST.
  - Role in connections: Consumer
  - Relations:
    - Consumes `API Gateway` (HTTP/REST)


**api-gateway**

  - Type: Gateway / API Orchestrator
  - Responsibility: Central entry point for all clients (web and mobile). Routes, aggregates, and authenticates API requests to backend microservices.

  - Protocols:
    - HTTP/GraphQL/REST — to `be-analytics`
    - HTTP/REST — to `be-authentication-and-roles` and `be-user-plant-management`.

  - Relations:
    - Receives requests from `fe-web` and `fe-mobile`.
    - Consumes services from backend modules (`be-user-plant-management`, `be-authentication-and-roles`, `be-analytics`).

**be-authentication-and-roles**
  - Type: Backend Microservice
  - Responsibility: Authentication and authorization service that manages users, credentials, and access control
  - Relations:
    - Serves the `FrontEnd` (HTTP/REST)
    - Connects to `DB Authentication-and-roles`
    - Connects to `STG Authentication-and-roles`

**be-user-plant-management**
  - Type: Backend Microservice
  - Responsibility: Plant and user management service that administers users, plants and devices relationships
  - Relations:
    - Serves the `FrontEnd` (HTTP/REST)
    - Connects to `DB User-plant-management`
    - Connects to `STG User-plant-management`

**be-analytics**
  - Type: Backend Microservice
  - Responsibility: Analytics and reporting service for analytical processing and insight generation
  - Protocols: HTTP/GraphQL for flexible queries
  - Relations:
    - Serves the `FrontEnd` (HTTP/GraphQL)
    - Connects to `DB Data-management`

**be-data-ingestion**
  - Type: Backend Microservice
  - Responsibility: Receives and ingests sensor data from microcontroller devices, validating and forwarding to the processing pipeline.
  - Protocols: HTTP/REST (red) from devices, Kafka WIRE (pink dotted) to `queue-data-ingestion`.
  - Relations:
    - Consumes data from `microcontroller-device`
    - Produces asynchronous messages to `queue-data-ingestion`.


**be-data-processing**  
  - Type: Backend Microservice
  - Responsibility: Transforms, aggregates, and stores data received from ingestion queues.
  - Protocols: Kafka WIRE, HTTP/GraphQL (blue data resource lines).
  - Relations:
    - Consumes data from `queue-data-ingestion`.
    - Produces processed results stored in `stg-data-processing` and `db-data-processing`.
    - Provides processed `data to be-analytics`.

**queue-data-ingestion**
  - Type: Message Broker (Kafka)
  - Responsibility: Asynchronous event queue for decoupling ingestion and processing services.
  - Protocols: Kafka WIRE connector (pink dotted).
  - Relations:
    - Producer: be-data-ingestion.
    - Consumer: be-data-processing.


### IoT Devices
- **microcontroller-device**  
  - Capture field data such as soil humidity.  
  - Send measurements to the Data Ingestion Service via **REST API calls**.  

### Data Components
- **Databases (DB)**
  - `DB Authentication-and-roles`: Transactional storage for users and permissions
  - `DB User-plant-management`: Transactional storage for plants and relationships
  - `DB Data Processing`: Storage for processed and analytical data
- **Storage (STG)**
  - `STG Authentication-and-roles`: Reference data for authentication
  - `STG User-plant-management`: Master data and plant configurations
  - `STG Data-processing`: Temporary or raw storage for device data

---

### Architectural Styles
1. **Client–Server**  
   The browser (client) requests resources from the frontend (server) via HTTP. This ensures separation between presentation and business logic.  

2. **Microservices Architecture**  
   The backend is decomposed into a set of fine-grained, independently deployable services. Each service is responsible for a specific business capability (e.g., authentication, user management, analytics). This approach promotes scalability, resilience, and technological flexibility, as each service can be developed, deployed, and scaled independently. Communication between services is handled via well-defined APIs (REST/GraphQL) and asynchronous messaging (Kafka).

### Architectural Patterns

1.  **API Gateway Pattern**
    This pattern provides a single, unified entry point for all client requests. The `api-gateway` component acts as a reverse proxy, routing requests from the frontend clients to the appropriate backend microservice. This simplifies the client application, offloads responsibilities like authentication and SSL termination, and provides an additional layer of security.

---

### Architectural Elements & Relations


#### **Web Browser**
- External actor that runs the frontend (SPA in React).  
- Consumes REST/GraphQL APIs with JWT authentication.  
- Serves as the user interface between client and system.  

#### **Frontend (rootly-frontend)**
- User interface built with React + TypeScript.  
- Displays agricultural metrics and real-time dashboards.  
- Communicates only with backend services.  

#### **Mobile App (rootly-mobileapp)**
- Native mobile application that provides a user-friendly interface for on-the-go monitoring and management.
- Consumes the same backend services as the web frontend via the API Gateway.

#### **API Gateway (api-gateway)**
- Single entry point for all client applications (web and mobile).
- Routes requests to the appropriate microservice, handles authentication, and aggregates responses.

#### **Backend Services**
- **Authentication and Roles (be-authentication-and-roles):** Handles login, roles, and JWT tokens.  
- **User and Plant Management (be-user-plant-management):** Manages users, plants, and microcontrollers.  
- **Analytics (be-analytics):** Processes data, generates metrics, and analytical reports.  
- **Data Ingestion (be-data-ingestion):** Receives sensor data from IoT devices and publishes it to the message queue.
- **Data Processing (be-data-processing):** Consumes data from the message queue, transforms it, and stores it in the appropriate database.

#### **Message Queue (queue-data-ingestion)**
- Kafka-based message broker that decouples the data ingestion process from data processing.
- Ensures reliable, asynchronous communication for high-volume sensor data.

#### **microcontroller-device**
- ESP8266 microcontrollers with environmental sensors.  
- Capture humidity, temperature, etc., and send data to the backend via HTTP POST.  

#### **Databases and Storage**
- **PostgreSQL:** SQL storage of users, roles, plants and physical devices.  
- **InfluxDB:** Time-series storage of agricultural data (port 8086).  
- **MinIO:** Distributed storage for profiles, plant images, and backups.  

### Frontend and Backend Services

| Component / Service | Port(s) | Responsibilities | Boundaries | Interfaces |
|---|---|---|---|---|
| **Frontend** | 3001 | Provides main user interface, real-time visualization of sensor data, dashboards, and plant/device management. | SPA executed in the browser, depends on the `api-gateway`. | Communicates via `api-gateway` using REST/GraphQL. |
| **api-gateway** | 8080 | Central entry point for clients. Routes, aggregates, and authenticates API requests to backend microservices. | Orchestrates backend services; no business logic. | REST/GraphQL to clients; REST/GraphQL to backends. |
| **be-authentication-and-roles** | 8001 | User authentication (login/logout), role-based access control (RBAC), JWT token management, user lifecycle. | Independent service with dedicated database; does not access sensor/plant data directly. | REST APIs for auth and roles. |
| **be-user-plant-management** | 8003 | Manages plants, user-device associations, microcontroller lifecycle, enabling/disabling devices. | Specialized in domain entities (plants, devices); relies on auth service for user information. | REST APIs for CRUD on devices/plants. |
| **be-analytics** | 8000 | Processes and analyzes sensor data, computes agricultural metrics, trend analysis. | Read-only access to sensor data; no modification of primary records; specialized in analytics. | REST/GraphQL APIs for reports and trends. |
| **be-data-ingestion** | 8005 | Receives and ingests sensor data from microcontroller devices, validating and forwarding to the processing pipeline. | Single entry point for IoT data; ensures data integrity; does not perform advanced analytics. | HTTP endpoints for ingestion. |
| **be-data-processing** | 8002 | Transforms, aggregates, and stores data received from ingestion queues. | Consumes from message queue and stores data. | Communicates via Kafka; provides data to other services. |


### Databases and Storage

| Component / Service | Port(s) | Description |
|---|---|---|
| **queue-data-ingestion** | 9092 (UI: 8082) | Kafka message broker for asynchronous event queue. |
| **db-authentication-and-roles** | 5432 | PostgreSQL: `authentication_and_roles_db` with users, roles, permissions, sessions, and tokens. |
| **db-user-plant-management** | 5433 | PostgreSQL: `user_plant_management_db` with user–plant–device associations. |
| **db-data-processing** | 8086 | InfluxDB: `agricultural_data` bucket with processed measurements and historical metrics. |
| **stg-authentication-and-roles** | 9002 (Console: 9003) | MinIO storage for profile photos. |
| **stg-data-processing** | 9000 (Console: 9001) | MinIO storage for data files, backups, unstructured content. |
| **stg-user-plant-management** | 9004 (Console: 9005) | MinIO storage for plant images. |

---

---

## Deployment Structure
### Deployment view

![deployment-view](deployView.png)

### Description of Architectural Elements and Relations

Based on the deployment view, the system's elements are allocated as follows:

*   **Software Elements:** The core of the system resides in `Node 1`, where all services are deployed as Docker containers. This includes:
    *   **Client Applications:** The `fe-ssr-web` (Web Frontend).
    *   **API Gateway:** The `api-gateway` acts as the central communication hub.
    *   **Backend Microservices:** A suite of specialized services for authentication (`be-authentication-and-roles`), analytics (`be-analytics`), data handling (`be-data-ingestion`, `be-data-processing`), and user/plant management (`be-user-plant-manager`).
    *   **Support Services:** Asynchronous communication is managed by `queue-data-ingestion` (Kafka), and persistence is handled by dedicated database (`db-*`) and storage (`stg-*`) containers.

*   **Environmental Elements:**
    *   **Node 1 (Server Cluster):** A physical or virtual server that hosts the Docker environment, running all the containerized software elements on a shared internal network.
    *   **Node 2 (Mobile Client):** A physical device (e.g., a smartphone) running the `fe-mobile` application.
    *   **Node 3 (IoT Device):** A physical microcontroller deployed in the field to collect data.

*   **Relations (Network Communication):**
    *   **External Communication:** The `fe-mobile` (Node 2) and `microcontroller` (Node 3) communicate with services in Node 1 over the same LAN network. The mobile app connects to the `api-gateway`, while the microcontroller sends data directly to `be-data-ingestion`.
    *   **Internal Communication:** Inside Node 1, all containers communicate securely over a private Docker network (`rootly-network`). The `fe-ssr-web` and `api-gateway` interact, and the gateway routes requests to the appropriate backend microservices, which in turn connect to their databases or the message queue.

### Description of Architectural Patterns Used

The deployment structure reveals several key architectural patterns:

*   **Event-Driven (Message Broker):** The `queue-data-ingestion` (Kafka) container decouples the data ingestion and processing pipelines. This allows the system to handle high volumes of sensor data asynchronously and reliably.
*   **Database per Service:** Each microservice has its own dedicated persistence container (e.g., `be-authentication-and-roles` has `db-authentication-and-roles` and `stg-authentication-and-roles`). This ensures data isolation and allows each service to evolve independently.
*   **Containerization:** Every software component in Node 1 is deployed as a Docker container. This standardizes the deployment, simplifies dependency management, and ensures consistency across environments.


## Layered Structure

### Layered view

![Layer-view](Layer_view_Prototype2.png)

** **Layered view  - Layers** The structure of the logic layer is shown below to avoid redundancy in the main view. In general, each component follows the same structure.

### Layer Specifications
- **Layer T1 – Presentation Layer**
  - Responsibility: Renders user interfaces and handles client-side logic
  - Components: Mobile frontend (iOS/Android), Web frontend (browsers)
  - Allowed dependencies: Only `T2` (synchronous communication)
  - Communication style: Synchronous HTTP/REST APIs
  - Constraint: Cannot bypass `T2` to reach lower layers
- **Layer T2 – Synchronous Communication Layer**
  - Responsibility: API gateway, request routing, and synchronous facade
  - Capabilities: Request validation, authentication, rate limiting, throttling, routing, composition, and protocol translation
  - Allowed dependencies: Only `T3`
  - Communication style: Synchronous (HTTP/gRPC)
  - Pattern: API Gateway routing pattern
- **Layer T3 – Logic Layer**
  - Responsibility: Core business logic and application functionality
  - Allowed dependencies: Can access `T4` (Asynchronous Communication) and `T5` (Data).
  - Synchronous services (request/response):
    - `Authentication and Roles BE`
    - `User Plant Management BE`
    - `Analytics BE`
  - Asynchronous services (producer/consumer with queues):
    - `Data Ingestion BE` (producer): Receives and validates incoming data streams
    - `Data Processing BE` (consumer): Processes queued data asynchronously
- **Layer T4 – Asynchronous Communication Layer**
  - Responsibility: Message queues and decoupling
  - Component: `Queue Data Ingestion` as the messaging broker for the ingestion pipeline
  - Capabilities: Reliable delivery, decoupling ingestion from processing, buffering during load peaks
- **Layer T5 – Data Layer**
  - Responsibility: Data persistence and storage management
  - Datastores:
    - `Authentication and Roles DB` and `STG Authentication and Roles`
    - `User Plant Management DB` and `STG User Plant Management`
    - `DB Data Processing` and `STG Data Processing`

**Logic Layer Structure (Internal)**
The diagram below illustrates the internal architecture of each microservice within the Logic Layer (T3). To maintain a clear separation of concerns and promote modularity, each service adopts a 4-layer architecture.

![Layer-logic-view](Layer_logic_Prototype2.png)

### Architectural patterns
1. **Layered Architecture (Within each microservice)**
   Each microservice is internally structured into four distinct layers. This pattern ensures that responsibilities are clearly segregated, making the service easier to develop, test, and maintain. The dependencies flow in one direction, from the outer layers to the inner layers.
    -   **Layer 1: Adapters:** This outermost layer is responsible for protocol-specific communication. It adapts incoming requests (e.g., from Kafka consumers or other services) and translates them into a format that the application's core can understand.
    -   **Layer 2: Controllers:** This layer acts as the entry point for API requests (e.g., HTTP REST/GraphQL). It handles request validation, parsing, and calls the appropriate business logic in the Services layer.
    -   **Layer 3: Services:** This is the core of the microservice, containing the main business logic and orchestrating application operations. It is completely independent of the delivery mechanism (e.g., HTTP).
    -   **Layer 4: Models:** This innermost layer represents the application's data structures and domain entities. It is responsible for data access and persistence logic, interacting directly with the database.

2.  **5-Tier Architecture Pattern**
    The system is structured into five logical tiers, each with a distinct responsibility, creating a clear separation of concerns:
    -   **Tier 1: Presentation:** The user-facing layer, which includes the web and mobile frontends. It is responsible for rendering the user interface and capturing user interactions.
    -   **Tier 2: Synchronous Communication & Orchestration:** This tier is embodied by the `api-gateway`, which manages all synchronous (request-response) communication from the presentation layer to the backend.
    -   **Tier 3: Logic:** This is the core of the application, containing the business logic implemented within the individual microservices (`be-authentication-and-roles`, `be-user-plant-management`, `be-analytics`, etc.).
    -   **Tier 4: Asynchronous Communication:** This tier uses a message broker (`queue-data-ingestion` with Kafka) to decouple services and manage asynchronous data flows, particularly for data ingestion and processing.
    -   **Tier 5: Data:** The persistence layer, which includes all databases and storage systems (`db-*`, `stg-*`) responsible for storing and managing data.

## Decomposition Structure
### Decomposition View

![Decomposition-view](Decmp_view_Prototype2.png)


### Purpose
Shows the hierarchical breakdown of the system into functional modules, clarifying responsibilities from high-level features down to services.

### Decomposition Hierarchy
1. **Authentication and User Management**
   - User management: Create account, update account, delete account
   - User authentication: Sign in, sign out, change password
2. **Plant and Device Administration**
   - Plant management:
     - Create plant, delete plant, update plant
     - Add plant photo, remove plant photo
     - List all plants, list plant by ID
     - List devices per plant
     - Enable monitoring, disable monitoring
     - Associate device, disassociate device
   - Device management:
     - Create device, update device, delete device
     - List all devices, list device by ID
     - List devices belonging to a user
     - Update device for a user, delete device for a user
     - Enable device, disable device
3. **Data Ingestion**
   - Sensor data reception
   - Publication to Kafka
4. **Data Processing**
   - Kafka consumption
   - Data storage
5. **Data Analytics**
   - Data processing:
     - Query historical data
     - Query averaged historical data
   - Visualization processing:
     - Perform trend analysis
   - Report generation:
     - Generate single-metric report
     - Generate multi-metric report

### Description of architectural elements and relations

| Element | Type | Description | Relations |
| --- | --- | --- | --- |
| Authentication and User Management | Module | Manages the entire user lifecycle, including account creation, authentication, credential maintenance, sign-in, sign-out, and password changes. | Interacts with every other module to validate user identity and authentication before operations proceed. |
| User Management | Submodule | Handles the creation, update, and deletion of user accounts within the system. | Triggered by administrators or user self-service registration flows. |
| User Authentication | Submodule | Manages sign-in, sign-out, and password change workflows. | Depends on the authentication layer to validate credentials and issue access tokens. |
| Plant and Device Administration | Module | Governs plant and device resources, covering configuration, monitoring, and associations between assets. | Coordinates with data ingestion and processing modules to obtain telemetry from registered devices. |
| Plant Management | Submodule | Oversees plant lifecycle operations, photo attachments, monitoring status, and device associations. | Relies on the authentication module to verify the user responsible for each plant. |
| Device Management | Submodule | Administers device lifecycle tasks and relationships with users and plants. | Communicates with the ingestion module to receive data captured by devices. |
| Data Ingestion | Module | Receives real-time sensor and device data, publishing it to the messaging system (Kafka) for downstream processing. | Provides ingested data streams to the data processing module. |
| Data Processing | Module | Consumes Kafka topics, performs transformations, cleansing, and aggregations, and persists processed outputs. | Supplies clean datasets to the analytics module for further insights. |
| Data Analytics | Module | Analyzes and visualizes processed data to generate insights, reports, and trend identifications. | Depends on the data processing module to access consolidated information. |
| Data Processing (Analytics Submodule) | Submodule | Executes statistical calculations, aggregations, and historical or averaged queries. | Feeds computed datasets to the visualization submodule. |
| Statistics Processing | Submodule | Builds trend analyses based on processed data. | Supplies the report generation submodule with analytical results. |
| Report Generation | Submodule | Produces individual metric and comparative multi-metric reports for presentation or export. | Interfaces with the system's UI/dashboard layer to deliver finished reports. |

---
  
# Quality Attributes

## Security
### Network Segmentation
### Secure Channel
### Reverse Proxy
### Web Application Firewall

## Performance and Scalability
### Load Balancers
### otro patron

---
# Prototype – Deployment Instructions
Prototype – Deployment Instructions
