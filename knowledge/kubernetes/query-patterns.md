## Common PromQL Query Patterns for Kubernetes Monitoring

**Goal:** This document outlines standard and effective PromQL query patterns used specifically within a Kubernetes monitoring context. It assumes knowledge of basic PromQL syntax and the common Kubernetes metric sources (kube-state-metrics, cAdvisor, node-exporter). The focus here is on *how* to combine these metrics and understand their context to answer typical operational questions.

**Foundation: Labels are Key**
Remember, effective Kubernetes querying in PromQL relies heavily on the labels attached to metrics by exporters. These labels (e.g., `pod`, `namespace`, `node`, `container`, `deployment`, `service`, and especially `label_*` labels derived from Kubernetes object metadata) are essential for filtering, aggregation, and joining data.

**1. Pattern: Joining Resource Usage with Metadata (The Core Pattern)**

*   **Purpose:** To add context (like application name, environment, owner) to raw resource usage metrics (CPU, memory, network). This is fundamental for filtering and grouping meaningfully.
*   **Mechanism:** Use PromQL's binary operators (`*`, `/`, etc.) with `on(<matching_labels>)` and `group_left(<labels_to_add>)` or `group_right()` clauses to merge time series based on shared labels. Typically, you multiply a resource metric by a `kube-state-metrics` (KSM) label/info metric (which has a value of 1).
*   **Examples:**

    *   **CPU Usage per Deployment (Robust Method):**
        ```promql
        # Calculate CPU usage rate per pod, then join with ownership info layer by layer
        sum by (namespace, deployment) (
          rate(container_cpu_usage_seconds_total{container!=""}[5m]) # CPU rate per container/pod
          # Join Pod metric with Pod -> ReplicaSet owner info from KSM
          * on(pod, namespace) group_left(owner_name, owner_kind) kube_pod_owner{owner_kind="ReplicaSet"}
          # Join result with ReplicaSet -> Deployment owner info from KSM
          * on(owner_name, namespace) group_left(deployment) (kube_replicaset_owner{owner_kind="Deployment"} renaming (owner_name as replicaset))
        )
        ```
        *(Explanation: This joins pod usage to its ReplicaSet owner, then joins that to the ReplicaSet's Deployment owner, aggregating by the final deployment label.)*

    *   **CPU Usage per Deployment (Simpler, if `deployment` label exists on Pods):**
        ```promql
        # Assumes pods have a 'deployment' label propagated (common with Helm/Operators)
        sum by (namespace, label_deployment) (
          rate(container_cpu_usage_seconds_total{container!=""}[5m]) # CPU rate per container/pod
          # Join with KSM pod labels metric
          * on(pod, namespace) group_left(label_deployment) kube_pod_labels{label_deployment!=""}
        )
        ```

    *   **Memory Usage per Application Label (`app`):**
        ```promql
        sum by (namespace, label_app) (
          container_memory_working_set_bytes{container!=""} # Memory usage per container/pod
          # Join with KSM pod labels metric
          * on(pod, namespace) group_left(label_app) kube_pod_labels{label_app!=""}
        )
        ```

**2. Pattern: Calculating Resource Utilization (%)**

*   **Purpose:** To understand resource consumption relative to the requests or limits defined in Kubernetes specs.
*   **Mechanism:** Divide the resource usage metric by the corresponding request or limit metric obtained from KSM, typically using joins.
*   **Examples:**

    *   **CPU Utilization % vs. CPU Request:**
        ```promql
        # Pod CPU usage rate divided by CPU request defined in KSM
        (
          sum by (pod, namespace) (rate(container_cpu_usage_seconds_total{container!=""}[5m]))
        / on(pod, namespace) group_left() # Join on pod and namespace
          kube_pod_container_resource_requests{resource="cpu"}
        ) * 100
        ```
        *(Note: May need `ignoring(resource)` or further label manipulation depending on exact metric labels.)*

    *   **Memory Utilization % vs. Memory Limit:**
        ```promql
        # Pod memory working set divided by memory limit defined in KSM
        (
          sum by (pod, namespace) (container_memory_working_set_bytes{container!=""})
        / on(pod, namespace) group_left() # Join on pod and namespace
          kube_pod_container_resource_limits{resource="memory"}
        ) * 100
        ```
        *(Note: Handle cases where limits are not set - division by zero or missing metric). Use `> 0` on the limit metric if needed.*

**3. Pattern: Aggregation by Kubernetes Concepts**

*   **Purpose:** To summarize metrics across logical Kubernetes groupings (namespaces, deployments, nodes, applications).
*   **Mechanism:** Use aggregation operators (`sum`, `avg`, `max`, `min`, `count`, `topk`, `bottomk`) with a `by (<labels>)` or `without (<labels>)` clause. The labels used in `by()` often come from joins performed as per Pattern 1.
*   **Examples:**

    *   **Total CPU Cores used per Namespace:**
        ```promql
        sum by (namespace) (rate(container_cpu_usage_seconds_total{container!=""}[5m]))
        ```

    *   **Average Memory Usage (Working Set) per Deployment:**
        ```promql
        # Assumes label_deployment is available after joining (see Pattern 1)
        avg by (namespace, label_deployment) (
          container_memory_working_set_bytes{container!=""}
          * on(pod, namespace) group_left(label_deployment) kube_pod_labels{label_deployment!=""}
        )
        ```

    *   **Total Container Restarts per Node:**
        ```promql
        sum by (node) (rate(kube_pod_container_status_restarts_total[5m]))
        ```

**4. Pattern: Filtering by Kubernetes Status**

*   **Purpose:** To focus queries on objects in a specific state (e.g., only Running Pods, only Ready Nodes).
*   **Mechanism:** Multiply the target metric by a KSM status metric that equals `1` for the desired state.
*   **Examples:**

    *   **CPU Usage of only *Running* Pods:**
        ```promql
        # Multiply usage by 1 if pod is running, 0 otherwise
        rate(container_cpu_usage_seconds_total{container!=""}[5m])
        * on(pod, namespace) group_left() kube_pod_status_phase{phase="Running"} == 1
        ```

    *   **Count of *Not Ready* Nodes:**
        ```promql
        # Count nodes where the 'Ready' condition status is not 'true'
        sum(kube_node_status_condition{condition="Ready", status!="true"})
        # Alternative: count nodes where Ready condition is 'false' or 'unknown'
        # sum(kube_node_status_condition{condition="Ready", status=~"false|unknown"})
        ```

    *   **Network Traffic for *Ready* Pods:**
        ```promql
        rate(container_network_receive_bytes_total{container!=""}[5m])
        * on(pod, namespace) group_left() kube_pod_container_status_ready == 1
        ```

**5. Pattern: Node/Cluster Resource Calculations**

*   **Purpose:** To assess overall cluster resource availability and utilization.
*   **Mechanism:** Combine metrics from `node-exporter` (for node capacity/usage) often aggregated across the cluster.
*   **Examples:**

    *   **Cluster CPU Utilization %:**
        ```promql
        (
          # Sum of CPU time spent NOT being idle, across all nodes
          sum(rate(node_cpu_seconds_total{mode!="idle"}[5m]))
        /
          # Total CPU cores available (count modes per CPU, divide by # modes)
          count(node_cpu_seconds_total) by (cpu) / count(count(node_cpu_seconds_total) by (cpu, mode))
        ) * 100
        # Simpler approximation: sum(rate(node_cpu_seconds_total{mode!="idle"}[5m])) / sum(rate(node_cpu_seconds_total[5m])) * 100
        ```

    *   **Cluster Memory Availability %:**
        ```promql
        (
          sum(node_memory_MemAvailable_bytes) / sum(node_memory_MemTotal_bytes)
        ) * 100
        ```

    *   **Number of Schedulable Nodes:**
        ```promql
        # Count nodes that are Ready and do not have the 'unschedulable' spec field set
        sum(kube_node_status_condition{condition="Ready", status="true"} * on(node) group_left() (kube_node_spec_unschedulable == 0))
        ```

**6. Pattern: Top-K Resource Consumers**

*   **Purpose:** To quickly identify the pods, deployments, namespaces, etc., consuming the most resources.
*   **Mechanism:** Use the `topk(<k>, <expression>)` aggregation operator.
*   **Examples:**

    *   **Top 10 Pods by CPU Usage:**
        ```promql
        topk(10, sum by (pod, namespace) (rate(container_cpu_usage_seconds_total{container!=""}[5m])))
        ```

    *   **Top 5 Namespaces by Memory Usage:**
        ```promql
        topk(5, sum by (namespace) (container_memory_working_set_bytes{container!=""}))
        ```

---

## Understanding Context and Units

Correctly interpreting Kubernetes metrics requires understanding their units and types:

*   **CPU:**
    *   **Kubernetes Definition:** Requests/limits are specified in "cores" or "millicores/millicpu" (e.g., `1`, `0.5`, `500m`). `1` core = 1 AWS vCPU = 1 GCP Core = 1 Azure vCore = 1 Hyperthread on a bare-metal CPU.
    *   **Prometheus Metric:** `container_cpu_usage_seconds_total` (and `node_cpu_seconds_total`) are **Counters** measuring cumulative CPU time in seconds.
    *   **Querying:** Always apply `rate()` or `irate()` to CPU counters. The result `rate(container_cpu_usage_seconds_total[1m])` gives the average CPU core usage over the last minute. A value of `1.5` means the container averaged 1.5 CPU cores during that minute.

*   **Memory:**
    *   **Kubernetes Definition:** Requests/limits are specified in bytes, often using power-of-two suffixes (Ki, Mi, Gi) or power-of-ten suffixes (K, M, G).
    *   **Prometheus Metric:** `container_memory_working_set_bytes`, `container_memory_usage_bytes`, `node_memory_MemAvailable_bytes`, etc., are **Gauges** representing the current value in bytes.
    *   **Querying:** Use gauge values directly. `container_memory_working_set_bytes` is often preferred for utilization calculations against limits as it represents memory less likely to be reclaimed under pressure (excluding inactive file cache).

*   **Rates vs. Gauges:**
    *   **Counters (`_total` suffix):** Always increasing values (e.g., requests served, bytes transmitted). Use `rate()`, `irate()`, or `increase()` to see the per-second rate of change or the total increase over a time window.
    *   **Gauges:** Represent a value that can go up or down (e.g., current memory usage, temperature, queue length). Use their current value directly or apply functions like `avg_over_time()`, `max_over_time()`.

*   **Interpreting Statuses:**
    *   `kube_pod_status_phase`: `Pending` (Scheduled but not running), `Running` (Containers running), `Succeeded` (Completed successfully), `Failed` (Terminated with error), `Unknown` (Node communication lost).
    *   `kube_node_status_condition`: `Ready` (Node healthy and accepting pods), `MemoryPressure` (Node memory is low), `DiskPressure` (Node disk space is low), `PIDPressure` (Too many processes on node), `NetworkUnavailable` (Node network not configured correctly).
