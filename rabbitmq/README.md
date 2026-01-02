# RabbitMQ Integration (Fast Track) — MeetingService ↔ ElectionService

> **Purpose:** Get reliable inter-service messaging working **quickly** with minimal code and minimal new infrastructure.
> 
> **Scope:** This guide focuses on **application integration** (publish/consume) and **verification**. It assumes RabbitMQ + monitoring are already deployed in the cluster.

---

## 1) Quick mental model (what you need to know)

RabbitMQ uses the AMQP model: producers publish to an **exchange**, which routes messages to **queues** (consumers read from queues). You generally **publish to an exchange**, not directly to a queue. [^rmq-tut]  
Connection settings are conveniently represented as a single AMQP URI: `amqp://user:pass@host:port/vhost`. [^rmq-uri]

---

## 2) Integration strategy (painless + consistent)

### Recommended minimal pattern
- Use **one shared exchange**: `events` (type: `topic`) [^rmq-tut]
- Each service owns **one durable queue** (e.g., `election-service`) and binds routing keys it cares about (e.g., `agenda.election.*`). [^rmq-tut]
- MeetingService **publishes** events when state changes.
- ElectionService **consumes** events and performs its business logic.

### Why this works well under time pressure
- No synchronous coupling between services.
- Easy to extend with more events/services later.
- Simple to debug in RabbitMQ UI and Prometheus/Grafana.

---

## 3) Required configuration (env vars only)

### Kubernetes DNS for RabbitMQ
In Kubernetes, Services are resolvable with the name format:

`<service>.<namespace>.svc.cluster.local` [^k8s-dns]

So RabbitMQ is reachable at:
- `rabbitmq.rabbitmq-dev.svc.cluster.local:5672`

### Standardize to one variable (best)
Set this in both services:

```bash
AMQP_URL=amqp://USER:PASSWORD@rabbitmq.rabbitmq-dev.svc.cluster.local:5672/
MQ_EXCHANGE=events
SERVICE_NAME=meeting-service   # or election-service
```

The URI format is standardized by RabbitMQ so you don’t need separate host/port/user/pass variables unless you prefer them. [^rmq-uri]

---

## 4) Add the smallest possible RabbitMQ helper (shared module)

> **Python services:** We use `pika` (RabbitMQ’s common Python AMQP client; also used in official tutorials). [^rmq-tut]

### Step 4.1 Add dependency
Add to each service requirements:

```txt
pika==1.3.2
```

### Step 4.2 Create `mq.py` (same file in both repos)
Create `app/src/mq.py` (or equivalent) with:

```python
import os, json, uuid
from datetime import datetime, timezone
import pika

EXCHANGE = os.getenv("MQ_EXCHANGE", "events")
EXCHANGE_TYPE = os.getenv("MQ_EXCHANGE_TYPE", "topic")  # topic recommended

def _conn():
    return pika.BlockingConnection(pika.URLParameters(os.environ["AMQP_URL"]))

def publish_event(routing_key: str, data: dict, *, event_version: int = 1):
    """Publish one JSON event to the exchange."""
    connection = _conn()
    ch = connection.channel()

    ch.exchange_declare(exchange=EXCHANGE, exchange_type=EXCHANGE_TYPE, durable=True)

    event = {
        "event_type": routing_key,
        "event_version": event_version,
        "event_id": str(uuid.uuid4()),
        "timestamp": datetime.now(timezone.utc).isoformat(),
        "producer": os.getenv("SERVICE_NAME", "unknown-service"),
        "data": data,
    }

    ch.basic_publish(
        exchange=EXCHANGE,
        routing_key=routing_key,
        body=json.dumps(event).encode("utf-8"),
        properties=pika.BasicProperties(
            content_type="application/json",
            delivery_mode=2,  # persistent
        ),
    )
    connection.close()

def start_consumer(*, queue: str, bindings: list[str], on_event):
    """Start a background consumer thread; calls on_event(event_dict)."""
    import threading

    def _run():
        connection = _conn()
        ch = connection.channel()

        ch.exchange_declare(exchange=EXCHANGE, exchange_type=EXCHANGE_TYPE, durable=True)
        ch.queue_declare(queue=queue, durable=True)

        for rk in bindings:
            ch.queue_bind(queue=queue, exchange=EXCHANGE, routing_key=rk)

        def callback(ch_, method, properties, body):
            try:
                event = json.loads(body.decode("utf-8"))
                on_event(event)
                ch_.basic_ack(delivery_tag=method.delivery_tag)
            except Exception:
                # keep it simple for the course: requeue on error
                ch_.basic_nack(delivery_tag=method.delivery_tag, requeue=True)

        ch.basic_consume(queue=queue, on_message_callback=callback, auto_ack=False)
        ch.start_consuming()

    t = threading.Thread(target=_run, daemon=True)
    t.start()
    return t
```

**Notes:**
- Uses an exchange + routing keys (topic pattern), matching RabbitMQ’s recommended messaging model. [^rmq-tut]
- Uses a single AMQP URL string, matching the RabbitMQ URI spec. [^rmq-uri]

---

## 5) MeetingService: publish events (2 calls)

### Event names (minimum viable)
- `agenda.election.create`
- `agenda.election.start`

### Where to publish
In the code path where MeetingService:
1) moves to an election agenda item → publish `agenda.election.create`
2) starts the election voting → publish `agenda.election.start`

Example (Flask route or service function):

```python
from mq import publish_event

publish_event("agenda.election.create", {
    "meeting_id": meeting_id,
    "agenda_item_id": agenda_item_id,
    "title": title,
})

publish_event("agenda.election.start", {
    "meeting_id": meeting_id,
    "agenda_item_id": agenda_item_id,
})
```

---

## 6) ElectionService: consume events (one consumer thread)

ElectionService should own a durable queue, e.g. `election-service`, and bind it to election routing keys.

### Step 6.1 Add handler
In `app/src/app.py` (or your Flask startup module):

```python
import os
from flask import Flask
from mq import start_consumer

app = Flask(__name__)

def handle_event(event: dict):
    et = event.get("event_type")
    data = event.get("data", {})

    if et == "agenda.election.create":
        # TODO: create election item + open nominations
        print("[ElectionService] create election", data)

    elif et == "agenda.election.start":
        # TODO: close nominations + start voting
        print("[ElectionService] start election", data)

    else:
        print("[ElectionService] ignoring", et)

start_consumer(
    queue=os.getenv("MQ_QUEUE", "election-service"),
    bindings=os.getenv("MQ_BINDINGS", "agenda.election.*").split(","),
    on_event=handle_event,
)

@app.get("/health")
def health():
    return {"status": "ok"}, 200
```

### Step 6.2 Add env vars

```bash
MQ_QUEUE=election-service
MQ_BINDINGS=agenda.election.*
```

Binding with wildcards is a standard topic-exchange pattern. [^rmq-tut]

---

## 7) Local test loop (no new containers)

### Option A — Use cluster RabbitMQ via port-forward

```bash
kubectl -n rabbitmq-dev port-forward svc/rabbitmq 5672:5672
```

Then:

```bash
export AMQP_URL='amqp://USER:PASSWORD@localhost:5672/'
export MQ_EXCHANGE='events'
export SERVICE_NAME='election-service'
export MQ_QUEUE='election-service'
export MQ_BINDINGS='agenda.election.*'

flask --app app/src/app.py run
```

Run MeetingService locally and hit the endpoint that publishes the event.

### Option B — Observe via RabbitMQ Management UI
Port-forward:

```bash
kubectl -n rabbitmq-dev port-forward svc/rabbitmq 15672:15672
```

Open http://localhost:15672 and watch:
- queues
- message rates
- consumers

The management UI is part of RabbitMQ’s management image tags and is useful for debugging. [^rmq-docker]

---

## 8) Verify with Monitoring (Prometheus + Grafana)

RabbitMQ exposes metrics via the Prometheus plugin at `/metrics`. [^rmq-prom]

### Prometheus (truth source)
- Prometheus UI → **Status → Targets** shows the RabbitMQ ServiceMonitor target **UP**.

### Grafana (visualize)
In Grafana Explore (Prometheus datasource):

```promql
up{job="rabbitmq"}
```

And a proof-of-life metric:

```promql
rabbitmq_build_info
```

Grafana datasource configuration should point to the Prometheus service URL, not `localhost`, because `localhost` inside containers refers to Grafana itself. [^grafana-prom]

---

## 9) Common gotchas (save hours)

1) **React should not connect directly to RabbitMQ.** Browsers don’t speak AMQP 0‑9‑1 by default; keep AMQP in backend services.
2) **Use one queue per service** (consumer group). Don’t create one queue per producer.
3) **Ack messages** only after successful handling.
4) If messages are not delivered: verify exchange name, routing key, and queue bindings.

---

## 10) Minimal roadmap (aligned with team plan)

- MeetingService → `meeting.created` → PermissionService grants manager
- MeetingService → `agenda.election.create` → ElectionService sets up nominations
- MeetingService → `agenda.election.start` → ElectionService starts voting
- ElectionService → `voting.create` → VotingService creates ballot
- VotingService → `voting.completed` → ElectionService finalizes results

---

## References

[^rmq-uri]: RabbitMQ AMQP URI specification (`amqp://user:pass@host:port/vhost`). https://www.rabbitmq.com/docs/uri-spec
[^rmq-tut]: RabbitMQ tutorials explain publish/subscribe model (publish to exchanges, bind queues). https://www.rabbitmq.com/tutorials/tutorial-three-python
[^rmq-prom]: RabbitMQ Prometheus monitoring guide (`rabbitmq_prometheus` plugin, `/metrics`). https://www.rabbitmq.com/docs/prometheus
[^grafana-prom]: Grafana Prometheus datasource config (note on container `localhost`). https://grafana.com/docs/grafana/latest/datasources/prometheus/configure/
[^k8s-dns]: Kubernetes DNS for Services and Pods (service DNS format & namespace behavior). https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/
[^rmq-docker]: RabbitMQ Docker official image (management UI, tags, ports). https://hub.docker.com/_/rabbitmq
