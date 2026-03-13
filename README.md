# BuffGate

[![Go](https://img.shields.io/badge/Go-1.16+-00ADD8?style=flat&logo=go)](https://golang.org)
[![MongoDB](https://img.shields.io/badge/MongoDB-3.6+-47A248?style=flat&logo=mongodb)](https://www.mongodb.com)
[![License](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

> **The Eyes and Ears of Your AI Agents**
>
> Because you can't fix what you can't see.

---

## Here's the Thing About AI Agents...

You've built an agent. It's clever, it's capable, it's shipping code or answering customer queries or whatever you built it to do. Cool.

But here's the uncomfortable truth: **most of the time, you have no idea what it's actually doing.**

Did it call the right tool? Did it waste 3 API calls before getting it right? Is it hallucinating on edge cases? Which prompts work better than others?

If you're honest with yourself, you're mostly flying blind. You ship a prompt change, cross your fingers, and hope things get better.

**That's where BuffGate comes in.**

---

## What BuffGate Actually Does

Think of BuffGate as the flight recorder for your AI agents. Every time your agent:
- 📞 Makes a tool call
- 🤔 Makes a decision
- 💥 Hits an error
- ✅ Completes a milestone
- 👍 Gets user feedback

...BuffGate captures it, stores it, and makes it queryable for analysis.

It's the difference between:
```
"I think the agent is doing okay... maybe?"
```

and

```
"In the last 10K runs, tool_call success rate is 94.2%,
average latency is 340ms, and error spike correlates
with prompts longer than 2000 tokens."
```

---

## Why This Matters for AI Infrastructure

### The Missing Piece in Your AI Stack

Look at your typical AI infrastructure:

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   LLM API   │     │  Vector DB  │     │  Agent SDK  │
│  (the brain)│     │ (the memory)│     │(the actions)│
└─────────────┘     └─────────────┘     └─────────────┘
       ▲                   ▲                   ▲
       │                   │                   │
       └───────────────────┴───────────────────┘
                           │
                    Where's the observability?
```

You've got the brain (LLM), the memory (vector DB), and the action layer (tools/APIs). But where's the **observation layer**?

Without it, you're basically driving blindfolded. Every prompt change is a gamble. Every new tool integration is a mystery.

**BuffGate fills that gap.**

### The Feedback Loop You've Been Missing

Here's what a proper AI development loop looks like:

```
┌─────────────────────────────────────────────────────────┐
│                                                          │
│    Agent Runs ──▶ Events Collected ──▶ Data Stored      │
│         ▲                                      │         │
│         │                                      ▼         │
│    Better Agent ◀── Prompt Optimized ◀── Analysis       │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

Right now, most teams are stuck at "Agent Runs" with maybe a few log lines here and there. BuffGate gives you the full loop.

### Real-World Impact

| Without BuffGate | With BuffGate |
|------------------|---------------|
| "Why is the agent slow?" 🤷 | "47% of latency comes from file_read operations on files >1MB" |
| "Did the new prompt help?" 🤷 | "Error rate dropped from 12% to 3.2% after the change" |
| "What tools does it use most?" 🤷 | "Top 3 tools: search (42%), read (31%), write (19%)" |
| "Is this even worth the cost?" 🤷 | "Average tokens/run: 1,247. ROI positive after 2 weeks" |

---

## The Technical Edge

### It's Fast. Like, Really Fast.

We're not talking about "it works fine for my side project" fast. We're talking about:

- **Double-buffered writes**: Ingestion doesn't wait for database writes
- **Object pooling**: GC pressure? What GC pressure?
- **Adaptive batching**: Low traffic = instant writes. High traffic = bulk mode.
- **10K+ events/sec**: On modest hardware. Without breaking a sweat.

```go
// This is how we handle incoming events
stomach := make(chan *ClientEvent, 10240)  // Yeah, 10K buffer

// Two buffers, one for reading (flush to DB), one for writing (new events)
r_buff, w_buff := make([]*ClientEvent, 0, 128), make([]*ClientEvent, 0, 128)

// Swap them when it's time to flush
r_buff, w_buff = w_buff, r_buff  // Boom. Zero copy.
```

### It Plays Nice With MongoDB

MongoDB gets a bad rap sometimes, but for event logs? It's actually perfect:
- Schema-flexible (your events will evolve)
- Queryable (need all errors from last week? Easy)
- Horizontally scalable (sharding when you need it)

BuffGate uses MongoDB's bulk write API, which means:
- 1000 events in one round-trip
- Ordered or unordered inserts (your call)
- Automatic retry on network blips

### It's Just HTTP

No gRPC, no message queues, no service mesh required. Just a simple HTTP POST:

```bash
curl -X POST http://buffgate:8100/collect -d '{"en": "click", ...}'
```

This means:
- Works from any language
- Easy to debug
- Plays nice with load balancers
- No vendor lock-in

---

## The Bigger Picture

BuffGate isn't trying to be everything. It does one thing and does it well: **collect events at scale**.

But here's where it fits in your stack:

```
Your Agents ──▶ BuffGate ──▶ MongoDB ──▶ ???
                                        │
                    ┌───────────────────┴───────────────────┐
                    │                                       │
                    ▼                                       ▼
            ┌──────────────┐                      ┌──────────────┐
            │   Your       │                      │   ML/AI      │
            │   Dashboards │                      │   Analytics  │
            └──────────────┘                      └──────────────┘
                    │                                       │
                    └───────────────────┬───────────────────┘
                                        │
                                        ▼
                              ┌──────────────────┐
                              │   Better Prompts │
                              │   Better Agents  │
                              │   Better Outcomes│
                              └──────────────────┘
```

The "???" is where the magic happens — your analysis tools, your ML pipelines, your dashboards. BuffGate just makes sure you have the data when you need it.

---

## Getting Started (It's Pretty Simple)

### What You'll Need

- Go 1.16+ (or just use Docker)
- MongoDB 3.6+ (local or Atlas, doesn't matter)
- 5 minutes of your time

### Quick Install

```bash
# Clone it
git clone https://github.com/lightlfyan/buffgate.git
cd buffgate

# Grab dependencies
go mod download

# Create your config
mkdir -p config
cat > config/config.json << 'EOF'
{
  "port": ":8100",
  "mgo_url": "mongodb://localhost:27017/agent_events"
}
EOF

# Run it
go run example_server.go
```

That's it. You're now collecting events.

### Docker? Sure.

```bash
docker build -t buffgate .
docker run -p 8100:8100 buffgate
```

---

## The Event Schema (Keep It Simple)

We designed the schema to be flexible enough for anything but structured enough to be useful:

```json
{
  "v": "1.0",           // Schema version (future-proofing)
  "an": "my-agent",     // Agent name (which agent is this?)
  "ds": "cli",          // Data source (where did it come from?)
  "av": "2.0.0",        // Agent version (A/B testing, anyone?)
  "t": "tool_call",     // Event type (what category?)
  "z": "uuid-here",     // Nonce (for signature verification)
  "sgn": "hmac-sha256", // Signature (because security)
  "cid": "conv-123",    // Conversation ID (tie events together)
  "uid": "user-456",    // User ID (who's using the agent?)
  "en": "file_write",   // Event name (what happened?)
  "ev": {               // Event value (anything you want)
    "file": "app.ts",
    "tokens": "847",
    "success": "true"
  }
}
```

The `ev` field is where the magic happens — throw whatever key-value pairs you want in there. We'll store it all.

---

## Sending Events

### The Easy Way (curl)

```bash
curl -X POST http://localhost:8100/collect \
  -H "Content-Type: application/json" \
  -d '{
    "v": "1.0",
    "an": "my-awesome-agent",
    "ds": "cli",
    "av": "1.0.0",
    "t": "tool_call",
    "z": "random-nonce-123",
    "sgn": "your-signature-here",
    "cid": "conversation-456",
    "uid": "user-789",
    "en": "file_read",
    "ev": {
      "file": "src/main.go",
      "success": "true",
      "latency_ms": "42"
    }
  }'
```

Response: `ok` (or `payload error` if you messed up the JSON)

### From Your Python Agent

```python
import requests
import uuid

def track_event(agent_name, event_type, event_name, conv_id, payload=None):
    """Literally just send events to BuffGate."""
    requests.post("http://buffgate:8100/collect", json={
        "v": "1.0",
        "an": agent_name,
        "ds": "sdk",
        "av": "1.0.0",
        "t": event_type,
        "z": str(uuid.uuid4()),
        "sgn": "your-signature",  # or compute it properly
        "cid": conv_id,
        "en": event_name,
        "ev": payload or {}
    })

# Use it
track_event(
    agent_name="code-bot",
    event_type="tool_call",
    event_name="write_file",
    conv_id="session-123",
    payload={"file": "app.py", "tokens": "234"}
)
```

### From Your TypeScript/Node Agent

```typescript
async function trackEvent(
  agentName: string,
  eventType: string,
  eventName: string,
  convId: string,
  payload?: Record<string, string>
) {
  await fetch("http://buffgate:8100/collect", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      v: "1.0",
      an: agentName,
      ds: "sdk",
      av: "1.0.0",
      t: eventType,
      z: crypto.randomUUID(),
      sgn: "your-signature",
      cid: convId,
      en: eventName,
      ev: payload,
    }),
  });
}
```

---

## What's Next?

BuffGate is just the foundation. Here's where we're headed:

### ✅ What's Working Now
- High-throughput event collection (10K+ events/sec)
- MongoDB persistence with bulk writes
- Double-buffer architecture (ingestion ≠ blocking)
- Simple HTTP API

### 🚧 What's Coming
- **Prometheus metrics** — because you want Grafana dashboards
- **Health endpoints** — `/health`, `/ready`, the usual suspects
- **Query API** — fetch aggregated stats without hitting MongoDB directly
- **Export connectors** — send data to BigQuery, Snowflake, or wherever

### 🔮 What We're Thinking About
- **ML-based anomaly detection** — "hey, your agent's acting weird"
- **Prompt optimization suggestions** — "this prompt causes 3x more errors"
- **Real-time alerting** — Slack/Teams when things go sideways

Got ideas? [Open an issue](https://github.com/lightlfyan/buffgate/issues). We're listening.

---

## FAQ

**Q: Why MongoDB? Why not Kafka/Pulsar/ClickHouse?**

A: MongoDB is good enough for most use cases, and it's queryable. Kafka is great but overkill for event collection. ClickHouse is fast but another thing to manage. Start simple, optimize later.

**Q: What about data retention?**

A: MongoDB TTL indexes are your friend. Set it and forget it.

```javascript
db.clientlog.createIndex(
  { "timestamp": 1 },
  { expireAfterSeconds: 2592000 }  // 30 days
)
```

**Q: Can I use this in production?**

A: People do. But test it first. It's MIT licensed — use at your own risk.

**Q: How do I analyze the data?**

A: However you want. MongoDB aggregation pipeline, export to Pandas, connect your BI tool, build a custom dashboard. The data is yours.

---

## Contributing

Found a bug? Have a feature idea? PRs welcome.

1. Fork it
2. Create a feature branch
3. Make your changes
4. Submit a PR

We'll review it.

---

## License

MIT. Use it however you want. Just don't blame us if something breaks.

---

**[中文文档](README_CN.md)** | [Technical Docs](TECHNICAL_DOC.md)

---

*Built with ☕ by engineers who got tired of guessing why their agents were failing.*
