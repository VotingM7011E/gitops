# Monitoring + RabbitMQ (Dev) — GitOps (Argo CD + Helm)

> **Audience:** Team members onboarding to our dev cluster setup.
>
> **Goal:** Understand **what we deployed**, **why**, **how to verify it works**, and **how to onboard more services** (metrics + async messaging).

---

## TL;DR (What we built)

We deployed two platform components using **Argo CD + Helm**:

1. **Monitoring stack** (**`monitoring-dev`** namespace)
   - **Prometheus** (collects/stores metrics)
   - **Grafana** (visualizes metrics)
   - **Prometheus Operator** (auto-discovers scrape targets via `ServiceMonitor`)

2. **RabbitMQ** (**`rabbitmq-dev`** namespace)
   - Message broker for **asynchronous service-to-service communication** (publish/consume)
   - Exposes Prometheus metrics on **`/metrics`** (via `rabbitmq_prometheus` plugin)

This gives us:
- **Observability**: cluster + app metrics in Prometheus/Grafana
- **Loose coupling**: services communicate through RabbitMQ events

**Key docs:** kube‑prometheus‑stack uses Prometheus Operator + Grafana; Prometheus Operator discovers targets via `ServiceMonitor` CRDs; Argo CD supports Helm apps with `valueFiles` and sync options. [^kps] [^argocd-helm] [^argocd-sync]

---

## Repo convention (how it’s organized)

We follow our GitOps structure:

- `manifests/` → Argo CD `Application` YAMLs (and optional raw Kubernetes manifests)
- `environments/dev/*.yaml` → dev-specific values per app
- `<chart>/` → Helm charts stored in Git (e.g., `monitoring/`, `rabbitmq/`)

Argo CD renders Helm charts from Git and applies them to the cluster; values can be provided via `valueFiles`. [^argocd-helm]

---

## Part 1 — Monitoring stack (`monitoring-dev`)

### What is Prometheus?
Prometheus is a **metrics collector** and **time-series database**. It periodically **scrapes** metric endpoints and stores the results for querying with PromQL. [^kps]

### What is Grafana?
Grafana is the **visualization UI**. It queries Prometheus and renders dashboards/graphs. Grafana has built‑in support for a Prometheus data source (you configure its Prometheus URL). [^grafana-prom]

### What is Prometheus Operator?
Prometheus Operator is a Kubernetes operator that manages Prometheus instances and config. It **discovers scrape targets** primarily through `ServiceMonitor`/`PodMonitor` resources (CRDs). [^kps]

### Why did we enable Argo CD Server-Side Apply?
Large CRDs (common in monitoring stacks) can hit the Kubernetes annotation size limit with client-side apply. Argo CD supports `ServerSideApply=true`, which avoids “annotations too long” failures. [^argocd-sync]

---

## Part 2 — RabbitMQ (`rabbitmq-dev`)

RabbitMQ provides asynchronous communication:
- Producers publish messages/events
- Consumers process them later

### Ports we expose (service `rabbitmq`)
- **5672/TCP**: AMQP (apps connect here)
- **15672/TCP**: Management UI
- **15692/TCP**: Prometheus metrics endpoint (HTTP) at **`/metrics`**

RabbitMQ’s built-in Prometheus support comes from the `rabbitmq_prometheus` plugin and exposes metrics in Prometheus format. [^rabbitmq-prom]

---

## How to verify everything works (the “checklist”)

### 1) Pods running

```bash
kubectl get pods -n monitoring-dev
kubectl get pods -n rabbitmq-dev
```

Expected:
- `monitoring-dev`: Prometheus, Grafana, operator, node-exporter, kube-state-metrics, etc.
- `rabbitmq-dev`: RabbitMQ pod Running

### 2) Prometheus is scraping RabbitMQ (source of truth)

Port-forward Prometheus UI:

```bash
kubectl -n monitoring-dev port-forward svc/monitoring-dev-kube-promet-prometheus 9090:9090
```

Open: http://localhost:9090 → **Status → Targets**

Expected:
- You see a target like `serviceMonitor/rabbitmq-dev/rabbitmq/0` and it is **UP**.

Why this works: Prometheus Operator generates scrape config from `ServiceMonitor`. [^kps]

### 3) Grafana can query Prometheus

Port-forward Grafana:

```bash
kubectl -n monitoring-dev port-forward svc/monitoring-dev-grafana 3000:80
```

Open: http://localhost:3000

In Grafana:
- **Explore → Data source: Prometheus**
- Run:

```promql
up
```

You should see many targets with value `1`.

**Important:** In Grafana’s Prometheus datasource config, do **not** use `localhost:9090` (that would point to Grafana itself). Use the in-cluster service URL:

```
http://monitoring-dev-kube-promet-prometheus:9090
```

Grafana’s own docs explain this “localhost in containers” pitfall. [^grafana-prom]

### 4) RabbitMQ metrics exist

In Grafana Explore, run:

```promql
rabbitmq_build_info
```

If you see a result series, you’re reading real RabbitMQ metrics.

You can also hit the endpoint directly:

```bash
kubectl -n rabbitmq-dev port-forward svc/rabbitmq 15692:15692
curl -s http://localhost:15692/metrics | head
```

RabbitMQ documents this Prometheus endpoint behavior. [^rabbitmq-prom]

---

## How monitoring discovery works (and the “release label” gotcha)

### How Prometheus decides what to scrape
In Operator-based setups, Prometheus does **not** scrape random services automatically. It scrapes what is described by `ServiceMonitor`/`PodMonitor` resources **that match Prometheus selectors**. [^kps]

### Our key gotcha: `release: monitoring-dev`
Our Prometheus instance was selecting only ServiceMonitors labeled with:

```yaml
release: monitoring-dev
```

So RabbitMQ’s `ServiceMonitor` had to include:

```yaml
metadata:
  labels:
    release: monitoring-dev
```

Once added, RabbitMQ appeared as an **UP** target.

---

## Pattern to monitor ANY service (copy/paste guide)

To monitor another service (MeetingService, ElectionService, Todo, etc.), you need 3 things:

### Step A — Expose metrics from the service
Your app must expose metrics (commonly `/metrics`).

### Step B — Expose a Kubernetes Service port named `metrics`
Example service port snippet:

```yaml
ports:
  - name: metrics
    port: 9091
    targetPort: 9091
```

### Step C — Create a ServiceMonitor
Template example (place in the service Helm chart under `templates/servicemonitor.yaml`):

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: {{ .Release.Name }}
  labels:
    release: monitoring-dev
spec:
  selector:
    matchLabels:
      app: {{ .Release.Name }}
  namespaceSelector:
    matchNames:
      - {{ .Release.Namespace }}
  endpoints:
    - port: metrics
      path: /metrics
      interval: 30s
```

Then verify:
- Prometheus Targets shows it as **UP**
- Grafana query `up{job="<jobname>"}` returns `1`

---

## Using RabbitMQ for inter-service communication (practical next step)

RabbitMQ gives you async messaging:

- **MeetingService publishes** event `meeting.created`
- **ElectionService consumes** and reacts

### Cluster DNS name
From any namespace:

```
rabbitmq.rabbitmq-dev.svc.cluster.local
```

### Connection values to put in service dev values

```yaml
rabbitmq:
  host: rabbitmq.rabbitmq-dev.svc.cluster.local
  port: 5672
  username: <user>
  password: <pass>
  exchange: events
  routingKeyMeetingCreated: meeting.created
```

### Inject env vars in Helm Deployment

```yaml
env:
  - name: RABBITMQ_HOST
    value: {{ .Values.rabbitmq.host | quote }}
  - name: RABBITMQ_PORT
    value: {{ .Values.rabbitmq.port | quote }}
  - name: RABBITMQ_USER
    value: {{ .Values.rabbitmq.username | quote }}
  - name: RABBITMQ_PASS
    value: {{ .Values.rabbitmq.password | quote }}
  - name: RABBITMQ_EXCHANGE
    value: {{ .Values.rabbitmq.exchange | quote }}
  - name: RABBITMQ_ROUTINGKEY_MEETING_CREATED
    value: {{ .Values.rabbitmq.routingKeyMeetingCreated | quote }}
```

Once services publish/consume messages, your Grafana RabbitMQ dashboard becomes “alive” with throughput/queue behavior.

---

## Security note (recommended): Sealed Secrets for credentials

Kubernetes Secrets are **base64-encoded** and stored **unencrypted by default** unless encryption-at-rest is configured. [^k8s-secrets]

For GitOps, prefer **Sealed Secrets**:
- Store encrypted secrets in Git
- Controller decrypts inside the cluster

Argo CD explicitly recommends “destination cluster secret management” approaches like Sealed Secrets (rather than injecting secrets at render time). [^argocd-secrets]  
Sealed Secrets workflow (create Secret → `kubeseal` → commit SealedSecret) is documented by the project. [^sealed-secrets]

---

## Troubleshooting (quick fixes)

### RabbitMQ not appearing in Prometheus Targets
1. Confirm `ServiceMonitor` exists in `rabbitmq-dev`
2. Confirm it has `release: monitoring-dev`
3. Confirm RabbitMQ service has `port: metrics` and path `/metrics`
4. Check Prometheus UI → **Status → Targets** for why it’s missing

### Grafana Explore shows no “Prometheus” datasource
- Ensure Grafana datasource is configured to:
  - `http://monitoring-dev-kube-promet-prometheus:9090`
- Do not use `localhost:9090` in containers. [^grafana-prom]

---

## References

[^argocd-helm]: Argo CD Helm support (Applications, valueFiles): https://argo-cd.readthedocs.io/en/stable/user-guide/helm/
[^argocd-sync]: Argo CD sync options (incl. ServerSideApply): https://argo-cd.readthedocs.io/en/stable/user-guide/sync-options/
[^kps]: kube-prometheus-stack overview (Prometheus Operator stack): https://unstructured-io.github.io/prometheus-community-helm-charts/
[^grafana-prom]: Grafana Prometheus datasource configuration (URL and localhost note): https://grafana.com/docs/grafana/latest/datasources/prometheus/configure/
[^rabbitmq-prom]: RabbitMQ monitoring with Prometheus/Grafana (`rabbitmq_prometheus`, `/metrics`): https://www.rabbitmq.com/docs/prometheus
[^sealed-secrets]: Sealed Secrets project README (workflow): https://github.com/bitnami-labs/sealed-secrets
[^argocd-secrets]: Argo CD secret management guidance (recommends Sealed Secrets approach): https://argo-cd.readthedocs.io/en/stable/operator-manual/secret-management/
[^k8s-secrets]: Kubernetes good practices for Secrets (base64, encryption-at-rest): https://kubernetes.io/docs/concepts/security/secrets-good-practices/

