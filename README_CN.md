# BuffGate

[![Go](https://img.shields.io/badge/Go-1.16+-00ADD8?style=flat&logo=go)](https://golang.org)
[![MongoDB](https://img.shields.io/badge/MongoDB-3.6+-47A248?style=flat&logo=mongodb)](https://www.mongodb.com)
[![License](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

> **AI Agent 时代的可观测性基础设施**
>
> 采集、存储、分析 —— Agent 行为优化的数据基石

BuffGate 是专为 AI Agent 时代设计的高性能事件采集服务。作为 AI 工程化体系中的可观测性基础设施，它负责采集 Agent 运行时事件，为后续的 Agent 行为分析提供数据源，从而实现提示词和运行策略的持续优化。

---

## 为什么需要 BuffGate？

在 AI Agent 时代，理解 **Agent 做了什么** 与 **Agent 产出了什么** 同样重要。BuffGate 提供：

| 挑战 | 解决方案 |
|------|----------|
| Agent 行为难以追踪 | 采集每一次行动、决策和结果 |
| 提示词需要迭代优化 | 基于真实使用数据驱动优化 |
| 运行策略需要演进 | 分析跨数千次运行的行为模式 |
| 基础设施需要弹性扩展 | 轻松处理高吞吐事件流 |

## 愿景

```
┌─────────────────────────────────────────────────────────────────────┐
│                        AI 工程化 Harness                             │
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
│                    │      事件采集器      │                          │
│                    └──────────┬──────────┘                          │
│                               │                                      │
└───────────────────────────────┼──────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                           数据层                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   ┌──────────────┐                         ┌──────────────────┐     │
│   │   MongoDB    │  ◀── 事件流 ───▶        │    分析引擎      │     │
│   │   (存储)     │                         │   (规划中)       │     │
│   └──────────────┘                         └──────────────────┘     │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         优化闭环                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐           │
│    │   分析      │───▶│   优化      │───▶│   部署      │           │
│    │  行为数据   │    │   提示词    │    │  新策略     │           │
│    └─────────────┘    └─────────────┘    └─────────────┘           │
│           ▲                                        │                 │
│           └────────────────────────────────────────┘                 │
│                          持续改进                                    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

## 核心能力

### 🎯 事件采集

捕获 Agent 运行时的全量事件：

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

### 📊 行为分析就绪

专为下游分析设计的事件结构：

| 事件类型 | 用途 |
|----------|------|
| `tool_call` | 分析工具使用模式，识别高频操作 |
| `decision` | 追踪推理路径，评估决策质量 |
| `error` | 识别失败模式，优先修复问题 |
| `milestone` | 衡量任务完成度，统计成功率 |
| `feedback` | 关联用户反馈与 Agent 行为 |

### ⚡ 高性能

- **双缓冲机制** - 读写分离，削峰填谷
- **对象池复用** - 降低 GC 压力
- **自适应批量** - 低负载直写，高负载批量
- **Bulk 写入** - 最大化 MongoDB 吞吐量

## 系统架构

```
                    Agent 运行时
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
              │   (CORS 已启用)     │
              └──────────┬──────────┘
                         │
                         ▼
              ┌─────────────────────┐
              │   Giant 缓冲层      │
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

## 快速开始

### 安装

```bash
git clone https://github.com/lightlfyan/buffgate.git
cd buffgate
go mod download
```

### 配置

创建 `config/config.json`：

```json
{
  "port": ":8100",
  "mgo_url": "mongodb://localhost:27017/agent_events"
}
```

### 运行

```bash
go run example_server.go
```

### Docker 部署

```bash
docker build -t buffgate .
docker run -p 8100:8100 buffgate
```

## API 参考

### POST /collect

提交 Agent 事件。

**请求示例：**

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

**字段说明：**

| 字段 | JSON Key | 必填 | 描述 |
|------|----------|:----:|------|
| 版本 | `v` | ✓ | 协议版本 |
| Agent 名称 | `an` | ✓ | Agent 标识符 |
| 数据来源 | `ds` | ✓ | 来源 (cli, api, plugin 等) |
| Agent 版本 | `av` | ✓ | Agent 版本号 |
| 事件类型 | `t` | ✓ | 事件类型 |
| 随机数 | `z` | ✓ | 请求随机数 |
| 签名 | `sgn` | ✓ | 请求签名 |
| 会话 ID | `cid` | ✓ | 会话/对话标识符 |
| 用户 ID | `uid` | | 可选用户标识 |
| 事件名 | `en` | ✓ | 事件名称 |
| 事件值 | `ev` | | 任意键值对载荷 |

**响应：**

| 状态码 | 响应体 |
|--------|--------|
| 200 | `ok` |
| 400 | `payload error` |

## 集成示例

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

# 使用示例
client = BuffGateClient("http://localhost:8100", "code-agent", "secret")

# 发送工具调用事件
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

## 发展路线

### 第一阶段：基础设施 ✅
- [x] 高性能事件采集
- [x] MongoDB 持久化
- [x] 双缓冲架构

### 第二阶段：可观测性（规划中）
- [ ] Prometheus 指标导出
- [ ] 健康检查端点
- [ ] 实时监控面板

### 第三阶段：分析能力（规划中）
- [ ] 内置事件查询 API
- [ ] 行为模式聚合管道
- [ ] 数据仓库导出

### 第四阶段：智能化（未来）
- [ ] 基于机器学习的异常检测
- [ ] 自动化提示词优化建议
- [ ] 策略推荐引擎

## 贡献

欢迎贡献代码！提交 PR 前请阅读贡献指南。

## 许可证

[MIT](LICENSE)

---

**[English](README.md)** | [技术文档](TECHNICAL_DOC.md)
