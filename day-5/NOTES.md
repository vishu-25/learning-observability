# Day 5: Logging with EFK Stack (Elasticsearch, FluentBit, Kibana)

## 🔍 Introduction to Logging

### What is Logging?
**Logging** is the process of recording messages that explain what an application is doing at any particular point in time. These messages are written by developers to help:
- **Debug applications** and identify issues
- **Understand application flow** and behavior  
- **Track critical events** and state changes
- **Audit system activities** and user actions

### Simple Example: With vs Without Logging

#### Without Logging:
```javascript
function addNumbers(a, b) {
    return a + b;
}
const result = addNumbers(5, 3);
```

#### With Logging:
```javascript
function addNumbers(a, b) {
    console.log(`Starting addition: ${a} + ${b}`);
    const result = a + b;
    console.log(`Addition completed: Result = ${result}`);
    return result;
}
const result = addNumbers(5, 3);
console.log(`Final result stored: ${result}`);
```

The second version provides visibility into what's happening at each step!

---

## 🎯 Why Logging is Critical in Distributed Systems

### The Challenge
In complex applications with **thousands of lines of code** and **multiple microservices**, identifying issues becomes extremely difficult without proper logging.

### Key Benefits of Logging

| Benefit | Description | Example |
|---------|-------------|---------|
| 🐛 **Debugging** | Identify where and why failures occur | "Database connection failed at line 245" |
| 📊 **Performance Monitoring** | Track response times and bottlenecks | "API call took 2.5 seconds" |
| 🔒 **Security** | Detect unauthorized access attempts | "Failed login attempt from IP 192.168.1.100" |
| 📋 **Auditing** | Track user actions and system changes | "User John deleted file config.yaml" |
| 🔍 **Troubleshooting** | Understand application flow during incidents | "Payment processing started → validation failed → transaction rolled back" |

### Real-World Scenarios
- **Database Connection Issues**: Logs help identify intermittent connection problems across services
- **Security Vulnerabilities**: Quickly identify and react to issues like Log4J vulnerabilities
- **Performance Degradation**: Track slow queries or high latency endpoints
- **Business Logic Errors**: Understand why critical workflows are failing

---

## 🏗️ The Three Pillars of Observability

| Pillar | Purpose | What It Tells You |
|--------|---------|-------------------|
| 📊 **Metrics** | Quantitative measurements | *How much?* (CPU usage, request count) |
| 📝 **Logs** | Event records and messages | *What happened?* (Error messages, user actions) |
| 🔍 **Traces** | Request journey tracking | *How did it flow?* (Request path through services) |

**Logs** specifically help answer: **"Why is my application failing?"**

---

## 🏠 Centralized Logging: The Solution

### The Problem with Distributed Logs
Without centralization:
- ❌ Need to check logs on **hundreds of different services**
- ❌ **Time-consuming** manual investigation
- ❌ **Difficult correlation** between services
- ❌ **No unified search** capabilities

### The Centralized Solution
With a centralized logging system:
- ✅ **Single query** across all services
- ✅ **Fast identification** of issues
- ✅ **Correlation** between different services
- ✅ **Unified search** and filtering

### Example Query Benefits
Instead of checking 50 different services manually:
```bash
# Single query to find database issues across ALL services
"database connection timeout" AND namespace:"production"
```

---

## 📦 EFK Stack Overview

The **EFK Stack** consists of three main components that work together to provide comprehensive logging:

### 🔍 **E**lasticsearch
- **Purpose**: Database for storing and indexing logs
- **Features**:
  - Full-text search capabilities
  - Horizontal scaling
  - Real-time indexing
  - RESTful API
- **Storage**: Can be backed by EBS volumes for persistence
- **Capacity**: Handle gigabytes of logs per day

### 🚀 **F**luentBit
- **Purpose**: Lightweight log forwarder and collector
- **Deployment**: Runs as DaemonSet on every Kubernetes node
- **Function**: 
  - Reads logs from container log files
  - Applies filters and transformations
  - Forwards logs to Elasticsearch
- **Benefits**: 
  - Low resource consumption
  - Vendor-neutral configuration
  - Reliable log delivery

### 📊 **K**ibana
- **Purpose**: Visualization and query interface
- **Features**:
  - Interactive dashboards
  - Advanced query language (KQL)
  - Real-time log exploration
  - Custom visualizations
- **Access**: Web-based UI for end users

---

## 🏗️ EFK Architecture

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│   Application   │    │    FluentBit     │    │  Elasticsearch  │
│     Pods        │───▶│   (DaemonSet)    │───▶│   (Database)    │
│                 │    │                  │    │                 │
│ • Service A     │    │ • Log Collection │    │ • Log Storage   │
│ • Service B     │    │ • Filtering      │    │ • Indexing      │
│ • Service C     │    │ • Forwarding     │    │ • Search API    │
└─────────────────┘    └──────────────────┘    └─────────────────┘
                                                         │
                                                         ▼
                                               ┌─────────────────┐
                                               │     Kibana      │
                                               │ (Visualization) │
                                               │                 │
                                               │ • Dashboards    │
                                               │ • Queries       │
                                               │ • Analytics     │
                                               └─────────────────┘
```

### Data Flow
1. **Applications** write logs to stdout/stderr
2. **Kubernetes** stores logs in `/var/log/containers/`
3. **FluentBit** reads logs from each node
4. **FluentBit** applies filters and forwards to Elasticsearch
5. **Elasticsearch** indexes and stores logs
6. **Kibana** provides UI for querying and visualization

![Project Architecture](images/architecture.gif)
---

## 🛠️ EFK vs ELK: Understanding the Difference

| Component | EFK Stack | ELK Stack |
|-----------|-----------|-----------|
| **Log Processor** | FluentBit | Logstash |
| **Resource Usage** | Lightweight | Resource-intensive |
| **Features** | Basic forwarding | Advanced processing |
| **Complexity** | Simple configuration | Complex pipelines |
| **Reliability** | Lower chance of issues | More potential failures |
| **Use Case** | Most organizations | Advanced filtering needs |

**Recommendation**: FluentBit is sufficient for most organizations and provides better reliability.

---

## 🚀 Step-by-Step EFK Deployment

### Prerequisites
- Kubernetes cluster (EKS recommended)
- Helm installed
- kubectl configured
- AWS CLI configured (for EKS)

### Step 1: Create IAM Role for EBS CSI Driver

```bash
# Create IAM service account for EBS CSI controller
eksctl create iamserviceaccount \
    --name ebs-csi-controller-sa \
    --namespace kube-system \
    --cluster observability \
    --role-name AmazonEKS_EBS_CSI_DriverRole \
    --role-only \
    --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
    --approve
```

**Why this step?** Elasticsearch needs persistent storage (EBS volumes) to survive pod restarts.

### Step 2: Retrieve IAM Role ARN

```bash
# Get the ARN of the created role
ARN=$(aws iam get-role --role-name AmazonEKS_EBS_CSI_DriverRole --query 'Role.Arn' --output text)
echo $ARN
```

### Step 3: Deploy EBS CSI Driver

```bash
# Install EBS CSI driver addon
eksctl create addon --cluster observability --name aws-ebs-csi-driver --version latest \
    --service-account-role-arn $ARN --force
```

**Result**: Kubernetes can now create and manage EBS volumes automatically.

### Step 4: Create Logging Namespace

```bash
# Create dedicated namespace for logging components
kubectl create namespace logging
```

### Step 5: Install Elasticsearch

```bash
# Add Elastic Helm repository
helm repo add elastic https://helm.elastic.co

# Install Elasticsearch with persistent storage
helm install elasticsearch \
 --set replicas=1 \
 --set volumeClaimTemplate.storageClassName=gp2 \
 --set persistence.labels.enabled=true \
 elastic/elasticsearch -n logging
```

**Configuration Explained**:
- `replicas=1`: Single Elasticsearch instance (increase for production)
- `storageClassName=gp2`: Use AWS EBS GP2 volumes
- `persistence.labels.enabled=true`: Enable persistent storage labels

### Step 6: Retrieve Elasticsearch Credentials

```bash
# Get username (usually 'elastic')
kubectl get secrets --namespace=logging elasticsearch-master-credentials \
  -ojsonpath='{.data.username}' | base64 -d

# Get password (save this!)
kubectl get secrets --namespace=logging elasticsearch-master-credentials \
  -ojsonpath='{.data.password}' | base64 -d
```

**⚠️ Important**: Save the password - you'll need it for FluentBit configuration!

### Step 7: Install Kibana

```bash
# Install Kibana with LoadBalancer service
helm install kibana --set service.type=LoadBalancer elastic/kibana -n logging
```

**Access**: Kibana will be accessible via LoadBalancer DNS on port 5601.

### Step 8: Configure and Install FluentBit

#### Update FluentBit Configuration
Edit `fluentbit-values.yaml` and update the password:

```yaml
config:
  outputs: |
    [OUTPUT]
        Name es
        Match *
        Host elasticsearch-master
        Port 9200
        HTTP_User elastic
        HTTP_Passwd YOUR_ELASTICSEARCH_PASSWORD_HERE  # Update this!
        TLS On
        TLS.Verify Off
```

#### Install FluentBit
```bash
# Add FluentBit Helm repository
helm repo add fluent https://fluent.github.io/helm-charts

# Install FluentBit with custom configuration
helm install fluent-bit fluent/fluent-bit -f fluentbit-values.yaml -n logging
```

### Step 9: Verify Installation

```bash
# Check all logging components
kubectl get pods -n logging

# Expected output:
# NAME                             READY   STATUS    RESTARTS   AGE
# elasticsearch-master-0           1/1     Running   0          5m
# kibana-kibana-7c8b8b8b8b-xyz12   1/1     Running   0          3m
# fluent-bit-xxxxx                 1/1     Running   0          2m
```

---

## 🧪 Testing the EFK Stack

### Step 1: Deploy Test Application

```bash
# Deploy sample applications from day-4
kubectl apply -k ../day-4/kubernetes-manifest/
```

### Step 2: Generate Logs

```bash
# Check application logs
kubectl logs -f deployment/service-a -n dev

# Generate some activity
curl http://your-loadbalancer/logs
curl http://your-loadbalancer/serverError
```

### Step 3: Verify Log Collection

```bash
# Check FluentBit is collecting logs
kubectl logs -f daemonset/fluent-bit -n logging

# Look for messages like:
# [INFO] [output:es:es.0] worker #0 started
# [INFO] [input:tail:tail.0] inotify_fs_add(): inode=12345 watch_fd=1 name=/var/log/containers/service-a-*.log
```

### Step 4: Access Kibana

1. Get Kibana LoadBalancer URL:
```bash
kubectl get svc -n logging kibana-kibana
```

2. Access Kibana at: `http://LOAD_BALANCER_DNS:5601`

3. Login with:
   - **Username**: `elastic`
   - **Password**: `<password from Step 6>`

---

## 📊 Using Kibana for Log Analysis

### Step 1: Create Data View

1. Navigate to **Stack Management** → **Data Views**
2. Click **Create data view**
3. Set **Name**: `kubernetes-logs`
4. Set **Index pattern**: `logstash-*`
5. Set **Timestamp field**: `@timestamp`
6. Click **Save data view to Kibana**

### Step 2: Explore Logs

1. Go to **Analytics** → **Discover**
2. Select your data view: `kubernetes-logs`
3. You should see logs from all your applications!

### Step 3: Create Useful Queries

#### Filter by Namespace
```
kubernetes.namespace_name: "dev"
```

#### Find Error Messages
```
log: "error" OR log: "ERROR" OR log: "exception"
```

#### Filter by Application
```
kubernetes.pod_name: service-a*
```

#### Exclude System Logs
```
NOT kubernetes.namespace_name: ("kube-system" OR "logging")
```

### Step 4: Create Visualizations

1. Go to **Analytics** → **Dashboard**
2. Click **Create dashboard**
3. Add visualizations:
   - **Log volume over time** (Line chart)
   - **Error rate by service** (Pie chart)
   - **Top error messages** (Data table)

---

## ⚙️ FluentBit Configuration Deep Dive

FluentBit configuration has four main sections:

### 1. Service Configuration
```ini
[SERVICE]
    Flush         1
    Log_Level     info
    Daemon        off
    Parsers_File  parsers.conf
```

### 2. Input Configuration
```ini
[INPUT]
    Name              tail
    Path              /var/log/containers/*.log
    Parser            docker
    Tag               kube.*
    Refresh_Interval  5
    Mem_Buf_Limit     50MB
```

**Explanation**: Reads container logs from Kubernetes nodes.

### 3. Filter Configuration
```ini
[FILTER]
    Name                kubernetes
    Match               kube.*
    Kube_URL            https://kubernetes.default.svc:443
    Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    Kube_Token_File     /var/run/secrets/kubernetes.io/serviceaccount/token
```

**Explanation**: Enriches logs with Kubernetes metadata (pod name, namespace, etc.).

### 4. Output Configuration
```ini
[OUTPUT]
    Name            es
    Match           *
    Host            elasticsearch-master
    Port            9200
    HTTP_User       elastic
    HTTP_Passwd     your-password
    TLS             On
    TLS.Verify      Off
    Index           logstash
```

**Explanation**: Forwards logs to Elasticsearch with authentication.

### Advanced Filtering Example

To exclude logs from specific namespaces:

```ini
[FILTER]
    Name    grep
    Match   *
    Exclude log ^(?!.*(kube-system|logging)).*$
```

---

## 📈 Production Considerations

### 1. Storage Planning

#### Elasticsearch Storage Requirements
- **Small deployment**: 10-50 GB per day
- **Medium deployment**: 100-500 GB per day  
- **Large deployment**: 1-10 TB per day

#### EBS Volume Configuration
```bash
# For production, use larger volumes
--set volumeClaimTemplate.resources.requests.storage=100Gi
--set volumeClaimTemplate.storageClassName=gp3  # Better performance
```

### 2. High Availability Setup

#### Multiple Elasticsearch Replicas
```bash
helm install elasticsearch \
 --set replicas=3 \
 --set minimumMasterNodes=2 \
 --set volumeClaimTemplate.resources.requests.storage=100Gi \
 elastic/elasticsearch -n logging
```

#### Elasticsearch Backup Strategy
```bash
# Enable snapshots to S3
--set esConfig."elasticsearch\.yml"="cluster.name: my-cluster\nxpack.security.enabled: true\npath.repo: [\"/usr/share/elasticsearch/backup\"]"
```

### 3. Performance Optimization

#### FluentBit Tuning
```yaml
config:
  service: |
    [SERVICE]
        Flush         5      # Increase for better performance
        Log_Level     warn   # Reduce log verbosity
        
  inputs: |
    [INPUT]
        Name              tail
        Path              /var/log/containers/*.log
        Mem_Buf_Limit     100MB  # Increase buffer size
        Skip_Long_Lines   On     # Skip very long log lines
```

### 4. Security Best Practices

#### Enable TLS
```yaml
# In fluentbit-values.yaml
config:
  outputs: |
    [OUTPUT]
        Name es
        Match *
        Host elasticsearch-master
        Port 9200
        TLS On
        TLS.Verify On  # Enable certificate verification
```

#### Network Policies
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: logging-network-policy
  namespace: logging
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: logging
```

---

## 🔍 Troubleshooting Common Issues

### 1. FluentBit Not Collecting Logs

#### Check FluentBit Status
```bash
kubectl logs -f daemonset/fluent-bit -n logging
```

#### Common Issues:
- **Permission problems**: Check RBAC configuration
- **Path issues**: Verify log file paths in configuration
- **Memory limits**: Increase memory limits if needed

### 2. Elasticsearch Connection Issues

#### Check Elasticsearch Health
```bash
kubectl exec -it elasticsearch-master-0 -n logging -- curl -u elastic:PASSWORD localhost:9200/_cluster/health
```

#### Common Issues:
- **Authentication failures**: Verify username/password
- **TLS problems**: Check certificate configuration
- **Network connectivity**: Verify service names and ports

### 3. Kibana Access Issues

#### Check Kibana Service
```bash
kubectl get svc -n logging kibana-kibana
kubectl describe svc -n logging kibana-kibana
```

#### Common Issues:
- **LoadBalancer not provisioned**: Check AWS load balancer controller
- **Security groups**: Verify inbound rules for port 5601
- **Health checks failing**: Check Kibana pod logs

---

## 🎯 Best Practices

### 1. Log Management
- **Retention policies**: Set appropriate log retention (7-30 days typically)
- **Index lifecycle management**: Automatically delete old indices
- **Log levels**: Use appropriate log levels (DEBUG, INFO, WARN, ERROR)

### 2. Query Optimization
- **Use filters**: Always filter by time range and namespace
- **Avoid wildcards**: Use specific field matches when possible
- **Index patterns**: Create specific index patterns for different applications

### 3. Monitoring the Monitoring
- **Set up alerts**: Monitor Elasticsearch cluster health
- **Disk space**: Alert when storage is running low
- **FluentBit health**: Monitor log collection rates

---

## 🧹 Cleanup

When you're done experimenting:

```bash
# Remove EFK stack
helm uninstall fluent-bit -n logging
helm uninstall elasticsearch -n logging  
helm uninstall kibana -n logging

# Remove test applications
kubectl delete -k ../day-4/kubernetes-manifest/
kubectl delete -k ../day-4/alerts-alertmanager-servicemonitor-manifest/

# Remove monitoring stack if installed
helm uninstall monitoring -n monitoring

# Delete the entire cluster (optional)
eksctl delete cluster --name observability
```

---

## 🎯 Key Takeaways

1. **Logging is Essential**: Without proper logging, debugging distributed systems is nearly impossible
2. **Centralization is Key**: Centralized logging makes troubleshooting exponentially easier
3. **EFK Stack is Powerful**: Provides enterprise-grade logging capabilities
4. **FluentBit is Lightweight**: Better choice than Logstash for most use cases
5. **Configuration Matters**: Proper FluentBit configuration is crucial for reliable log collection
6. **Plan for Scale**: Consider storage, performance, and costs in production
7. **Security First**: Always enable TLS and proper authentication

---

## 🔗 Additional Resources

- [Elasticsearch Documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)
- [FluentBit Documentation](https://docs.fluentbit.io/)
- [Kibana User Guide](https://www.elastic.co/guide/en/kibana/current/index.html)
- [Kubernetes Logging Architecture](https://kubernetes.io/docs/concepts/cluster-administration/logging/)
- [EFK Stack Best Practices](https://www.elastic.co/guide/en/elasticsearch/reference/current/best-practices.html)

---