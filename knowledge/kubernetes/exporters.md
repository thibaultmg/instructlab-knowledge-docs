## Understanding Kubernetes Monitoring Data Sources (Exporters)

**Goal:** This document details the essential Prometheus exporters and built-in metric sources within a Kubernetes environment. It explains their specific roles, the types of metrics they expose, and how these metrics are labeled. This knowledge is critical for an LLM to formulate effective PromQL queries for monitoring Kubernetes cluster health and application performance.

**Core Concept: Exporters and Labels**

In a Prometheus-based monitoring setup for Kubernetes, dedicated processes called **exporters** run within the cluster. Their job is to collect data about specific components (like Nodes, Pods, K8s objects) and expose this data as metrics in a format Prometheus can scrape (usually via an HTTP `/metrics` endpoint).

The **most critical aspect** is that these exporters enrich the metrics with **labels** derived from Kubernetes object metadata (names, namespaces, Kubernetes labels, owner references). These labels are the key to filtering, aggregating, and correlating data across the cluster using PromQL.

**1. `kube-state-metrics` (KSM)**

*   **Purpose:** Provides metrics *about the state of Kubernetes objects themselves*. It listens to the Kubernetes API server and generates metrics based on the status, configuration, and metadata of objects like Pods, Deployments, Nodes, Services, etc. It does *not* provide resource usage metrics (CPU/memory).
*   **How it Helps:** KSM allows you to query the state and configuration of your cluster using PromQL. It's essential for understanding object counts, desired vs. actual states, object statuses, resource requests/limits, and crucially, **linking resource metrics to Kubernetes metadata**.
*   **Key Metrics & Labels:**
    *   **Object Labels:** `kube_<object>_labels` (e.g., `kube_pod_labels`, `kube_deployment_labels`, `kube_node_labels`, `kube_service_labels`).
        *   Type: Gauge (always value 1).
        *   Role: **CRITICAL BRIDGE.** These metrics carry the object's *Kubernetes Labels* (defined in YAML) as *Prometheus Labels* (often prefixed like `label_app="..."`, `label_version="..."`). Used via `group_left()` joins to add application/environment context to other metrics.
    *   **Object Info:** `kube_<object>_info` (e.g., `kube_pod_info`, `kube_node_info`).
        *   Type: Gauge (always value 1).
        *   Role: Provides non-label metadata as labels (e.g., `node` name for a Pod via `kube_pod_info{node="..."}`, `kubelet_version` via `kube_node_info`). Useful for lookups.
    *   **Object Status:**
        *   `kube_pod_status_phase`: Gauge (value 1) indicating Pod phase (`{phase="Running"}`, `{phase="Pending"}`, `{phase="Succeeded"}`, `{phase="Failed"}`, `{phase="Unknown"}`).
        *   `kube_deployment_status_replicas_available`, `_unavailable`, `_updated`: Gauges tracking Deployment rollout status.
        *   `kube_node_status_condition`: Gauge (value 1 or 0) indicating Node conditions (`{condition="Ready", status="true"}`, `{condition="MemoryPressure", status="false"}`).
        *   `kube_pod_container_status_ready`: Gauge (value 1) if a container is ready.
        *   `kube_pod_container_status_restarts_total`: Counter tracking container restarts (often indicates crash loops).
    *   **Object Configuration (Spec):**
        *   `kube_deployment_spec_replicas`: Gauge showing the desired replica count for a Deployment.
        *   `kube_pod_container_resource_requests`, `kube_pod_container_resource_limits`: Gauges showing CPU (`{resource="cpu"}`) and memory (`{resource="memory"}`) requests/limits defined for containers. Essential for calculating utilization percentages.
    *   **Ownership:** `kube_<object>_owner` (e.g., `kube_pod_owner`, `kube_replicaset_owner`).
        *   Type: Gauge (always value 1).
        *   Role: Shows ownership relationships (`{owner_kind="ReplicaSet", owner_name="..."}`). Used in joins to link Pods to Deployments/StatefulSets etc.

**2. `kubelet` / `cAdvisor` (Container/Pod Resources)**

*   **Purpose:** Provides resource usage metrics for containers running on each Node. The `kubelet` (the agent running on every Node) embeds `cAdvisor` functionality to collect and expose these metrics.
*   **How it Helps:** This is the primary source for understanding how much CPU, memory, network, and disk I/O individual application containers and Pods are consuming.
*   **Key Metrics & Labels:** Metrics are inherently labeled with `pod`, `namespace`, `container` (name within the Pod), and `node`.
    *   **CPU:** `container_cpu_usage_seconds_total`
        *   Type: Counter. Represents total CPU time consumed in seconds.
        *   Usage: Needs `rate()` or `irate()` to get average core usage over time.
    *   **Memory:** `container_memory_working_set_bytes`
        *   Type: Gauge. Represents memory actively being used and not easily reclaimable. Often the best measure for memory usage relative to limits.
        *   Others: `container_memory_usage_bytes` (total usage including cache), `container_memory_rss` (Resident Set Size).
    *   **Network:** `container_network_receive_bytes_total`, `container_network_transmit_bytes_total`
        *   Type: Counters. Total bytes received/transmitted by the container's network interface.
        *   Usage: Need `rate()` to get bandwidth usage. Often filtered by `interface`.
    *   **Filesystem:**
        *   `container_fs_usage_bytes`: Gauge. Bytes used by the container's writable layer.
        *   `container_fs_reads_bytes_total`, `container_fs_writes_bytes_total`: Counters. Bytes read/written. Need `rate()` for I/O throughput. Often filtered by `device`.

**3. `node-exporter` (Node/OS Resources)**

*   **Purpose:** Provides detailed hardware and operating system level metrics for each Node in the cluster. It runs as a DaemonSet (one instance per Node).
*   **How it Helps:** Essential for understanding the overall health and resource pressure on the cluster's worker machines. Helps diagnose if Pod performance issues are due to Node-level bottlenecks.
*   **Key Metrics & Labels:** Metrics are labeled with `node` and often specific identifiers like `device`, `mountpoint`, `cpu` (core number), `mode` (for CPU state).
    *   **CPU:** `node_cpu_seconds_total`
        *   Type: Counter. Seconds spent by the CPU in various modes (`idle`, `user`, `system`, `iowait`, etc.).
        *   Usage: Use `rate()` and filter `mode!="idle"` then aggregate by `node` to get total node CPU utilization.
    *   **Memory:** `node_memory_MemTotal_bytes`, `node_memory_MemFree_bytes`, `node_memory_MemAvailable_bytes`, `node_memory_Buffers_bytes`, `node_memory_Cached_bytes`
        *   Type: Gauges. Represent different aspects of system memory. `MemAvailable_bytes` is often the best indicator of memory available for new processes.
    *   **Disk I/O:** `node_disk_read_bytes_total`, `node_disk_written_bytes_total`, `node_disk_io_time_seconds_total`
        *   Type: Counters. Need `rate()` for throughput and utilization. Filter/aggregate by `device`.
    *   **Network:** `node_network_receive_bytes_total`, `node_network_transmit_bytes_total`
        *   Type: Counters. Need `rate()` for bandwidth. Filter/aggregate by `device` (network interface).
    *   **Filesystem:** `node_filesystem_avail_bytes`, `node_filesystem_size_bytes`, `node_filesystem_files`
        *   Type: Gauges. Available/total size and inodes. Filter/aggregate by `mountpoint` and `fstype`.
    *   **Load Average:** `node_load1`, `node_load5`, `node_load15` (Gauges).
    *   **System:** `node_uname_info` (Info Gauge), `node_time_seconds` (Gauge).

**4. Control Plane Components (Direct Metrics)**

*   **Purpose:** `etcd`, `kube-apiserver`, `kube-scheduler`, `kube-controller-manager` expose their own internal performance and health metrics directly.
*   **How it Helps:** Critical for diagnosing issues with the cluster's core functionality (API responsiveness, scheduling latency, etcd health).
*   **Key Metrics:** (As detailed in the previous Control Plane document)
    *   `etcd_...`: Leader status, commit/fsync latencies, peer latency.
    *   `apiserver_...`: Request latency, request rate (by verb/code), inflight requests, etcd request latency (from apiserver's perspective).
    *   `scheduler_...`: Scheduling latency (e2e, algorithm, binding), pending pods, schedule attempts (by result).
    *   `workqueue_...` (from controller-manager): Queue depth, latency, work duration, often labeled by controller name.

**5. Other Common Exporters**

*   **Application Exporters:** Applications themselves can (and should) expose metrics about their specific business logic (e.g., queue depths, items processed, request latency per endpoint). These are scraped directly by Prometheus.
*   **Ingress Controller Exporters:** Nginx Ingress, Traefik, etc., expose detailed metrics about incoming traffic, routing, latency, response codes, often labeled by `ingress`, `service`, `path`, `host`.
*   **Database/Cache Exporters:** Exporters exist for common databases (Postgres, MySQL) and caches (Redis) providing internal metrics.

**Connecting the Dots: The Power of Labels**

No single exporter tells the whole story. The real power comes from combining metrics using PromQL, leveraging the shared labels:

*   Use `kube_pod_labels` (from KSM) to filter `container_cpu_usage_seconds_total` (from kubelet/cAdvisor) by `label_app` or `label_version`.
*   Use `kube_pod_info` (from KSM) to link `container_memory_working_set_bytes` (from kubelet/cAdvisor) to `node_memory_MemAvailable_bytes` (from node-exporter) via the `node` label to see if Pod memory usage correlates with Node memory pressure.
*   Use `kube_pod_owner` (from KSM) to aggregate `container_network_receive_bytes_total` (from kubelet/cAdvisor) by `deployment` or `statefulset`.

**Understanding Context and Units:**

*   **CPU:** K8s requests/limits are in "cores". Usage metrics derived from `container_cpu_usage_seconds_total` (via `rate()`) are also in "cores" (average over time).
*   **Memory:** Usually in bytes. Compare `container_memory_working_set_bytes` against `kube_pod_container_resource_limits{resource="memory"}`.
*   **Rates vs. Gauges:** Remember Counters (like `_total` metrics) need `rate()` or `increase()` to be meaningful over time, while Gauges represent current values.

By knowing which exporter provides which metric and, crucially, what labels are attached, an LLM can translate natural language questions about Kubernetes performance ("Which app is using the most CPU?", "Is the production namespace running out of memory?", "Are nodes under pressure?") into precise and effective PromQL queries.
