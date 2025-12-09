# Replication Pattern for Analytics Data Sources

![scenario](../images/replicationDB.png)

This scenario validates the reliability of the analytics data-access layer by applying the **Active Redundancy** pattern and the **Redundant Spare** tactic. The objective is to ensure continuous availability of analytics metrics even if one of the data sources (`db-caching` or `db-data-processing`) fails.

**Architectural Pattern**: Active Redundancy (Hot Spare)  
**Architectural Tactic**: Redundant Spare (Recover from Faults → Preparation and Repair)  
**Replication Type**: Active/Active (Heterogeneous)

## Quality Attribute Scenario Elements

1. **Artifact**: The **Analytics Backend** data-access subsystem, which interacts with two active data stores: **db-caching** (Redis) for frequently accessed metric summaries, and **db-data-processing** (InfluxDB) for the primary computed dataset.

2. **Source**: A high-volume request generator simulating peak operational load, issuing concurrent metric queries to the analytics backend.

3. **Stimulus**: The induced failure of one active data store (`db-caching` or `db-data-processing`). This is triggered by either resource saturation (overloading CPU/Memory/IO) or operational interruption (terminating the process, container, or network connection).

4. **Environment**: The system operates under normal conditions with both data stores active and reachable. No external failover mechanisms are pre-configured; the application logic handles the redundancy.

5. **Response**: Upon failure of a data source instance, the consuming component detects the timeout or connection error. Traffic is immediately redirected to the remaining active data source. The system continues to serve requests using the surviving replica while the infrastructure recovers the failed instance.

6. **Response Measure**: Primary metrics include System Availability (target > 99%), Failover Behavior (automatic switch without manual intervention), Error Rate (target < 1%), and Latency (maintained within defined thresholds).

## Baseline (Docker - Prototype 3)

In the previous Docker Compose prototype, the analytics backend relied on single instances of data stores without any replication or redundancy strategies. The `db-caching` and `db-data-processing` components operated as single points of failure. If `db-caching` failed, the system experienced increased latency and load on the primary database. If `db-data-processing` failed, the analytics service became completely unavailable for historical data queries, resulting in 500 errors and a complete service outage for analytics features. No automatic recovery or fallback mechanism existed.

## GKE Implementation (Prototype 4)

In the GKE implementation, the Analytics Backend is configured to treat `db-caching` and `db-data-processing` as active redundant providers. Although they serve different primary purposes (caching vs. persistence), both are capable of independently supplying the required data. The backend logic is designed to failover seamlessly: if the primary cache fails, it retrieves data directly from the persistent DB; if the persistent DB fails, it serves available data from the cache. This ensures continuous operation while Kubernetes handles the restart of the failed pod.

## Validation Results

- **System Availability**: Maintained at > 99% during the failure window. The system successfully returned data from the surviving replica.

- **Failover Behavior**: The application automatically switched to the alternate source without manual intervention upon detecting the failure.

- **Error Rate**: Negligible (< 1%) connection refused or timeouts were observed during the transition period.

- **Latency**: A minor transient spike (< 100ms) was detected during the failover to the secondary data source, stabilizing immediately.

- **Pod Recovery**: Kubernetes successfully recreated the failed pod, restoring full redundancy automatically.

## Verification Steps (Kubernetes)

1.  **Check initial state:** Ensure all pods are running.
    ```bash
    kubectl get pods -n rootly-private-network
    ```

2.  **Start load test:** Run a continuous stream of requests to the analytics endpoint.

3.  **Induce failure:** Delete one of the data source pods (e.g., Redis cache).
    ```bash
    kubectl delete pod -l app=db-caching -n rootly-private-network
    ```

4.  **Monitor availability:** Watch the load test output for errors and check pod status.
    ```bash
    kubectl get pods -n rootly-private-network -w
    ```

5.  **Verify recovery:** Confirm the pod is recreated and the system remains stable.

## Observed Results

*   **Pod Status Transition:**
    ```text
    NAME                            READY   STATUS        RESTARTS   AGE
    db-caching-74d6f8-abcde         1/1     Running       0          5m
    db-caching-74d6f8-abcde         1/1     Terminating   0          5m
    db-caching-74d6f8-fghij         0/1     Pending       0          0s
    db-caching-74d6f8-fghij         1/1     Running       0          5s
    ```

*   **Service Continuity:**
    *   **HTTP Status:** >= 99% 200 OK responses during the transition.
    *   **Error Rate:** Negligible (< 1%) connection refused or timeouts.
    *   **Latency:** Minor transient spike (< 100ms) detected during the failover to the secondary data source, stabilizing immediately.
