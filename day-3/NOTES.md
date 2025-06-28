# Day 3: Deep Dive into Prometheus Architecture and PromQL

---

## 🎯 Learning Objectives
- Understand Prometheus architecture in detail
- Learn about key components: Node Exporter and Kube State Metrics
- Master PromQL (Prometheus Query Language) basics
- Explore Grafana for visualization
- Practice querying metrics and creating dashboards

---

## 🏗️ Prometheus Architecture Overview

### Core Components
1. **Prometheus Server**: The main component that scrapes and stores metrics
2. **Time Series Database**: Stores metrics as key-value pairs with timestamps
3. **HTTP Server**: Provides API for querying data using PromQL
4. **Alert Manager**: Handles alerts when metrics exceed thresholds

### Data Flow
```
Sources (Node Exporter, Kube State Metrics) → Prometheus Server → Time Series DB → PromQL Queries → Grafana/Alerts
```
---

## 📊 Key Metric Sources

### 1. Node Exporter
- **Purpose**: Collects infrastructure-related information from Kubernetes nodes
- **Deployment**: Runs as a DaemonSet on each node
- **Metrics Collected**:
  - CPU utilization and statistics
  - Memory usage and statistics
  - Disk I/O and filesystem metrics
  - Network statistics
  - System information from `/proc` and `/sys`
- **Endpoint**: Exposes metrics on port `9100` at `/metrics`

### 2. Kube State Metrics
- **Purpose**: Collects Kubernetes cluster state information
- **Deployment**: Runs as a single pod, communicates with API server
- **Metrics Collected**:
  - Pod status and lifecycle information
  - Deployment status and replica counts
  - Service and endpoint information
  - ConfigMap and Secret counts
  - Container restart counts
  - Resource quotas and limits
- **Endpoint**: Exposes metrics on port `8080` at `/metrics`

### 3. Custom Application Metrics
- **Purpose**: Application-specific metrics defined by developers
- **Examples**:
  - HTTP request duration and count
  - User registration counts
  - Business logic metrics
  - Error rates and response codes
- **Implementation**: Added through metric instrumentation in application code

---

## 📊 Metrics in Prometheus

### What are Metrics?
- Metrics are the core data objects representing measurements from monitored systems
- Provide insights into **system performance, health, and behavior**
- Stored as time series data with timestamps

### 🏷️ Labels
- **Definition**: Key-value pairs that differentiate dimensions of a metric
- **Purpose**: Allow filtering and grouping of metrics
- **Example**:
  ```bash
  container_cpu_usage_seconds_total{namespace="kube-system", endpoint="https-metrics"}
  ```
  - `container_cpu_usage_seconds_total` = metric name
  - `{namespace="kube-system", endpoint="https-metrics"}` = labels

---

## 🔍 PromQL (Prometheus Query Language)

### What is PromQL?
- Powerful and flexible query language for Prometheus data
- Allows retrieval and manipulation of time series data
- Supports mathematical operations and data aggregation

### 🔑 Key Features
- **Time Series Selection**: Filter and retrieve specific metrics
- **Mathematical Operations**: Perform calculations on metrics
- **Aggregation**: Combine data across multiple time series
- **Functions**: Wide range of built-in functions for data analysis

### 💡 Basic PromQL Examples

#### Simple Metric Selection
```bash
# Return all time series for a specific metric
container_cpu_usage_seconds_total
```

#### Label Filtering
```bash
# Filter by specific labels
container_cpu_usage_seconds_total{namespace="kube-system",pod=~"kube-proxy.*"}
```

#### Range Queries
```bash
# Get 5 minutes of historical data
container_cpu_usage_seconds_total{namespace="kube-system",pod=~"kube-proxy.*"}[5m]
```

#### Container Restart Tracking
```bash
# Query container restarts (demonstrated in video)
kube_pod_container_status_restarts_total{namespace="default"}
```

---

## ⚙️ Aggregation & Functions in PromQL

### Common Aggregation Functions

#### Sum - Total CPU Usage
```bash
sum(rate(node_cpu_seconds_total[5m]))
```
- Aggregates CPU usage across all nodes

#### Average - Memory Usage per Namespace
```bash
avg(container_memory_usage_bytes) by (namespace)
```
- Provides average memory usage grouped by namespace

### Important PromQL Functions

#### rate() Function
- Calculates per-second average rate of increase
```bash
rate(container_cpu_usage_seconds_total[5m])
```
- Shows CPU usage rate over 5 minutes

#### increase() Function
- Returns total increase in counter over time range
```bash
increase(kube_pod_container_status_restarts_total[1h])
```
- Shows total container restarts in the last hour

#### histogram_quantile() Function
- Calculates quantiles (percentiles) from histogram data
```bash
histogram_quantile(0.95, sum(rate(apiserver_request_duration_seconds_bucket[5m])) by (le))
```
- Shows 95th percentile of Kubernetes API request durations

---

## 📈 Grafana Integration

### What is Grafana?
- **Purpose**: Visualization and dashboard platform (not a monitoring tool)
- **Independence**: Supports multiple data sources (Prometheus, Nagios, Graphite, InfluxDB)
- **Features**:
  - Custom dashboard creation
  - Authentication and authorization
  - User permission management
  - SSO/IAM integration

### Using Grafana with Prometheus
1. **Data Source Configuration**: Add Prometheus as data source
2. **Pre-built Dashboards**: Use existing dashboards for common metrics
3. **Custom Dashboards**: Create custom visualizations using PromQL queries
4. **Time Range Selection**: Filter data by time periods
5. **Namespace Filtering**: View metrics for specific namespaces

### Dashboard Examples from Video
- **Node Resource Utilization**: CPU and memory usage per node
- **Pod Resource Utilization**: Resource usage per pod/namespace
- **Container Restart Tracking**: Monitor pod crashes and restart patterns
- **Persistent Volume Monitoring**: Storage-related metrics

---

## 🔧 Practical Implementation

### Accessing Services (from video demonstration)
```bash
# Port forwarding for local access (works across all K8s clusters)
kubectl port-forward svc/prometheus-server 9090:80
kubectl port-forward svc/grafana 3000:80
kubectl port-forward svc/alertmanager 9093:80
```

### Verifying Metric Collection
```bash
# Check Node Exporter metrics
curl <node-exporter-ip>:9100/metrics

# Check Kube State Metrics
curl <kube-state-metrics-ip>:8080/metrics
```

### Testing with Crash Loop Example
```bash
# Create a pod that crashes (for testing)
kubectl run busybox-crash --restart=Never --rm -it --image=busybox -- /bin/sh -c "exit 1"
```

---

## 📚 Important Metrics to Know

### Infrastructure Metrics
- `node_cpu_seconds_total` - CPU usage statistics
- `node_memory_MemTotal_bytes` - Total memory
- `node_filesystem_size_bytes` - Filesystem size

### Kubernetes Metrics
- `kube_pod_container_status_restarts_total` - Container restarts
- `kube_deployment_status_replicas` - Deployment replica status
- `kube_pod_status_phase` - Pod lifecycle status
- `kube_configmap_info` - ConfigMap information
- `kube_secret_info` - Secret information

### Application Metrics (Custom)
- HTTP request counts and durations
- User activity metrics
- Business logic counters
- Error rates and response codes

---

## 🎯 Key Takeaways

1. **Architecture Understanding**: Prometheus scrapes metrics from various sources and stores them in a time series database
2. **Essential Sources**: Node Exporter and Kube State Metrics are fundamental pillars
3. **Query Power**: PromQL provides flexible querying capabilities for analysis
4. **Visualization**: Grafana enhances Prometheus with better dashboards and visualization
5. **Real-time Monitoring**: The combination enables effective real-time system monitoring

