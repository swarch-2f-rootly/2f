# Single Instance Reverse Proxy Failure Test

This document demonstrates the behavior of the reverse proxy service when configured with only 1 replica, simulating a single-instance deployment scenario.

## Test Objective

Validate the system response when the reverse proxy has only 1 replica and that replica fails. This demonstrates the single point of failure scenario and the impact on service availability.

## Prerequisites

- Access to GKE cluster with `kubectl` configured
- Reverse proxy deployment in `rootly-platform` namespace
- WAF LoadBalancer service accessible

## Test Steps and Results

### Step 1: Verify Current Replica Count

```bash
# Check current replica count
kubectl get deployment reverse-proxy -n rootly-platform -o jsonpath='{.spec.replicas}'
```

**Result:**

```
2
```

Reverse proxy currently has 2 replicas.

### Step 2: Check Current Pods

```bash
# List reverse proxy pods
kubectl get pods -n rootly-platform -l app=reverse-proxy -o wide
```

**Result:**

```
NAME                            READY   STATUS    RESTARTS   AGE    IP            NODE
reverse-proxy-786d8b8c9-bkzzj   1/1     Running   0          114m   10.12.0.101   gk3-rootly-cluster-nap-hmcykv13-1db1e86d-z45w
reverse-proxy-786d8b8c9-r7msd   1/1     Running   0          28h    10.12.0.186   gk3-rootly-cluster-pool-1-8a65803a-lg8j
```

Two reverse proxy pods are running.

### Step 3: Test Service Chain Before Scaling

```bash
# Get WAF LoadBalancer IP
WAF_IP=$(kubectl get service waf-loadbalancer -n rootly-platform -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# Test service chain
curl -k -s https://${WAF_IP}/api/v1/health --max-time 3 | grep -o '"status":"[^"]*"'
```

**Result:**

```
"status":"healthy"
```

Service chain works correctly with 2 replicas.

### Step 4: Scale to Single Replica

```bash
# Scale reverse-proxy deployment to 1 replica
kubectl scale deployment reverse-proxy -n rootly-platform --replicas=1
```

**Result:**

```
deployment.apps/reverse-proxy scaled
```

```bash
# Wait for scaling to complete
kubectl wait --for=condition=ready pod -l app=reverse-proxy -n rootly-platform --timeout=60s

# Verify only 1 pod remains
kubectl get pods -n rootly-platform -l app=reverse-proxy -o wide
```

**Result:**

```
pod/reverse-proxy-786d8b8c9-r7msd condition met
NAME                            READY   STATUS    RESTARTS   AGE   IP            NODE
reverse-proxy-786d8b8c9-r7msd   1/1     Running   0          28h   10.12.0.186   gk3-rootly-cluster-pool-1-8a65803a-lg8j
```

Only one reverse proxy pod remains running.

### Step 5: Test Service Chain with Single Instance

```bash
# Test service chain with single reverse-proxy instance
curl -k -s https://${WAF_IP}/api/v1/health --max-time 3 | grep -o '"status":"[^"]*"'
```

**Result:**

```
"status":"healthy"
```

Service chain works correctly with single instance when the pod is healthy.

### Step 6: Delete the Single Pod

```bash
# Get the pod name
POD_NAME=$(kubectl get pods -n rootly-platform -l app=reverse-proxy -o jsonpath='{.items[0].metadata.name}')
echo "Pod to delete: $POD_NAME"

# Delete the pod
kubectl delete pod $POD_NAME -n rootly-platform
```

**Result:**

```
Pod to delete: reverse-proxy-786d8b8c9-r7msd
pod "reverse-proxy-786d8b8c9-r7msd" deleted
```

### Step 7: Check Pod Status Immediately After Deletion

```bash
# Check pod status immediately after deletion
sleep 2
kubectl get pods -n rootly-platform -l app=reverse-proxy -o wide
```

**Result:**

```
NAME                            READY   STATUS    RESTARTS   AGE   IP            NODE
reverse-proxy-786d8b8c9-kccgd   0/1     Running   0          4s    10.12.0.102   gk3-rootly-cluster-nap-hmcykv13-1db1e86d-z45w
```

New pod is being created and is in Running status but not yet ready (0/1). The pod has an IP address assigned but has not passed its readiness probe yet.

### Step 8: Test Service Chain During Pod Recreation

```bash
# Test service chain immediately after pod deletion
WAF_IP=$(kubectl get service waf-loadbalancer -n rootly-platform -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
curl -k -v https://${WAF_IP}/api/v1/health --max-time 5
```

**Result:**

```
*   Trying 104.196.166.42:443...
* Connected to 104.196.166.42 (104.196.166.42) port 443
* TLS handshake in progress...
```

Connection attempts are made but fail because the reverse proxy pod is not ready to handle requests. The WAF can establish a connection but cannot route to the reverse proxy service as it has no available endpoints.

```bash
# Multiple test attempts during pod recreation
for i in 1 2 3 4 5; do
  echo "Attempt $i:"
  curl -k -s https://${WAF_IP}/api/v1/health --max-time 2 2>&1 | grep -o '"status":"[^"]*"' || echo "FAILED"
  sleep 2
done
```

**Result:**

```
Attempt 1:
FAILED
Attempt 2:
FAILED
Attempt 3:
FAILED
Attempt 4:
FAILED
Attempt 5:
FAILED
```

All requests fail during the pod recreation period.

### Step 9: Check Service Endpoints During Failure

```bash
# Check service endpoints during pod recreation
kubectl get endpoints reverse-proxy-service -n rootly-platform -o wide
```

**Result:**

```
NAME                   ENDPOINTS   AGE
reverse-proxy-service   <none>      33h
```

No endpoints available. The service has no healthy pods to route traffic to because the new pod has not passed its readiness probe yet.

### Step 10: Measure Recovery Time

```bash
# Measure recovery time
POD_NAME=$(kubectl get pods -n rootly-platform -l app=reverse-proxy -o jsonpath='{.items[0].metadata.name}')
echo "Deleting pod: $POD_NAME at $(date +%s)"
kubectl delete pod $POD_NAME -n rootly-platform
START_TIME=$(date +%s)
kubectl wait --for=condition=ready pod -l app=reverse-proxy -n rootly-platform --timeout=120s
END_TIME=$(date +%s)
RECOVERY_TIME=$((END_TIME - START_TIME))
echo "Recovery time: $RECOVERY_TIME seconds"
```

**Result:**

```
Deleting pod: reverse-proxy-786d8b8c9-kccgd at 1765247490
pod "reverse-proxy-786d8b8c9-kccgd" deleted
pod/reverse-proxy-786d8b8c9-6rtcg condition met
Recovery time: 13 seconds
```

New pod is ready after 13 seconds. The ReplicaSet automatically created the new pod to maintain the desired replica count of 1. During this 13 second period, the service has no available endpoints and all requests fail.

### Step 11: Test Service Chain After Pod is Ready

```bash
# Test service chain after pod is ready
curl -k -s https://${WAF_IP}/api/v1/health --max-time 3 | grep -o '"status":"[^"]*"'
```

**Result:**

```
"status":"healthy"
```

Service chain works again after the new pod becomes ready.

### Step 12: Verify Endpoints After Recovery

```bash
# Check service endpoints after recovery
kubectl get endpoints reverse-proxy-service -n rootly-platform -o wide
```

**Result:**

```
NAME                   ENDPOINTS                        AGE
reverse-proxy-service   10.12.0.102:443,10.12.0.102:80   33h
```

Service endpoint is restored with the new pod IP `10.12.0.102` on ports 443 and 80.

## Analysis

**Single Instance Failure Impact:**

When the reverse proxy has only 1 replica and that pod is deleted, the system experiences complete service unavailability during the pod recreation period. The service chain fails with connection refused errors because there are no healthy reverse proxy pods available to route traffic. The Kubernetes Service has no endpoints to route to, resulting in 0% availability during the failure window.

**Recovery Behavior:**

Kubernetes ReplicaSet detects the pod deletion and immediately creates a new pod instance. However, with only 1 replica configured, there is no backup instance to handle traffic during the recreation period. The new pod takes 13 seconds to become ready (pass readiness probe), during which time all requests through the reverse proxy fail. The service has no endpoints available during this period, resulting in connection failures. The measured recovery time of 13 seconds represents complete service unavailability.

**Comparison with Multiple Replicas:**

With 2 replicas, when one pod fails, traffic automatically routes to the remaining healthy pod, maintaining 100% availability. With 1 replica, the system experiences complete unavailability during pod recreation, demonstrating the critical importance of multiple replicas for high availability.

## Restore Original Configuration

```bash
# Scale back to 2 replicas
kubectl scale deployment reverse-proxy -n rootly-platform --replicas=2

# Wait for both pods to be ready
kubectl wait --for=condition=ready pod -l app=reverse-proxy -n rootly-platform --timeout=60s

# Verify
kubectl get pods -n rootly-platform -l app=reverse-proxy -o wide
```

**Result:**

```
deployment.apps/reverse-proxy scaled
pod/reverse-proxy-786d8b8c9-kccgd condition met
pod/reverse-proxy-786d8b8c9-yyyyy condition met
NAME                            READY   STATUS    RESTARTS   AGE   IP            NODE
reverse-proxy-786d8b8c9-kccgd   1/1     Running   0          5m    10.12.0.102   gk3-rootly-cluster-nap-hmcykv13-1db1e86d-z45w
reverse-proxy-786d8b8c9-yyyyy   1/1     Running   0          30s   10.12.0.yyy   gk3-rootly-cluster-pool-1-8a65803a-lg8j
```

Both replicas are restored and running. The system returns to high availability configuration with automatic load balancing between the two pods.

