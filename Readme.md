# OTEL (OpenTelemetry) - Complete Guide

## Table of Contents
1. [What is OTEL?](#what-is-otel)
2. [Telemetry & Observability](#telemetry--observability)
3. [Core Components](#core-components)
4. [Protocols](#protocols)
5. [Architecture](#architecture)
6. [Implementation & Benefits](#implementation--benefits)
7. [Getting Started](#getting-started)

---

## What is OTEL?

**OpenTelemetry (OTEL)** is an open-source CNCF observability framework providing unified, vendor-agnostic collection, processing, and export of telemetry data. It is a **set of standards, APIs, and tools** for instrumenting applications and infrastructure to generate and collect telemetry signals.

**Key Aspects:**
- **Standard Specification**: Defines how telemetry should be collected, processed, and exported
- **Language SDKs**: Implementations of the standard for multiple languages
- **APIs & Protocols**: OTLP (OpenTelemetry Protocol) for communication
- **Ecosystem**: Tools, exporters, and integrations across the observability landscape

**Why OTEL Matters:**
- **Vendor Neutral**: Independent of any platform
- **Standardized**: Common protocols & formats
- **Language Agnostic**: Multi-language support
- **Future-Proof**: Industry-backed standard

---

## Telemetry & Observability

### What is Telemetry?

**Telemetry** = All signals emitted by systems (logs, metrics, traces, events) providing visibility into behavior and state.

| Component | Characteristics | Best For |
|-----------|-----------------|----------|
| **Logs** | Time-stamped events, high volume | Debugging, investigation |
| **Metrics** | Numeric values, time-series, low volume | Trends, alerting |
| **Traces** | Request flows, hierarchical | Root cause analysis |
| **Events** | Notable occurrences | Context & correlation |

### Telemetry vs Monitoring vs Observability

| Aspect | Telemetry | Monitoring | Observability |
|--------|-----------|-----------|---------------|
| **What** | The data itself | Predefined metrics + alerts | Ask any question |
| **Approach** | Data collection | Reactive (wait for alerts) | Proactive |
| **Discovery** | Foundation | Known issues only | Unknown issues |
| **Time to Root Cause** | N/A | Hours | Minutes |

### Real Example: 3 AM Checkout Service Fails

**Monitoring Only:**
- No alerts triggered (CPU 45%, Memory 62%, Error rate 2%)
- Issue found next morning by users
- 3-4 hours to diagnose: Bad deployment at 2:45 AM

**Observability:**
- Traces show validation-service timeout
- Logs reveal deployment failure
- 347 transactions affected (~$5,200 loss)
- Diagnosed in 5 minutes, immediate rollback

### Telemetry Best Practices
1. **Structured Logging**: Use JSON format
2. **Correlation IDs**: Link logs with traces
3. **Sampling**: Strategic, not all data
4. **Cardinality Control**: Avoid high-cardinality tags
5. **Privacy**: No passwords, PII, tokens
6. **Retention**: Balance cost vs retention
7. **Context**: Enrich all signals with metadata

---

## Core Components

| Component | Purpose |
|-----------|---------|
| **Instrumentation** | Adding observability code (automatic, manual, or library-based) |
| **Collector** | Standalone service: receives → processes → exports telemetry |
| **SDKs** | Language-specific implementations (Python, Java, Go, Node.js, .NET, Ruby, PHP, C++) |
| **APIs** | Standard interfaces (Trace API, Metrics API, Logs API) |

**Collector Flow:**
```
Receivers (OTLP, Prometheus, Jaeger, Zipkin, Syslog, etc.)
    ↓
Processors (Batch, Sampling, Memory Limiting)
    ↓
Exporters (Jaeger, Prometheus, Datadog, Honeycomb, etc.)
```

---

## Protocols

### OTLP (OpenTelemetry Protocol)
Native OTEL protocol with two transport options:

| Transport | Port | Characteristics | Best For |
|-----------|------|-----------------|----------|
| **gRPC** | 4317 | Binary, HTTP/2, low overhead, fast | Production, high-volume |
| **HTTP** | 4318 | Text-based, firewall-friendly, easy to debug | Development, testing |

### Receiver Protocols Supported

| Protocol | Port | Use Case |
|----------|------|----------|
| OTLP (gRPC/HTTP) | 4317/4318 | Direct OTEL instrumentation |
| Prometheus | 9090 | Metrics scraping |
| Jaeger | 14250/14268 | Distributed tracing |
| Zipkin | 9411 | Distributed tracing |
| Syslog | 514 | System logs |
| Fluent Forward | 24224 | Fluentd integration |
| Kafka | 9092 | High-volume streaming |
| StatsD | 8125 | Legacy metrics |
| Collectd | 25826 | System monitoring |

### gRPC vs HTTP Comparison

| Aspect | gRPC | HTTP |
|--------|------|------|
| Overhead | Low | Higher |
| Speed | Faster (HTTP/2) | Slower (HTTP/1.1) |
| Compatibility | Firewall issues | Better support |
| Bandwidth | Efficient | Less efficient |
| Debugging | Binary (harder) | Text (easier) |
| Payload Size | Smaller | Larger |

---

## Architecture

```
Apps (Instrumented) 
    ↓ OTLP/gRPC (4317) or HTTP (4318)
OTEL Collector
    ├─ Receivers (OTLP, Prometheus, Jaeger, Zipkin, Syslog, etc.)
    ├─ Processors (Batch, Sampling, Memory Limiting)
    └─ Exporters (Jaeger, Prometheus, Datadog, Honeycomb, etc.)
    ↓
Backends (Jaeger, Prometheus, Datadog, etc.)
```

**Example Collector Config:**
```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
processors:
  batch:
    timeout: 10s
exporters:
  jaeger:
    endpoint: jaeger:14250
service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [jaeger]
```

---

## Implementation & Benefits

### Three Pillars of Observability
| Pillar | Purpose |
|--------|---------|
| **Traces** | Request flows through services, identify bottlenecks |
| **Metrics** | Quantitative data (latency, errors, resource usage) |
| **Logs** | Detailed events with context and correlation IDs |

### Implementation Patterns

#### 1. SDK-Based Instrumentation
Direct integration of OTEL SDKs within application code. Developers explicitly add instrumentation.

**Characteristics:**
- Maximum control and flexibility
- Code changes required
- Can instrument specific business operations
- Language-specific SDKs (Python, Java, Go, Node.js, .NET, etc.)

**Example (Python):**
```python
from opentelemetry import trace
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor

# Initialize
otlp_exporter = OTLPSpanExporter(endpoint="localhost:4317")
trace.set_tracer_provider(TracerProvider())
trace.get_tracer_provider().add_span_processor(
    BatchSpanProcessor(otlp_exporter)
)
tracer = trace.get_tracer(__name__)

# Use in code
with tracer.start_as_current_span("process_payment") as span:
    span.set_attribute("payment.id", payment_id)
    # Business logic
```

**Pros:** Full control, lightweight, custom instrumentation  
**Cons:** Code changes needed, developer effort required

---

#### 2. Collector-Based Architecture
Centralized collection and processing of telemetry using OpenTelemetry Collector as a standalone service.

**Characteristics:**
- Receives data from multiple sources
- Processes and filters telemetry
- Exports to multiple backends
- Works as middleware between apps and backends

**Collector Configuration (YAML):**
```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318
  prometheus:
    config:
      scrape_configs:
        - job_name: 'app-metrics'
          static_configs:
            - targets: ['localhost:8888']

processors:
  batch:
    timeout: 10s
    send_batch_size: 1024
  memory_limiter:
    limit_mib: 512
  sampling:
    sampling_percentage: 10

exporters:
  jaeger:
    endpoint: jaeger:14250
  prometheus:
    endpoint: 0.0.0.0:8888
  datadog:
    api:
      key: ${DD_API_KEY}

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [jaeger, datadog]
    metrics:
      receivers: [otlp, prometheus]
      processors: [batch]
      exporters: [prometheus, datadog]
```

**Deployment in Kubernetes:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-collector-config
data:
  config.yaml: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
    exporters:
      jaeger:
        endpoint: jaeger:14250
    service:
      pipelines:
        traces:
          receivers: [otlp]
          exporters: [jaeger]
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: otel-collector
spec:
  replicas: 2
  selector:
    matchLabels:
      app: otel-collector
  template:
    metadata:
      labels:
        app: otel-collector
    spec:
      containers:
      - name: otel-collector
        image: otel/opentelemetry-collector:latest
        ports:
        - containerPort: 4317  # gRPC
        - containerPort: 4318  # HTTP
        volumeMounts:
        - name: config
          mountPath: /etc/otel/config.yaml
          subPath: config.yaml
      volumes:
      - name: config
        configMap:
          name: otel-collector-config
```

**Pros:** Centralized control, multiple exporters, data filtering  
**Cons:** Additional service to manage, network latency

---

#### 3. Auto-Instrumentation (Agent-Based)
Automatic injection of instrumentation agents without modifying application code. Agents intercept and instrument applications at runtime.

**Characteristics:**
- Zero-code instrumentation
- Agent/operator injects instrumentation
- Works transparently to developers
- Mutating webhooks (in Kubernetes) inject agents automatically
- Can instrument entire microservices architecture without code changes

**Example A: OpenTelemetry Operator in Kubernetes**

```bash
# Install OpenTelemetry Operator
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm install opentelemetry-operator open-telemetry/opentelemetry-operator \
  -n opentelemetry-operator-system \
  --create-namespace
```

**Instrumentation Configuration:**
```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: python-instrumentation
spec:
  python:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-python:latest
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
spec:
  template:
    metadata:
      labels:
        app: payment-service
      annotations:
        instrumentation.opentelemetry.io/inject-python: "true"  # Enables auto-instrumentation
    spec:
      containers:
      - name: app
        image: myapp:latest
        # NO instrumentation code needed!
```

**What happens:**
1. Operator's mutating webhook intercepts pod creation
2. Webhook injects Python instrumentation agent
3. Agent automatically instruments the application
4. Telemetry sent to configured collector

---

**Example B: Dynatrace Operator in EKS (Agent-Based)**

Dynatrace Operator is a Kubernetes operator that uses auto-instrumentation to inject Dynatrace agents.

```bash
# Install Dynatrace Operator via Helm in EKS
helm repo add dynatrace https://www.dynatrace.com/support/help/shortlink/helm-repo
helm install dynatrace-operator dynatrace/dynatrace-operator \
  -n dynatrace \
  --create-namespace \
  --set apiToken=$DT_API_TOKEN \
  --set paasToken=$DT_PAAS_TOKEN \
  --set environment=$DT_ENVIRONMENT_ID \
  --set clusterID=$CLUSTER_NAME
```

**Standard EKS Deployment (No code changes):**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: payment-service
  template:
    metadata:
      labels:
        app: payment-service
      annotations:
        dynatrace.com/inject: "true"  # Enables Dynatrace auto-instrumentation
    spec:
      containers:
      - name: app
        image: myapp:latest
        ports:
        - containerPort: 8080
        env:
        - name: SERVICE_PORT
          value: "8080"
        # NO instrumentation code, NO OTEL SDK required!
```

**What Dynatrace Operator Does:**
1. Watches for pod creation in EKS
2. Mutating webhook intercepts pods with `dynatrace.com/inject: "true"`
3. Injects Dynatrace agent (e.g., Java agent via `JAVA_TOOL_OPTIONS`)
4. Agent instruments application at startup
5. Automatically collects traces, metrics, logs
6. Sends to Dynatrace backend (proprietary format)

**Dynatrace Agent Injection (Behind the Scenes):**
```yaml
# What the webhook actually injects:
spec:
  containers:
  - name: app
    image: myapp:latest
    env:
    - name: JAVA_TOOL_OPTIONS
      value: "-javaagent:/opt/dynatrace/agent/lib/liboneagentloader.so"
    - name: LD_PRELOAD
      value: "/opt/dynatrace/agent/lib/liboneagentloader.so"
    volumeMounts:
    - name: dynatrace-agent
      mountPath: /opt/dynatrace/agent
  volumes:
  - name: dynatrace-agent
    emptyDir: {}
  initContainers:
  - name: dynatrace-agent-init
    image: dynatrace/oneagent-operator:latest
    command:
    - /bin/sh
    - -c
    - cp -r /opt/dynatrace/agent /mnt/agent
    volumeMounts:
    - name: dynatrace-agent
      mountPath: /mnt/agent
```

**Comparison: Dynatrace vs OTEL Auto-Instrumentation**

| Aspect | Dynatrace Operator | OTEL Operator |
|--------|-------------------|---------------|
| **Vendor** | Proprietary | Open-source (CNCF) |
| **Backend** | Dynatrace only | Any OTEL backend |
| **Zero-Code** | ✅ YES | ✅ YES |
| **Kubernetes** | ✅ YES (EKS, GKE, AKS) | ✅ YES |
| **Languages** | Java, .NET, Python, Node.js, Go | Python, Java, Node.js, Go, etc. |
| **Agent Type** | Language-specific agents | OTEL instrumentation libraries |
| **Cost** | Dynatrace SaaS pricing | Free (open-source) |
| **Flexibility** | Locked to Dynatrace | Multi-backend support |
| **Data Format** | Proprietary | OTLP standard |

**Pros:** Zero code changes, automatic for entire cluster, works across languages  
**Cons:** Operator overhead, limited flexibility (for proprietary solutions), vendor lock-in (Dynatrace)

---

#### Summary: Which Pattern to Use?

| Pattern | Use When |
|---------|----------|
| **SDK-Based** | Need maximum control, custom instrumentation, lightweight setup |
| **Collector-Based** | Want centralized telemetry hub, multiple backends, filtering |
| **Auto-Instrumentation** | Want zero-code approach, instrumenting entire Kubernetes cluster, prefer operational simplicity |

### Key Benefits

| Benefit | Impact |
|---------|--------|
| **Unified Observability** | Single framework for traces, metrics, logs |
| **Vendor Independence** | Switch backends without code changes |
| **Cost Efficiency** | Sampling & batching reduce data volume |
| **Developer Experience** | Simple APIs, minimal overhead |
| **Enterprise Ready** | Scales to large distributed systems |

### Use Cases
- Microservices monitoring & optimization
- Troubleshooting & root cause analysis
- SLA monitoring & capacity planning
- Application profiling & quality assurance

---

## Getting Started

1. **Choose Stack**: Language (Python/Java/Go/Node/.NET) + Backend (Jaeger/Prometheus/Datadog)
2. **Install SDK**: `pip install opentelemetry-api opentelemetry-sdk opentelemetry-exporter-otlp`
3. **Instrument App**: Add tracing to critical operations
4. **Deploy Collector**: Set up receiver/processor/exporter pipeline
5. **Configure Backend**: Connect to visualization platform
6. **Set Alerts**: Define thresholds and dashboards

---

## Conclusion

OpenTelemetry provides a standardized, vendor-agnostic observability framework essential for modern cloud-native applications. It enables organizations to gain deep insights into system behavior, reduce operational overhead, and make data-driven decisions about performance and reliability.