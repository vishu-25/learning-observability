# Day 6: Distributed Tracing with Jaeger

---

## 🔍 Introduction to Distributed Tracing

### What is Distributed Tracing?
**Distributed tracing** is the third pillar of observability that tracks requests as they flow through multiple services in a distributed system. It answers the question: **"How did the request flow through my system?"**

### Real-World Analogy: Travel Itinerary
Think of distributed tracing like planning a trip from **Hyderabad to Boston**:

```
Hyderabad → Dubai (2 hours) → London (8 hours) → Boston (7 hours)
Total Journey: 17 hours
```

If your journey takes 19 hours instead of 17, tracing helps you identify:
- ✅ **Which leg** caused the delay (e.g., wrong cab to airport = 2 hour delay)
- ✅ **How long** each segment actually took
- ✅ **Root cause** of the problem

### Microservices Context
In a microservices architecture, a single user request might travel through:

```
User Request → Login Service → Service A → Service B → Payment Service → Database
```

**Without tracing**: When something is slow, you don't know which service is the bottleneck.
**With tracing**: You can see exactly where time is being spent and optimize accordingly.

---

## 🏗️ The Three Pillars of Observability (Complete Picture)

| Pillar | Purpose | Question Answered | Example |
|--------|---------|-------------------|---------|
| 📊 **Metrics** | Quantitative measurements | *How much?* | CPU usage, request count, error rate |
| 📝 **Logs** | Event records | *What happened?* | "Database connection failed" |
| 🔍 **Traces** | Request journey | *How did it flow?* | Request path through 5 services, each taking X ms |

**Together, they provide complete observability** of your distributed systems!

---

## 🕵️‍♂️ What is Jaeger?

### Definition
**Jaeger** is an open-source, end-to-end distributed tracing system used for monitoring and troubleshooting microservices-based architectures.

### Why Use Jaeger?
- 🐢 **Identify bottlenecks**: See where your application spends most of its time
- 🔍 **Find root causes**: Trace errors back to their source  
- ⚡ **Optimize performance**: Understand and improve service latency
- 🎯 **Debug complex flows**: Understand request paths through multiple services

### Key Benefits
- **Lightweight and simple**: Minimalistic UI, easy to set up
- **Open source**: Free to use, large community support
- **Vendor neutral**: Works with OpenTelemetry standard
- **Scalable**: Suitable for startups to large enterprises

---

## 📚 Core Concepts of Jaeger

### 1. 🛤️ **Trace**
A **trace** represents the complete journey of a request through your system.

**Example**: User login request
```
Trace ID: abc123
Duration: 245ms
Services: 4
Spans: 12
```

### 2. 📏 **Span**
A **span** is a single operation within a trace (like a step in your journey).

**Example spans**:
- HTTP request to authentication service (50ms)
- Database query for user credentials (120ms)
- JWT token generation (15ms)
- Response formatting (10ms)

### 3. 🏷️ **Tags**
**Tags** are key-value pairs that provide additional context.

**Example tags**:
```
http.method: "POST"
http.status_code: 200
user.id: "12345"
service.version: "v1.2.3"
```

### 4. 📝 **Logs**
**Logs** within spans capture events and additional details.

**Example logs**:
```
timestamp: 2024-01-15T10:30:00Z
message: "User authentication successful"
level: "INFO"
```

### 5. 🔗 **Context Propagation**
**Context propagation** ensures trace information flows between services.

**How it works**:
1. Service A adds trace headers to outgoing requests
2. Service B receives headers and continues the trace
3. Service C receives headers and adds its own spans
4. Complete trace is visible in Jaeger UI

---

## 🏗️ Jaeger Architecture

### High-Level Architecture
```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│   Application   │    │   Jaeger Agent   │    │ Jaeger Collector│
│   (Instrumented)│───▶│   (DaemonSet)    │───▶│   (Processor)   │
│                 │    │                  │    │                 │
│ • Service A     │    │ • Trace Collection│    │ • Validation    │
│ • Service B     │    │ • Batching       │    │ • Processing    │
│ • Service C     │    │ • Forwarding     │    │ • Storage       │
└─────────────────┘    └──────────────────┘    └─────────────────┘
                                                         │
                                                         ▼
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│   Jaeger UI     │    │  Jaeger Query    │    │  Elasticsearch  │
│ (Visualization) │◀───│   (API Server)   │◀───│   (Storage)     │
│                 │    │                  │    │                 │
│ • Search Traces │    │ • Query API      │    │ • Trace Storage │
│ • View Details  │    │ • Data Retrieval │    │ • Indexing      │
│ • Performance   │    │ • Aggregation    │    │ • Persistence   │
└─────────────────┘    └──────────────────┘    └─────────────────┘
```


![Project Architecture](images/architecture.gif)

### Component Details

#### 🤖 **Jaeger Agent**
- **Purpose**: Collects traces from applications
- **Deployment**: DaemonSet on every Kubernetes node
- **Function**: Receives traces via UDP, batches them, forwards to collector

#### 🔄 **Jaeger Collector**
- **Purpose**: Receives, validates, and processes traces
- **Function**: 
  - Validates trace data
  - Applies sampling rules
  - Stores traces in database
  - Provides APIs for querying

#### 🗄️ **Storage Backend**
- **Options**: Elasticsearch, Cassandra, Kafka
- **Recommendation**: Elasticsearch (fast, scalable)
- **Function**: Persistent storage for trace data

#### 🔍 **Jaeger Query**
- **Purpose**: API server for retrieving traces
- **Function**: 
  - Queries storage backend
  - Provides REST API
  - Serves Jaeger UI

#### 🖥️ **Jaeger UI**
- **Purpose**: Web interface for viewing traces
- **Features**:
  - Search and filter traces
  - Visualize request flow
  - Analyze performance metrics
  - Compare traces

---

## 🛠️ Implementation: Two-Part Process

### Part 1: Developer Responsibility - Instrumentation

#### Using OpenTelemetry (Recommended)
OpenTelemetry is the **vendor-neutral standard** for instrumentation:

**Benefits**:
- ✅ **Vendor neutral**: Easy migration between tracing tools
- ✅ **Language support**: Available for all major programming languages
- ✅ **Community driven**: Industry standard
- ✅ **Future proof**: Backed by CNCF

#### Node.js Example with OpenTelemetry
```javascript
// tracing.js
const { NodeSDK } = require('@opentelemetry/sdk-node');
const { getNodeAutoInstrumentations } = require('@opentelemetry/auto-instrumentations-node');
const { JaegerExporter } = require('@opentelemetry/exporter-jaeger');

const jaegerExporter = new JaegerExporter({
  endpoint: 'http://jaeger-collector:14268/api/traces',
});

const sdk = new NodeSDK({
  traceExporter: jaegerExporter,
  instrumentations: [getNodeAutoInstrumentations()],
  serviceName: 'service-a',
  serviceVersion: '1.0.0',
});

sdk.start();
```

#### Express.js Application Example
```javascript
// index.js
require('./tracing'); // Import tracing first!

const express = require('express');
const app = express();

app.get('/api/users/:id', async (req, res) => {
  // OpenTelemetry automatically creates spans for:
  // - HTTP request/response
  // - Database queries
  // - External API calls
  
  const userId = req.params.id;
  const user = await getUserFromDatabase(userId);
  const profile = await getProfileFromAPI(userId);
  
  res.json({ user, profile });
});
```

### Part 2: DevOps/SRE Responsibility - Infrastructure

#### Responsibilities
- 🏗️ **Set up Kubernetes cluster**
- 📦 **Install and configure Jaeger**
- 🔧 **Configure storage backend**
- 🔐 **Set up authentication and RBAC**
- 🌐 **Expose Jaeger UI**
- 📊 **Monitor Jaeger infrastructure**

---

## 🚀 Step-by-Step Jaeger Deployment

### Prerequisites
- Kubernetes cluster (EKS recommended)
- Elasticsearch already deployed (from Day 5)
- Helm installed
- kubectl configured

### Step 1: Export Elasticsearch CA Certificate

```bash
# Extract CA certificate from Elasticsearch
kubectl get secret elasticsearch-master-certs -n logging \
  -o jsonpath='{.data.ca\.crt}' | base64 --decode > ca-cert.pem
```

**Why this step?** Jaeger needs to communicate securely with Elasticsearch using TLS.

### Step 2: Create Tracing Namespace

```bash
# Create dedicated namespace for tracing components
kubectl create namespace tracing
```

**Best Practice**: Separate namespaces for different observability components (monitoring, logging, tracing).

### Step 3: Create ConfigMap for TLS Certificate

```bash
# Create ConfigMap containing CA certificate
kubectl create configmap jaeger-tls --from-file=ca-cert.pem -n tracing
```

### Step 4: Create Secret for Elasticsearch TLS

```bash
# Create Secret for Elasticsearch TLS communication
kubectl create secret generic es-tls-secret --from-file=ca-cert.pem -n tracing
```

### Step 5: Add Jaeger Helm Repository

```bash
# Add Jaeger Helm repository
helm repo add jaegertracing https://jaegertracing.github.io/helm-charts
helm repo update
```

### Step 6: Configure Jaeger Values

Edit `jaeger-values.yaml` with your Elasticsearch credentials:

```yaml
# jaeger-values.yaml
storage:
  type: elasticsearch
  elasticsearch:
    host: elasticsearch-master.logging.svc.cluster.local
    port: 9200
    scheme: https
    username: elastic
    password: "YOUR_ELASTICSEARCH_PASSWORD_HERE"  # Update this!
    tls:
      ca: /tls/ca-cert.pem
      
query:
  service:
    type: LoadBalancer  # or NodePort/ClusterIP
    
collector:
  service:
    type: ClusterIP
    
agent:
  daemonset:
    useHostNetwork: true
```

**⚠️ Critical**: Update the password with the one from Day 5 Elasticsearch setup!

### Step 7: Install Jaeger

```bash
# Install Jaeger with custom configuration
helm install jaeger jaegertracing/jaeger -n tracing --values jaeger-values.yaml
```

### Step 8: Verify Installation

```bash
# Check all tracing components
kubectl get pods -n tracing

# Expected output:
# NAME                                READY   STATUS    RESTARTS   AGE
# jaeger-agent-xxxxx                  1/1     Running   0          2m
# jaeger-collector-xxxxxxxxx-xxxxx    1/1     Running   0          2m
# jaeger-query-xxxxxxxxx-xxxxx        1/1     Running   0          2m
```

### Step 9: Access Jaeger UI

#### Option 1: Port Forward (Development)
```bash
# Forward port to access locally
kubectl port-forward svc/jaeger-query 8080:80 -n tracing --address 0.0.0.0
```

**Access**: `http://your-server-ip:8080`

#### Option 2: LoadBalancer (Production)
```bash
# Get LoadBalancer URL
kubectl get svc -n tracing jaeger-query
```

**Access**: `http://LOAD_BALANCER_DNS:80`

---

## 🧪 Testing Distributed Tracing

### Step 1: Deploy Instrumented Applications

```bash
# Deploy sample applications from day-4 (already instrumented)
kubectl apply -k ../day-4/kubernetes-manifest/
```

### Step 2: Generate Trace Data

```bash
# Get LoadBalancer DNS
kubectl get svc -n dev

# Generate traces by calling endpoints
curl http://your-app-loadbalancer/healthy
curl http://your-app-loadbalancer/call-service-b
curl http://your-app-loadbalancer/serverError
```

### Step 3: View Traces in Jaeger UI

1. **Access Jaeger UI** at your configured URL
2. **Select Service**: Choose "service-a" from dropdown
3. **Click "Find Traces"**: View all traces
4. **Analyze Results**: Click on individual traces for details

---

## 📊 Using Jaeger UI for Analysis

### Basic Navigation

#### 1. **Search Interface**
```
Service: [service-a ▼]
Operation: [All ▼]
Tags: [key:value]
Lookback: [1h ▼]
Min Duration: [empty]
Max Duration: [empty]
Limit Results: [20 ▼]
```

#### 2. **Trace List View**
```
Trace ID: abc123def456
Duration: 245.67ms
Services: 2
Spans: 8
Start Time: 2024-01-15 10:30:00
```

#### 3. **Detailed Trace View**
```
service-a: GET /call-service-b                    [245.67ms]
├── middleware: authentication                     [12.34ms]
├── middleware: logging                           [5.67ms]
├── handler: call-service-b                       [215.23ms]
│   └── http-client: POST service-b/hello         [198.45ms]
│       └── service-b: POST /hello                [187.12ms]
│           ├── middleware: cors                  [3.45ms]
│           ├── middleware: logging               [2.34ms]
│           └── handler: hello                    [178.23ms]
└── response: formatting                          [8.43ms]
```

### Advanced Analysis

#### 1. **Identifying Bottlenecks**
Look for:
- 🔴 **Longest spans**: Where is most time spent?
- 🟡 **Unexpected delays**: Spans taking longer than expected
- 🟢 **Error spans**: Spans with error tags

#### 2. **Performance Optimization**
```
Before optimization:
service-a → service-b: 245ms
├── Network latency: 20ms
├── Database query: 180ms  ← BOTTLENECK!
└── Processing: 45ms

After optimization:
service-a → service-b: 85ms
├── Network latency: 20ms
├── Database query: 25ms   ← OPTIMIZED!
└── Processing: 40ms
```

#### 3. **Error Tracing**
```
Trace with Error:
service-a: GET /payment                          [ERROR]
├── validation: input-check                      [OK]
├── auth: verify-token                          [OK]
├── payment: process-payment                    [ERROR]
│   └── database: insert-transaction           [TIMEOUT]
└── response: error-handling                    [OK]
```

---

## 🔍 Troubleshooting Common Issues

### 1. **Jaeger Query Pod in CrashLoopBackOff**

#### Check Pod Status
```bash
kubectl get pods -n tracing
kubectl describe pod jaeger-query-xxxxx -n tracing
```

#### Common Causes:
- **Certificate issues**: Incorrect CA certificate
- **Authentication failure**: Wrong Elasticsearch credentials
- **Network connectivity**: Can't reach Elasticsearch

#### Solutions:
```bash
# Recreate certificates
kubectl delete configmap jaeger-tls -n tracing
kubectl delete secret es-tls-secret -n tracing

# Re-export certificate
kubectl get secret elasticsearch-master-certs -n logging \
  -o jsonpath='{.data.ca\.crt}' | base64 --decode > ca-cert.pem

# Recreate ConfigMap and Secret
kubectl create configmap jaeger-tls --from-file=ca-cert.pem -n tracing
kubectl create secret generic es-tls-secret --from-file=ca-cert.pem -n tracing

# Restart Jaeger
helm upgrade jaeger jaegertracing/jaeger -n tracing --values jaeger-values.yaml
```

### 2. **No Traces Appearing in UI**

#### Check Application Instrumentation
```bash
# Verify applications are instrumented
kubectl logs -f deployment/service-a -n dev

# Look for tracing initialization logs like:
# "OpenTelemetry initialized"
# "Jaeger tracer started"
```

#### Check Jaeger Agent
```bash
# Verify agent is receiving traces
kubectl logs -f daemonset/jaeger-agent -n tracing

# Look for messages like:
# "Received spans: 10"
# "Forwarded to collector"
```

#### Check Jaeger Collector
```bash
# Verify collector is processing traces
kubectl logs -f deployment/jaeger-collector -n tracing

# Look for messages like:
# "Spans processed: 25"
# "Stored in Elasticsearch"
```

### 3. **Port Forward Issues**

#### EC2 Instance Access
```bash
# Use --address flag for external access
kubectl port-forward svc/jaeger-query 8080:80 -n tracing --address 0.0.0.0
```

#### Firewall Rules
```bash
# Ensure port 8080 is open in security groups
# AWS Console → EC2 → Security Groups → Inbound Rules
```

---

## 📈 Production Considerations

### 1. **Sampling Strategy**

#### Why Sampling?
- **Performance**: Tracing every request can impact performance
- **Storage**: Reduces storage costs
- **Focus**: Concentrate on important traces

#### Sampling Configuration
```yaml
# jaeger-values.yaml
collector:
  config:
    sampling:
      default_strategy:
        type: probabilistic
        param: 0.1  # Sample 10% of traces
      per_service_strategies:
        - service: "critical-service"
          type: probabilistic
          param: 1.0  # Sample 100% of critical service
        - service: "background-job"
          type: probabilistic
          param: 0.01  # Sample 1% of background jobs
```

### 2. **Storage Management**

#### Elasticsearch Index Lifecycle
```bash
# Configure index lifecycle management
curl -X PUT "elasticsearch-master.logging.svc.cluster.local:9200/_ilm/policy/jaeger-policy" \
  -H "Content-Type: application/json" \
  -d '{
    "policy": {
      "phases": {
        "hot": {
          "actions": {
            "rollover": {
              "max_size": "50gb",
              "max_age": "1d"
            }
          }
        },
        "delete": {
          "min_age": "7d",
          "actions": {
            "delete": {}
          }
        }
      }
    }
  }'
```

### 3. **High Availability Setup**

#### Multiple Collectors
```yaml
# jaeger-values.yaml
collector:
  replicaCount: 3
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 500m
      memory: 512Mi
```

#### Load Balancer Configuration
```yaml
collector:
  service:
    type: LoadBalancer
    annotations:
      service.beta.kubernetes.io/aws-load-balancer-type: nlb
```

### 4. **Security Best Practices**

#### RBAC Configuration
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: tracing
  name: jaeger-reader
rules:
- apiGroups: [""]
  resources: ["pods", "services"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: jaeger-reader-binding
  namespace: tracing
subjects:
- kind: User
  name: jaeger-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: jaeger-reader
  apiGroup: rbac.authorization.k8s.io
```

#### Network Policies
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: jaeger-network-policy
  namespace: tracing
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: tracing
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: logging
    ports:
    - protocol: TCP
      port: 9200
```

---

## 🎯 Complete Observability Architecture

### The Full Stack
```
┌─────────────────────────────────────────────────────────────────────┐
│                           OBSERVABILITY STACK                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  📊 METRICS          📝 LOGS              🔍 TRACES                 │
│  ┌─────────────┐    ┌─────────────┐      ┌─────────────┐            │
│  │ Prometheus  │    │ Elasticsearch│      │ Elasticsearch│            │
│  │ AlertManager│    │ FluentBit   │      │ Jaeger      │            │
│  │ Grafana     │    │ Kibana      │      │ OpenTelemetry│            │
│  └─────────────┘    └─────────────┘      └─────────────┘            │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│                         KUBERNETES CLUSTER                          │
│                                                                     │
│  ┌─────────────┐    ┌─────────────┐      ┌─────────────┐            │
│  │ Node        │    │ FluentBit   │      │ Jaeger      │            │
│  │ Exporter    │    │ DaemonSet   │      │ Agent       │            │
│  │ (DaemonSet) │    │             │      │ (DaemonSet) │            │
│  └─────────────┘    └─────────────┘      └─────────────┘            │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    APPLICATION PODS                         │    │
│  │  • Service A (Instrumented with OpenTelemetry)            │    │
│  │  • Service B (Instrumented with OpenTelemetry)            │    │
│  │  • Service C (Instrumented with OpenTelemetry)            │    │
│  └─────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```

### Access Points
- **Grafana**: `http://grafana-loadbalancer` (Metrics & Dashboards)
- **Kibana**: `http://kibana-loadbalancer:5601` (Logs & Search)
- **Jaeger**: `http://jaeger-query-loadbalancer` (Traces & Performance)

---

## 🎮 Hands-On Demo Walkthrough

### Scenario: E-commerce Application

#### 1. **Deploy Demo Application**
```bash
# Deploy instrumented microservices
kubectl apply -k ../day-4/kubernetes-manifest/

# Verify deployment
kubectl get pods -n dev
kubectl get svc -n dev
```

#### 2. **Generate Realistic Traffic**
```bash
# Get LoadBalancer URL
LOAD_BALANCER=$(kubectl get svc -n dev service-a-service -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

# Simulate user journey
echo "=== User Registration ==="
curl http://$LOAD_BALANCER/healthy

echo "=== User Login ==="
curl http://$LOAD_BALANCER/logs

echo "=== Browse Products ==="
curl http://$LOAD_BALANCER/call-service-b

echo "=== Add to Cart ==="
curl http://$LOAD_BALANCER/example

echo "=== Checkout Process ==="
curl http://$LOAD_BALANCER/call-service-b

echo "=== Payment Processing ==="
curl http://$LOAD_BALANCER/serverError  # Simulate error
```

#### 3. **Analyze Traces in Jaeger**

**Step 1**: Access Jaeger UI
```bash
kubectl port-forward svc/jaeger-query 8080:80 -n tracing --address 0.0.0.0
```

**Step 2**: Search for Traces
- Service: `service-a`
- Operation: `All`
- Lookback: `1h`
- Click "Find Traces"

**Step 3**: Analyze Performance
```
Trace Analysis Results:
┌─────────────────────────────────────────────────────────────┐
│ Trace: User Checkout Flow                                   │
├─────────────────────────────────────────────────────────────┤
│ Total Duration: 487ms                                       │
│ Services: 2 (service-a, service-b)                         │
│ Spans: 12                                                  │
│                                                            │
│ Performance Breakdown:                                      │
│ ├── service-a: authentication (23ms)                      │
│ ├── service-a: input validation (15ms)                    │
│ ├── service-a: call service-b (398ms) ← BOTTLENECK!       │
│ │   └── service-b: process request (387ms)               │
│ │       ├── service-b: database query (340ms) ← SLOW!    │
│ │       └── service-b: response formatting (47ms)        │
│ └── service-a: response preparation (51ms)                │
└─────────────────────────────────────────────────────────────┘
```

**Step 4**: Identify Issues
- **Database query taking 340ms** - needs optimization
- **Network latency between services** - 11ms overhead
- **Opportunity**: Cache database results, optimize query

---

## 🧹 Cleanup

When you're done experimenting:

```bash
# Remove Jaeger
helm uninstall jaeger -n tracing

# Remove certificates and secrets
kubectl delete configmap jaeger-tls -n tracing
kubectl delete secret es-tls-secret -n tracing
kubectl delete namespace tracing

# Remove Elasticsearch (if not needed for logging)
helm uninstall elasticsearch -n logging

# Remove test applications
kubectl delete -k ../day-4/kubernetes-manifest/
kubectl delete -k ../day-4/alerts-alertmanager-servicemonitor-manifest/

# Remove monitoring stack
helm uninstall monitoring -n monitoring

# Delete the entire cluster (optional)
eksctl delete cluster --name observability
```

---

## 🎯 Key Takeaways

1. **Distributed Tracing Completes Observability**: Together with metrics and logs, traces provide complete system visibility
2. **Jaeger is Production-Ready**: Simple, lightweight, and scalable distributed tracing solution
3. **OpenTelemetry is the Standard**: Vendor-neutral instrumentation that works with any tracing backend
4. **Instrumentation is Key**: Without proper instrumentation, tracing provides minimal value
5. **Performance Insights**: Traces reveal bottlenecks that metrics and logs cannot show
6. **Sampling is Important**: Balance between visibility and performance/cost
7. **Integration Matters**: Jaeger works best as part of a complete observability stack

---

## 🔗 Additional Resources

- [Jaeger Documentation](https://www.jaegertracing.io/docs/)
- [OpenTelemetry Documentation](https://opentelemetry.io/docs/)
- [Distributed Tracing Best Practices](https://opentelemetry.io/docs/concepts/observability-primer/)
- [Jaeger Performance Tuning](https://www.jaegertracing.io/docs/latest/performance-tuning/)
- [OpenTelemetry Instrumentation Libraries](https://opentelemetry.io/docs/instrumentation/)

---

