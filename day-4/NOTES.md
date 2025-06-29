# Day 4: Instrumentation - Custom Metrics & Alerting

---

## 🎯 What is Instrumentation?

### Definition
**Instrumentation** is the process of writing/embedding code in your applications to collect **metrics**, **logs**, or **traces** that provide insights into system performance and behavior.

### Why Instrumentation Matters
- **Without instrumentation**, even the most robust observability stack (Prometheus, Grafana, EFK) is useless
- **Exporters** can only provide general system information (CPU, memory)
- **Application-specific metrics** (user logins, business logic performance) require custom instrumentation
- It's the **foundation** of effective observability

### Key Benefits
- 🔍 **Visibility**: Gain insights into application internal state
- 📊 **Metrics Collection**: Track performance indicators specific to your business logic
- 🐛 **Troubleshooting**: Quick diagnosis when issues occur
- 📈 **Performance Monitoring**: Understand application behavior patterns

---

## 👥 Responsibility Matrix

| Role | Responsibilities |
|------|------------------|
| **Developers** | • Instrument custom metrics<br>• Provide application-specific information<br>• Set up basic dashboards |
| **Platform/DevOps Engineers** | • Configure alerting rules<br>• Set up monitoring infrastructure<br>• Manage service discovery |
| **Senior Engineers** | • Design observability strategy<br>• Review instrumentation patterns<br>• Optimize monitoring costs |

---

## 📊 Prometheus Metric Types

Understanding metric types is crucial for proper instrumentation - just like programming languages have different data types!

### 1. 🔄 Counter
- **Behavior**: Always incrementing, never decreases
- **Use Cases**: 
  - HTTP requests received
  - User login attempts
  - Errors encountered
  - Tasks completed
- **Example**: `http_requests_total`

```javascript
const httpRequestsTotal = new promClient.Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'route', 'status']
});
```

### 2. 📏 Gauge
- **Behavior**: Can increase and decrease
- **Use Cases**:
  - CPU utilization
  - Memory usage
  - Active connections
  - Queue size
- **Example**: `node_gauge_example`

```javascript
const nodeGaugeExample = new promClient.Gauge({
  name: 'node_gauge_example',
  help: 'Example gauge metric',
  labelNames: ['task_type']
});
```

### 3. 📊 Histogram
- **Behavior**: Records observations in configurable buckets
- **Use Cases**:
  - Request duration
  - Response sizes
  - Latency measurements
- **Example**: `http_request_duration_seconds`

```javascript
const httpRequestDuration = new promClient.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route'],
  buckets: [0.1, 0.5, 1, 2, 5, 10]
});
```

### 4. 📝 Summary
- **Behavior**: Similar to histogram but provides quantiles
- **Use Cases**:
  - 95th percentile response times
  - Median request durations
- **Example**: `http_request_duration_summary_seconds`

---

## 🛠️ Practical Implementation

### Step 1: Instrument Your Application

#### Node.js Example with Express
```javascript
const express = require('express');
const promClient = require('prom-client');

const app = express();

// Create metrics
const httpRequestsTotal = new promClient.Counter({
  name: 'http_requests_total',
  help: 'Total HTTP requests',
  labelNames: ['method', 'route', 'status']
});

const httpRequestDuration = new promClient.Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request duration',
  labelNames: ['method', 'route'],
  buckets: [0.1, 0.5, 1, 2, 5, 10]
});

// Middleware to track metrics
app.use((req, res, next) => {
  const start = Date.now();
  
  res.on('finish', () => {
    const duration = (Date.now() - start) / 1000;
    
    httpRequestsTotal.inc({
      method: req.method,
      route: req.route?.path || req.path,
      status: res.statusCode
    });
    
    httpRequestDuration.observe({
      method: req.method,
      route: req.route?.path || req.path
    }, duration);
  });
  
  next();
});

// Expose metrics endpoint
app.get('/metrics', (req, res) => {
  res.set('Content-Type', promClient.register.contentType);
  res.end(promClient.register.metrics());
});
```

### Step 2: Deploy to Kubernetes

#### Application Deployment
```bash
# Build and push images
docker build -t your-repo/service-a:v1 application/service-a/
docker build -t your-repo/service-b:v1 application/service-b/

# Deploy to Kubernetes
kubectl create ns dev
kubectl apply -k kubernetes-manifest/
```

#### Verify Deployment
```bash
kubectl get pods -n dev
kubectl get svc -n dev
```

### Step 3: Configure Service Discovery

#### ServiceMonitor Configuration
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: app-service-monitor
spec:
  selector:
    matchLabels:
      app: your-app
  endpoints:
  - port: http
    path: /metrics
    interval: 30s
```

#### Apply Service Discovery
```bash
kubectl apply -f service-monitor.yaml
```

**⏰ Wait Time**: Allow 2-3 minutes for Prometheus to discover and start scraping your application.

---

## 🚨 AlertManager Configuration

### Step 1: Set Up Email Credentials

#### For Gmail:
1. Enable 2-factor authentication
2. Generate an app password
3. Convert password to base64:
```bash
echo -n "your-app-password" | base64
```

#### Create Email Secret
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: email-secret
data:
  password: <base64-encoded-password>
```

### Step 2: Configure AlertManager

#### AlertManager Config
```yaml
apiVersion: monitoring.coreos.com/v1alpha1
kind: AlertmanagerConfig
metadata:
  name: email-config
spec:
  route:
    groupBy: ['alertname']
    receiver: 'email-receiver'
  receivers:
  - name: 'email-receiver'
    emailConfigs:
    - to: 'your-email@gmail.com'
      from: 'alerts@yourcompany.com'
      smarthost: 'smtp.gmail.com:587'
      authUsername: 'your-email@gmail.com'
      authPassword:
        name: email-secret
        key: password
```

### Step 3: Define Alert Rules

#### High CPU Usage Alert
```yaml
- alert: HighCpuUsage
  expr: avg(rate(cpu_usage_total[5m])) > 0.5
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "High CPU usage detected"
    description: "CPU usage is above 50% for more than 5 minutes"
```

#### Pod Restart Alert
```yaml
- alert: PodRestart
  expr: increase(kube_pod_container_status_restarts_total[5m]) > 2
  for: 0m
  labels:
    severity: critical
  annotations:
    summary: "Pod restarting frequently"
    description: "Pod {{ $labels.pod }} has restarted more than 2 times"
```

---

## 🧪 Testing Your Setup

### 1. Test Application Endpoints
```bash
# Health check
curl http://your-loadbalancer/healthy

# Generate metrics
curl http://your-loadbalancer/example

# View metrics
curl http://your-loadbalancer/metrics

# Test service communication
curl http://your-loadbalancer/call-service-b
```

### 2. Automated Testing
```bash
# Use the provided test script
./test.sh your-loadbalancer-dns-name
```

### 3. Test Alerting
```bash
# Crash the application multiple times
curl http://your-loadbalancer/crash
curl http://your-loadbalancer/crash
curl http://your-loadbalancer/crash
```

**Expected Result**: You should receive an email notification after the third crash.

---

## 🔍 Verification Steps

### 1. Check Prometheus Targets
1. Access Prometheus UI: `kubectl port-forward svc/prometheus-server 9090:80`
2. Navigate to Status → Targets
3. Verify your application appears as a target

### 2. Query Custom Metrics
In Prometheus UI, search for:
- `http_requests_total`
- `http_request_duration_seconds`
- `node_gauge_example`

### 3. Verify Alerting
1. Check AlertManager UI: `kubectl port-forward svc/alertmanager 9093:9093`
2. Verify email configuration
3. Test alert firing

---

## 📈 Monitoring Best Practices

### 1. Metric Naming Conventions
- Use descriptive names: `http_requests_total` not `requests`
- Include units: `duration_seconds` not `duration`
- Be consistent across services

### 2. Label Strategy
- Use labels for dimensions you want to filter/group by
- Avoid high-cardinality labels (like user IDs)
- Keep label names consistent

### 3. Performance Considerations
- Don't over-instrument (avoid metric explosion)
- Use appropriate metric types
- Consider sampling for high-frequency events

---

## 🎯 Key Takeaways

1. **Instrumentation is Essential**: Without it, your observability stack is incomplete
2. **Choose the Right Metric Type**: Counter for always-increasing values, Gauge for fluctuating values, Histogram for distributions
3. **Service Discovery is Critical**: Prometheus needs to know where to scrape metrics
4. **Test Your Alerts**: Always verify that your alerting pipeline works end-to-end
5. **Start Simple**: Begin with basic metrics and gradually add more sophisticated instrumentation

---

## 🔗 Additional Resources

- [Prometheus Client Libraries](https://prometheus.io/docs/instrumenting/clientlibs/)
- [Prometheus Best Practices](https://prometheus.io/docs/practices/naming/)
- [AlertManager Configuration](https://prometheus.io/docs/alerting/latest/configuration/)
- [Service Discovery in Kubernetes](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#kubernetes_sd_config)

---

## 📝 Next Steps

1. **Practice**: Implement custom metrics in your own applications
2. **Explore**: Try different metric types and understand their use cases
3. **Advanced Topics**: Look into OpenTelemetry for standardized instrumentation
4. **Integration**: Connect with distributed tracing systems like Jaeger

Remember: Effective observability starts with thoughtful instrumentation! 🚀
