# BuffGate

[![Go](https://img.shields.io/badge/Go-1.16+-00ADD8?style=flat&logo=go)](https://golang.org)
[![MongoDB](https://img.shields.io/badge/MongoDB-3.6+-47A248?style=flat&logo=mongodb)](https://www.mongodb.com)
[![License](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

> **Observability Infrastructure for AI Agents**
>
> Collect, Store, Analyze — The Foundation for Agent Behavior Optimization

BuffGate is a high-performance event collection service designed for the AI Agent era. It serves as the observability backbone in AI engineering harness, capturing runtime events that become the data source for agent behavior analysis, enabling continuous optimization of prompts and execution strategies.

---

## Why BuffGate?

In the age of AI Agents, understanding **what your agents do** is as important as **what they produce**. BuffGate provides:

| Challenge | Solution |
|-----------|----------|
| Agent behavior is opaque | Capture every action, decision, and outcome |
| Prompts need iteration | Ground optimization in real usage data |
| Strategies must evolve | Analyze patterns across thousands of runs |
| Infrastructure must scale | Handle high-throughput event streams effortlessly |

## Vision

```
┌─────────────────────────────────────────────────────────────────────┐
│                        AI Engineering Harness                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐     │
│   │  Agent   │    │  Agent   │    │  Agent   │    │  Agent   │     │
│   │    A     │    │    B     │    │    C     │    │    D     │     │
│   └────┬─────┘    └────┬─────┘    └────┬─────┘    └────┬─────┘     │
│        │               │               │               │            │
│        └───────────────┴───────┬───────┴───────────────┘            │
│                                │                                     │
│                                ▼                                     │
│                    ┌─────────────────────┐                          │
│                    │      BuffGate       │                          │
│                    │   Event Collector   │                          │
│                    └──────────┬──────────┘                          │
│                               │                                      │
└───────────────────────────────┼──────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         Data Layer                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   ┌──────────────┐                         ┌──────────────────┐     │
│   │   MongoDB    │  ◀── Event Stream ───▶  │  Analysis Engine │     │
│   │  (Storage)   │                         │  (Future)        │     │
│   └──────────────┘                         └──────────────────┘     │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       Optimization Loop                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐           │
│    │   Analyze   │───▶│   Refine    │───▶│   Deploy    │           │
│    │  Behavior   │    │   Prompts   │    │  Strategy   │           │
│    └─────────────┘    └─────────────┘    └─────────────┘           │
│           ▲                                        │                 │
│           └────────────────────────────────────────┘                 │
│                      Continuous Improvement                          │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

## Core Capabilities

### 🎯 Event Collection

Capture the full spectrum of agent runtime events:

```json
{
  "v": "1.0",
  "an": "code-assistant",
  "ds": "ide-plugin",
  "av": "2.1.0",
  "t": "tool_call",
  "cid": "session_abc123",
  "uid": "user_001",
  "en": "file_write",
  "ev": {
    "tool": "Write",
    "file": "src/main.go",
    "tokens_used": "847",
    "latency_ms": "234",
    "success": "true"
  }
}
```

### 📊 Behavior Analysis Ready

Structured event schema designed for downstream analytics:

| Event Type | Use Case |
|------------|----------|
| `tool_call` | Analyze tool usage patterns, identify frequent operations |
| `decision` | Track reasoning paths, evaluate decision quality |
| `error` | Identify failure modes, prioritize fixes |
| `milestone` | Measure task completion, success rates |
| `feedback` | Correlate user feedback with agent actions |

### ⚡ High Performance

- **Double Buffering** - Decouple ingestion from persistence
- **Object Pooling** - Minimize GC pressure
- **Adaptive Batching** - Optimize for both low and high load
- **Bulk Writes** - Maximize MongoDB throughput

## Architecture

```
                    Agent Runtime
                         │
         ┌───────────────┼───────────────┐
         │               │               │
         ▼               ▼               ▼
    ┌─────────┐    ┌─────────┐    ┌─────────┐
    │ Agent A │    │ Agent B │    │ Agent N │
    └────┬────┘    └────┬────┘    └────┬────┘
         │               │               │
         └───────────────┼───────────────┘
                         │ HTTP POST /collect
                         ▼
              ┌─────────────────────┐
              │   Gin HTTP Server   │
              │   (CORS Enabled)    │
              └──────────┬──────────┘
                         │
                         ▼
              ┌─────────────────────┐
              │   Giant Buffer      │
              │  ┌─────┐   ┌─────┐  │
              │  │w_buf│◀▶│r_buf│  │
              │  └─────┘   └─────┘  │
              │  + sync.Pool        │
              └──────────┬──────────┘
                         │
                         ▼
              ┌─────────────────────┐
              │      MongoDB        │
              │  DB: gsensor        │
              │  Collection: logs   │
              └─────────────────────┘
```

## Quick Start

### Install

```bash
git clone https://github.com/lightlfyan/buffgate.git
cd buffgate
go mod download
```

### Configure

Create `config/config.json`:

```json
{
  "port": ":8100",
  "mgo_url": "mongodb://localhost:27017/agent_events"
}
```

### Run

```bash
go run example_server.go
```

### Docker

```bash
docker build -t buffgate .
docker run -p 8100:8100 buffgate
```

## API Reference

### POST /collect

Submit an agent event.

**Request:**

```bash
curl -X POST http://localhost:8100/collect \
  -H "Content-Type: application/json" \
  -d '{
    "v": "1.0",
    "an": "my-agent",
    "ds": "cli",
    "av": "1.0.0",
    "t": "tool_call",
    "z": "nonce_abc123",
    "sgn": "hmac_sha256_signature",
    "cid": "conv_xyz789",
    "uid": "user_001",
    "en": "read_file",
    "ev": {
      "file": "src/app.ts",
      "success": "true",
      "latency_ms": "45"
    }
  }'
```

**Schema:**

| Field | JSON Key | Required | Description |
|-------|----------|:--------:|-------------|
| Version | `v` | ✓ | Schema version |
| Agent Name | `an` | ✓ | Agent identifier |
| Data Source | `ds` | ✓ | Origin (cli, api, plugin, etc.) |
| Agent Version | `av` | ✓ | Agent version |
| Type | `t` | ✓ | Event type |
| Nonce | `z` | ✓ | Request nonce |
| Signature | `sgn` | ✓ | Request signature |
| Conversation ID | `cid` | ✓ | Session/conversation identifier |
| User ID | `uid` | | Optional user identifier |
| Event | `en` | ✓ | Event name |
| Event Value | `ev` | | Arbitrary key-value payload |

**Response:**

| Status | Body |
|--------|------|
| 200 | `ok` |
| 400 | `payload error` |

## Integration Examples

### Python Agent SDK

```python
import requests
import json
import time
import hmac
import hashlib
import uuid

class BuffGateClient:
    def __init__(self, endpoint, agent_name, secret):
        self.endpoint = endpoint
        self.agent_name = agent_name
        self.secret = secret

    def emit(self, event_type, event_name, conversation_id, payload=None, user_id=None):
        nonce = str(uuid.uuid4())
        timestamp = str(int(time.time()))

        data = {
            "v": "1.0",
            "an": self.agent_name,
            "ds": "sdk",
            "av": "1.0.0",
            "t": event_type,
            "z": nonce,
            "sgn": self._sign(nonce, timestamp),
            "cid": conversation_id,
            "uid": user_id,
            "en": event_name,
            "ev": payload or {}
        }

        requests.post(f"{self.endpoint}/collect", json=data)

    def _sign(self, nonce, timestamp):
        msg = f"{nonce}:{timestamp}".encode()
        return hmac.new(self.secret.encode(), msg, hashlib.sha256).hexdigest()

# Usage
client = BuffGateClient("http://localhost:8100", "code-agent", "secret")

# Emit tool call event
client.emit(
    event_type="tool_call",
    event_name="file_read",
    conversation_id="conv_123",
    payload={"file": "main.go", "success": True, "tokens": 150}
)
```

### TypeScript Agent SDK

```typescript
interface AgentEvent {
  v: string;
  an: string;
  ds: string;
  av: string;
  t: string;
  z: string;
  sgn: string;
  cid: string;
  uid?: string;
  en: string;
  ev?: Record<string, string>;
}

class BuffGateClient {
  constructor(
    private endpoint: string,
    private agentName: string,
    private secret: string
  ) {}

  async emit(
    eventType: string,
    eventName: string,
    conversationId: string,
    payload?: Record<string, string>,
    userId?: string
  ): Promise<void> {
    const nonce = crypto.randomUUID();

    const event: AgentEvent = {
      v: "1.0",
      an: this.agentName,
      ds: "sdk",
      av: "1.0.0",
      t: eventType,
      z: nonce,
      sgn: await this.sign(nonce),
      cid: conversationId,
      uid: userId,
      en: eventName,
      ev: payload,
    };

    await fetch(`${this.endpoint}/collect`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(event),
    });
  }

  private async sign(nonce: string): Promise<string> {
    const key = await crypto.subtle.importKey(
      "raw",
      new TextEncoder().encode(this.secret),
      { name: "HMAC", hash: "SHA-256" },
      false,
      ["sign"]
    );
    const sig = await crypto.subtle.sign(
      "HMAC",
      key,
      new TextEncoder().encode(nonce)
    );
    return Array.from(new Uint8Array(sig))
      .map((b) => b.toString(16).padStart(2, "0"))
      .join("");
  }
}
```

## Roadmap

### Phase 1: Foundation ✅
- [x] High-performance event ingestion
- [x] MongoDB persistence
- [x] Double-buffer architecture

### Phase 2: Observability (Planned)
- [ ] Prometheus metrics export
- [ ] Health check endpoints
- [ ] Real-time dashboard

### Phase 3: Analysis (Planned)
- [ ] Built-in query API for event analysis
- [ ] Aggregation pipelines for behavior patterns
- [ ] Export to data warehouses

### Phase 4: Intelligence (Future)
- [ ] ML-based anomaly detection
- [ ] Automated prompt optimization suggestions
- [ ] Strategy recommendation engine

## Contributing

Contributions are welcome! Please read our contributing guidelines before submitting PRs.

## License

[MIT](LICENSE)

---

**[中文文档](README_CN.md)** | [Technical Documentation](TECHNICAL_DOC.md)
