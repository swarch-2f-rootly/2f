# Quality Attribute Scenario: Interoperability Pattern Validation in GKE/GCP

## Architectural Context

The Rootly system must interoperate with cyber-physical components: microcontroller devices deployed in agricultural fields that continuously stream sensor data. This scenario validates that the Interoperability Pattern is successfully implemented to ensure effective communication between external IoT devices and the data ingestion service.

**Prototype 3 Baseline (Docker)**: Microcontroller devices communicate with the data ingestion service using HTTP/REST over a local area network (LAN). Devices must be on the same network as the service, limiting deployment to local environments. The service validates incoming payloads against the expected schema and rejects values that are outside acceptable ranges.

**Prototype 4 Objective**: Verify that microcontroller devices successfully communicate with the data ingestion service from internet locations through the GCP Load Balancer. The service validates payloads and rejects values outside acceptable ranges while enabling remote device communication, allowing devices deployed in agricultural fields to send data over the internet.

## Architectural Pattern and Tactic

**Pattern**: Interoperability Pattern (Interface Adaptation)

The Interoperability Pattern enables communication between heterogeneous systems with different data formats, protocols, or schemas. The pattern focuses on the system's ability to understand, interpret, and adapt data from external systems, ensuring successful information exchange despite differences in implementation.

**Tactics (SAIP - System-Application Integration Pattern):**

- **Discover Service**: Devices learn the ingestion endpoint and understand the communication protocol. The service exposes a well-defined REST/HTTP API accessible from internet locations.

- **Orchestrate**: The data ingestion service coordinates request validation, parsing, and processing. It validates incoming payloads against the defined schema, checks required fields, and ensures data types match expectations before processing.

- **Interface Adaptation**: The service validates incoming data against the expected schema and range constraints. It checks that required fields are present, validates data types, and ensures measurement values are within acceptable ranges. Values outside extreme ranges are rejected with clear error messages.

## Quality Attribute Scenario

<img src="quality-scenario.png" alt="Quality Attribute Scenario" width="1000"/>

### 1. Artifact

**Device-to-Ingestion Contract:**

The REST/HTTP payload contract between the microcontroller device and the data ingestion service. The contract defines the expected JSON structure with required fields: controller_id, timestamp, and measurements. The measurements object contains four sensor values: temperature, humidity, soil_humidity, and light_intensity. Data types and validation rules ensure values are within acceptable ranges.

### 2. Source

**Cyber-physical microcontroller device:**

A microcontroller device equipped with environmental sensors (temperature, humidity, soil humidity, light intensity) deployed in agricultural fields. The device continuously collects sensor readings and sends telemetry frames to the data ingestion service every few seconds using HTTP/REST protocol over HTTPS.

### 3. Stimulus

**Continuous telemetry stream with variability:**

The stimulus consists of a continuous telemetry stream (e.g., one sample per second) with sensor measurements from microcontroller devices.

### 4. Environment

**GKE Production Environment with Internet Access:**

The system operates in a GKE production environment with the data ingestion service deployed in the `rootly-platform` namespace. The GCP Load Balancer provides a stable external IP address accessible from internet locations. Multiple microcontroller devices communicate with the service from remote agricultural field locations over the internet.

### 5. Response

**System Response to Device Communication:**

- **Discover service**: The device successfully identifies the data ingestion endpoint through the GCP Load Balancer external IP and understands the communication protocol. The service provides clear API documentation accessible from internet locations.

- **Orchestrate**: The data ingestion service validates incoming payloads against the expected schema, checks required fields, validates data types, and ensures semantic consistency. Valid payloads are accepted and processed, while invalid payloads are rejected with clear error messages.

- **Accepted exchanges**: Successfully processed telemetry frames are validated and forwarded to Kafka topics for downstream processing. The system logs successful exchanges with device identification for tracking.

- **Rejected exchanges**: Malformed, semantically inconsistent, or out-of-range requests are rejected with clear HTTP responses (400 Bad Request for syntax errors, 422 Unprocessable Entity for semantic errors) and detailed error messages explaining the issue. Rejected requests are logged with device ID and specific error details for operator review.

### 6. Response Measure

**Primary Metrics:**

- **Correctly processed exchanges**: Valid telemetry frames with values within acceptable ranges are accepted and forwarded to the internal pipeline.

- **Correctly rejected exchanges**: All malformed, semantically inconsistent, or out-of-range frames are rejected and logged with device ID.

- **Validation success**: The system successfully validates data from different device formats and rejects values outside acceptable ranges.

- **Internet connectivity success**: Devices successfully communicate from remote locations over the internet.

**Measurement Formula:**

$$\text{Processing Success Rate} = \frac{\text{Accepted Valid Frames}}{\text{Total Valid Frames}} \times 100\%$$

$$\text{Validation Success Rate} = \frac{\text{Successfully Validated Frames}}{\text{Total Frames}} \times 100\%$$

## Baseline: Docker Deployment (Prototype 3) - LAN Communication

### Device Communication Over LAN

In Docker Compose, the data ingestion service runs on a local machine and devices must be on the same LAN:

```bash
# Simulate device on same LAN sending data
curl -X POST http://192.168.1.100:8080/api/v1/data \
  -H "Content-Type: application/json" \
  -d '{
    "controller_id": "controller_1",
    "timestamp": "2024-12-15T10:30:00Z",
    "measurements": {
      "temperature": 23.45,
      "humidity": 65.20,
      "soil_humidity": 42.10,
      "light_intensity": 1250
    }
  }'
```

**Result:**

```
{"status":"accepted","message":"Data ingested successfully","id":"12345"}
```

The device successfully sends data when connected to the same LAN network. The service validates the payload and accepts values within acceptable ranges.

### Limitations in Docker Baseline

- **LAN Requirement**: Devices must be on the same local area network as the service. Remote devices in agricultural fields cannot communicate with the service over the internet.

- **Local Deployment**: The service runs on a local machine, limiting scalability and requiring physical network access for device communication.

## GKE Implementation (Prototype 4) - Internet Communication

### Device Communication from Internet

```bash
# Get GCP Load Balancer external IP
DATA_INGESTION_IP=$(kubectl get svc data-ingestion-loadbalancer -n rootly-platform -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# Simulate device sending data from remote location over internet
curl -X POST http://${DATA_INGESTION_IP}/api/v1/data \
  -H "Content-Type: application/json" \
  -d '{
    "controller_id": "controller_1",
    "timestamp": "2024-12-15T10:30:00Z",
    "measurements": {
      "temperature": 23.45,
      "humidity": 65.20,
      "soil_humidity": 42.10,
      "light_intensity": 1250
    }
  }'
```

**Result:**

```
{"status":"accepted","message":"Data ingested successfully","id":"67890"}
```

The device successfully sends data from a remote location over the internet through the GCP Load Balancer.

### Value Range Validation

```bash
# Test with values within acceptable range
curl -X POST http://${DATA_INGESTION_IP}/api/v1/data \
  -H "Content-Type: application/json" \
  -d '{
    "controller_id": "controller_2",
    "timestamp": "2024-12-15T10:32:00Z",
    "measurements": {
      "temperature": 23.5,
      "humidity": 65.0,
      "soil_humidity": 42.0,
      "light_intensity": 1250
    }
  }'
```

**Result:**

```
{"status":"accepted","message":"Data ingested successfully","id":"22222"}
```

The service accepts the data when all values are within acceptable ranges and forwards it to Kafka.

```bash
# Test with values outside acceptable range
curl -X POST http://${DATA_INGESTION_IP}/api/v1/data \
  -H "Content-Type: application/json" \
  -d '{
    "controller_id": "controller_2",
    "timestamp": "2024-12-15T10:32:00Z",
    "measurements": {
      "temperature": 150.0,
      "humidity": 65.0,
      "soil_humidity": 42.0,
      "light_intensity": 1250
    }
  }'
```

**Result:**

```
{"status":"rejected","error":"Temperature value 150.0 is outside acceptable range","code":422}
```

The service rejects the request when values are outside acceptable ranges with clear error messages.

### Invalid Payload Rejection

```bash
# Test with malformed payload from remote device
curl -X POST http://${DATA_INGESTION_IP}/api/v1/data \
  -H "Content-Type: application/json" \
  -d '{
    "controller_id": "controller_3",
    "timestamp": "2024-12-15T10:33:00Z",
    "measurements": {
      "temperature": "invalid_string",
      "humidity": 65.0,
      "soil_humidity": 42.0,
      "light_intensity": 1250
    }
  }'
```

**Result:**

```
{"status":"rejected","error":"Invalid measurement type: temperature must be numeric","code":422}
```

The service correctly rejects malformed payloads with clear error messages and logs the rejection with device identification.

### Valid Payload Example

```bash
# Test with complete valid payload from remote device
curl -X POST http://${DATA_INGESTION_IP}/api/v1/data \
  -H "Content-Type: application/json" \
  -d '{
    "controller_id": "controller_4",
    "timestamp": "2024-12-15T10:34:00Z",
    "measurements": {
      "temperature": 23.5,
      "humidity": 65.0,
      "soil_humidity": 42.0,
      "light_intensity": 1250
    }
  }'
```

**Result:**

```
{"status":"accepted","message":"Data ingested successfully","id":"44444"}
```

The service accepts data when all required fields are present, values are within acceptable ranges, and the schema is valid.

## Analysis

**Internet Communication Capability:**

The GKE implementation enables microcontroller devices to communicate with the data ingestion service from remote agricultural field locations over the internet. The GCP Load Balancer provides a stable external IP address accessible from any internet location, eliminating the LAN requirement of the Docker baseline. Devices deployed in fields can send sensor data directly to the service without requiring local network access.

**Validation and Range Checking:**

The data ingestion service validates incoming payloads against the expected schema, checks required fields, validates data types, and ensures measurement values are within acceptable ranges. Values outside extreme ranges are rejected with clear error messages and logged for operator review. This ensures downstream consumers receive valid data within expected ranges, maintaining data quality in the processing pipeline.

**Schema Validation:**

The service validates incoming payloads against the expected schema. Devices sending data with the required fields (controller_id, timestamp, measurements with temperature, humidity, soil_humidity, light_intensity) and values within acceptable ranges successfully communicate with the service from internet locations.

## Conclusion

The Interoperability Pattern with Interface Adaptation is successfully implemented in GKE. The system provides effective communication between cyber-physical microcontroller devices and the data ingestion service from internet locations through the GCP Load Balancer. The service validates incoming payloads against the expected schema and rejects values outside acceptable ranges, achieving high processing success rate for valid telemetry frames. The system enables remote device communication from agricultural field locations over the internet.
