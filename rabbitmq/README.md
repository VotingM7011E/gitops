# VotingService + RabbitMQ (Fast Track)

> **Audience:** Team members integrating RabbitMQ quickly into the existing VotingService Flask code (`app.py`).
>
> **Goal:** Move *poll creation* from “HTTP-only” to also support *RabbitMQ-triggered* creation, with minimal refactor and minimal time.

---

## 0) Why RabbitMQ here?

RabbitMQ lets services communicate asynchronously: producers **publish** events, consumers **consume** them later (decoupling services). RabbitMQ’s AMQP model is *publish → exchange → route → queue → consume*. [^rmq-tut]  
Connection details are commonly carried as a single URI string: `amqp://user:pass@host:port/vhost` (standardized by RabbitMQ). [^rmq-uri]

In our architecture, **VotingService** should create polls when another service (e.g., ElectionService) requests one. In the current code, this is already noted:

> `# POST /polls/ - Create a new poll`  
> `# Should be moved to consume RabbitMQ messages.`  

(See `create_poll()` in `app.py`.)

---

## 1) What exists today (in `app.py`)

Your VotingService already has a complete “create poll” implementation:
- Validates request JSON (`meeting_id`, `pollType`, `options`)
- Writes `Poll` + `PollOption` rows via SQLAlchemy
- Returns `poll.uuid` and option list

Today it’s triggered via HTTP:
- `POST /polls/` (creates poll)

### The minimal change we want
Keep the same logic, but make it callable from:
1) HTTP (existing behavior)
2) RabbitMQ consumer handler (new behavior)

---

## 2) Required environment variables (no new images)

### Use a single connection variable: `AMQP_URL`
RabbitMQ recommends passing connection parameters as a single AMQP URI. [^rmq-uri]

In Kubernetes, your RabbitMQ Service DNS will be:

`rabbitmq.rabbitmq-dev.svc.cluster.local` (Kubernetes service DNS format) [^k8s-dns]

So you typically set:

```bash
AMQP_URL=amqp://USER:PASSWORD@rabbitmq.rabbitmq-dev.svc.cluster.local:5672/
MQ_EXCHANGE=events
SERVICE_NAME=voting-service
MQ_QUEUE=voting-service
MQ_BINDINGS=voting.create
```

- `MQ_EXCHANGE`: shared topic exchange name
- `MQ_QUEUE`: durable queue name owned by this service
- `MQ_BINDINGS`: routing keys this service consumes (comma-separated)

---

## 3) Add RabbitMQ dependency

Add to your requirements (where you keep Flask deps) e.g. `requirements.txt`:

```txt
pika==1.3.2
```

RabbitMQ’s tutorials use Pika in Python examples and the AMQP publish/consume model. [^rmq-tut]

---

## 4) Add a tiny RabbitMQ helper module (`mq.py`)

Create a file next to your Flask code, for example `mq.py` in the same folder as `app.py`.

```python
import os, json, uuid
from datetime import datetime, timezone
import pika

EXCHANGE = os.getenv("MQ_EXCHANGE", "events")
EXCHANGE_TYPE = os.getenv("MQ_EXCHANGE_TYPE", "topic")

def _conn():
    return pika.BlockingConnection(pika.URLParameters(os.environ["AMQP_URL"]))

def publish_event(routing_key: str, data: dict, *, event_version: int = 1):
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
            delivery_mode=2,
        ),
    )
    connection.close()

def start_consumer(*, queue: str, bindings: list[str], on_event):
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
                ch_.basic_nack(delivery_tag=method.delivery_tag, requeue=True)

        ch.basic_consume(queue=queue, on_message_callback=callback, auto_ack=False)
        ch.start_consuming()

    t = threading.Thread(target=_run, daemon=True)
    t.start()
    return t
```

This matches RabbitMQ’s “publish to exchange + routing key” model and uses the standardized AMQP URI. [^rmq-tut] [^rmq-uri]

---

## 5) Minimal refactor in `app.py` (make poll creation reusable)

### Step 5.1 Extract “create poll” logic into a pure function
Right now `create_poll()` reads from `request.get_json()` and returns Flask responses. That’s hard to reuse from a message consumer.

Add a helper **inside** `app.py`:

```python
def create_poll_from_vote_data(vote_data: dict):
    # Validate required fields
    meeting_id = vote_data.get("meeting_id")
    poll_type = vote_data.get("pollType")
    options = vote_data.get("options", [])

    if not meeting_id:
        raise ValueError("Missing 'meeting_id'")
    if poll_type not in ["single", "ranked"]:
        raise ValueError("Invalid 'pollType'. Must be 'single' or 'ranked'")
    if not options or len(options) < 2:
        raise ValueError("At least 2 options are required")

    poll = Poll(meeting_id=meeting_id, poll_type=poll_type)
    db.session.add(poll)
    db.session.flush()  # get poll.id

    for index, option_value in enumerate(options):
        db.session.add(PollOption(
            poll_id=poll.id,
            option_value=option_value,
            option_order=index,
        ))

    db.session.commit()

    return {
        "uuid": str(poll.uuid),
        "meeting_id": poll.meeting_id,
        "pollType": poll.poll_type,
        "options": options,
    }
```

### Step 5.2 Make the HTTP endpoint call the helper
Replace the body of `create_poll()` with:

```python
@blueprint.route("/polls/", methods=["POST"])
def create_poll():
    data = request.get_json()
    if not data or "vote" not in data:
        return jsonify({"error": "Missing 'vote' in request body"}), 400

    try:
        result = create_poll_from_vote_data(data["vote"])
        return jsonify(result), 200
    except ValueError as e:
        return jsonify({"error": str(e)}), 400
```

Now you can call the same logic from RabbitMQ.

---

## 6) Add the RabbitMQ consumer to VotingService (fastest: background thread)

> **Course-speed approach:** start a background consumer thread when Flask starts.
> In production you might run consumers as separate workers, but this is quickest for 6-day timelines.

Add near app startup (after `db.init_app(app)`), for example:

```python
from mq import start_consumer

def on_event(event: dict):
    # event envelope: {event_type, data, ...}
    et = event.get("event_type")
    data = event.get("data", {})

    if et == "voting.create":
        # IMPORTANT: we need app context for db.session
        with app.app_context():
            # expected payload: {"vote": {...}} OR just vote_data
            vote_data = data.get("vote") or data
            create_poll_from_vote_data(vote_data)

# Start consumer thread (after app exists)
start_consumer(
    queue=os.getenv("MQ_QUEUE", "voting-service"),
    bindings=os.getenv("MQ_BINDINGS", "voting.create").split(","),
    on_event=on_event,
)
```

### Message format expected
For compatibility with your current HTTP request schema (`{"vote": {...}}`), send events like:

```json
{
  "event_type": "voting.create",
  "data": {
    "vote": {
      "meeting_id": "...",
      "pollType": "single",
      "options": ["Alice", "Bob"]
    }
  }
}
```

---

## 7) How ElectionService (or MeetingService) should request a vote

When ElectionService is ready to create a ballot, it publishes:

- Exchange: `events`
- Routing key: `voting.create`
- Data: your existing poll schema

Any service can publish using the same AMQP URL and exchange model. RabbitMQ tutorials cover this publish pattern. [^rmq-tut]

---

## 8) Local smoke test (no Kubernetes changes)

### Step 1 — port-forward RabbitMQ AMQP

```bash
kubectl -n rabbitmq-dev port-forward svc/rabbitmq 5672:5672
```

### Step 2 — run VotingService locally

```bash
export AMQP_URL='amqp://USER:PASSWORD@localhost:5672/'
export MQ_EXCHANGE='events'
export MQ_QUEUE='voting-service'
export MQ_BINDINGS='voting.create'

python app.py
```

### Step 3 — publish a test message (quick python one-liner)

```bash
python - <<'PY'
import os, json, pika
os.environ['AMQP_URL']=os.environ.get('AMQP_URL')
conn=pika.BlockingConnection(pika.URLParameters(os.environ['AMQP_URL']))
ch=conn.channel()
ch.exchange_declare(exchange='events', exchange_type='topic', durable=True)
msg={"event_type":"voting.create","data":{"vote":{"meeting_id":"m1","pollType":"single","options":["A","B"]}}}
ch.basic_publish(exchange='events', routing_key='voting.create', body=json.dumps(msg).encode())
print('published')
conn.close()
PY
```

Then verify in DB (or via `GET /polls/<uuid>/` if available).

---

## 9) Verify with monitoring (optional but recommended)

RabbitMQ exposes `/metrics` using the Prometheus plugin and is meant to be monitored with Prometheus + Grafana. [^rmq-prom]

- Prometheus UI → **Status → Targets** should show RabbitMQ target **UP**.
- Grafana Explore:

```promql
up{job="rabbitmq"}
```

And a proof-of-life metric:

```promql
rabbitmq_build_info
```

Grafana datasource should point at Prometheus service URL (not localhost in container deployments). [^grafana-prom]

---

## 10) Common gotchas

- **Don’t connect React directly to RabbitMQ** (AMQP is backend-to-backend)
- **Ack only after DB commit** (avoid losing messages)
- Consider **idempotency** (messages can be retried → avoid duplicate polls)

---

## References

[^rmq-uri]: RabbitMQ URI specification (`amqp://user:pass@host:port/vhost`). https://www.rabbitmq.com/docs/uri-spec
[^rmq-tut]: RabbitMQ tutorials explain exchange→queue→consumer model (Python). https://www.rabbitmq.com/tutorials/tutorial-three-python
[^k8s-dns]: Kubernetes DNS for Services and Pods (service DNS format). https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/
[^rmq-prom]: RabbitMQ monitoring with Prometheus (`/metrics`). https://www.rabbitmq.com/docs/prometheus
[^grafana-prom]: Grafana Prometheus datasource configuration (localhost pitfall). https://grafana.com/docs/grafana/latest/datasources/prometheus/configure/
