# Observability: Introduction


---

## 🔍 What is Observability?

**Observability** is the capability to understand the internal state of a system—such as an application, its infrastructure, and network—by examining outputs like logs, metrics, and traces.

### Key Points:
- Helps determine **if a system is healthy** or facing issues.
- Goes beyond availability; it answers:
  - **What is happening?** → via metrics
  - **Why is it happening?** → via logs
  - **How to fix it?** → via traces

Think of observability like "X-ray vision" into your systems. It’s especially critical in distributed systems and microservice architectures.

---

## 🧱 The Three Pillars of Observability

### 1. 📈 **Metrics**
Quantitative data that represent the state of a system over time.

- Examples: CPU usage, memory usage, HTTP request count
- Collected at intervals
- Useful for detecting *what* is wrong
- Enables historical analysis (e.g., "What was CPU usage at 2 PM yesterday?")

### 2. 📜 **Logs**
Textual records of events that have occurred in the system.

- Types: Info, Debug, Error, Trace logs
- Help identify *why* something happened
- Useful during incident diagnosis and debugging
- Provide contextual details (e.g., stack traces, user actions)

### 3. 🧵 **Traces**
Step-by-step journey of a request through the system.

- Tells you *how* something happened
- Example: Request path from Load Balancer → Frontend → Backend → Database
- Great for understanding performance bottlenecks or pinpointing failure points in distributed architectures

---

## 📊 Observability vs Monitoring

| Feature              | Monitoring                                      | Observability                                      |
|----------------------|--------------------------------------------------|-----------------------------------------------------|
| Focus                | Metrics                                          | Metrics, Logs, Traces (holistic)                   |
| Reactive or Proactive| Primarily reactive                               | Enables both proactive and reactive analysis        |
| Scope                | Detect anomalies (e.g., via alerts/dashboards)  | Diagnose root cause, explain *why/how*              |
| Tools                | Prometheus, Grafana                             | Prometheus, ELK, Jaeger, OpenTelemetry, etc.       |

> 💡 Monitoring is a **subset** of observability.

---

## 📈 Why Do Companies Invest in Observability?

### SLA (Service Level Agreement) Example
- Company builds a resume builder deployed on AWS EKS.
- Signs an SLA promising:
  - 99.9% uptime
  - 99.99% of requests completed within 30ms

### Why Observability?
- To **prove** SLA adherence (data-backed)
- To **act immediately** when there's degradation
- To maintain **user trust** and application reliability

### Without Observability
- Impossible to know when requests are failing
- No root cause for performance issues
- No ability to improve proactively

---

## 👩‍💻 Who's Responsible for Observability?

### Developers
- Must **instrument** code:
  - Emit logs
  - Expose custom metrics
  - Trace request flows
- Use libraries like:
  - `Prometheus Client` (e.g., `prom-client` in Node.js)
  - OpenTelemetry SDKs

### DevOps / SRE Engineers
- **Set up** infrastructure:
  - Prometheus, Grafana
  - ELK/EFK stack
  - Jaeger for tracing
- **Ensure ingestion pipelines**, storage, alerting, and dashboards are work properly

### TL;DR: It’s a **collaborative effort**. One side without the other = half-baked observability.

---

## 🛠️ Tools Mentioned

| Tool          | Purpose                            |
|---------------|------------------------------------|
| Prometheus    | Metrics collection and alerting    |
| Grafana       | Visualization and dashboards       |
| Fluent Bit    | Log forwarding (EFK stack)         |
| Elasticsearch | Log storage and search             |
| Kibana        | Log visualization                  |
| Jaeger        | Distributed tracing                |
| OpenTelemetry | Vendor-neutral observability SDKs  |

---

## 🧠 Summary

Observability is:
- **More than just monitoring**
- Essential for modern apps (Kubernetes, microservices)
- Built on **metrics, logs, and traces**
- A **shared responsibility** between devs and ops
- Crucial for **diagnosing and fixing issues quickly**

---

## 📝 Tips for Applying Observability as a Student

- Start by understanding your application’s normal behavior.
- Add logging intentionally; avoid excessive or meaningless logs.
- Use Prometheus locally or via cloud-native tooling (e.g., kube-prometheus-stack).
- Familiarize yourself with OpenTelemetry—it's becoming the standard.
- Correlate metrics, logs, and traces during debugging—think like a detective.


---

Stay tuned for deep-dives into:
- Prometheus & PromQL
- Logging with EFK
- Tracing with Jaeger & OpenTelemetry
- Advanced observability with eBPF

> 🧠 Remember: Observability isn't just a tool—it's a **practice**.



