# BuffGate

[![Go](https://img.shields.io/badge/Go-1.16+-00ADD8?style=flat&logo=go)](https://golang.org)
[![MongoDB](https://img.shields.io/badge/MongoDB-3.6+-47A248?style=flat&logo=mongodb)](https://www.mongodb.com)
[![License](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

> High performance buffering frontend for MongoDB

BuffGate 是一个高性能的 MongoDB 前端缓冲服务，专为客户端事件日志收集场景设计。通过双缓冲机制和对象池技术，显著提升写入吞吐量。

---

## 特性

- 🚀 **高性能** - 双缓冲 + 批量写入，支持高并发场景
- 🔄 **对象复用** - sync.Pool 减少 GC 压力
- ⚡ **自适应策略** - 低负载直写，高负载批量
- 🌐 **CORS 支持** - 开箱即用的跨域配置
- 📦 **轻量级** - 核心代码简洁，易于部署

## 架构

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Clients   │────▶│  BuffGate   │────▶│   MongoDB   │
└─────────────┘     │  (Gin+Giant)│     │  (Bulk API) │
                    └─────────────┘     └─────────────┘
                           │
                    ┌──────┴──────┐
                    │  双缓冲机制  │
                    │ r_buff ⇄ w_buff │
                    └─────────────┘
```

## 快速开始

### 安装

```bash
git clone https://github.com/lightlfyan/buffgate.git
cd buffgate
go mod download
```

### 配置

创建 `config/config.json`:

```json
{
  "port": ":8100",
  "mgo_url": "mongodb://username:password@localhost:27017/collect"
}
```

### 运行

```bash
go run example_server.go
# 或
go build -o buffgate example_server.go
./buffgate
```

### Docker

```bash
docker build -t buffgate .
docker run -p 8100:8100 buffgate
```

## API

### POST /collect

提交事件日志。

**请求示例:**

```bash
curl -X POST http://localhost:8100/collect \
  -H "Content-Type: application/json" \
  -d '{
    "v": "1.0",
    "an": "my-app",
    "ds": "web",
    "av": "2.0.0",
    "t": "track",
    "z": "nonce123",
    "sgn": "signature",
    "cid": "client_001",
    "uid": "user_123",
    "en": "page_view",
    "ev": {"page": "/home"}
  }'
```

**字段说明:**

| 字段 | Key | 必填 | 说明 |
|------|-----|:----:|------|
| Version | v | ✓ | 协议版本 |
| AppName | an | ✓ | 应用名称 |
| DataSource | ds | ✓ | 数据来源 |
| AppVersion | av | ✓ | 应用版本 |
| Type | t | ✓ | 事件类型 |
| Nonce | z | ✓ | 随机数 |
| Sign | sgn | ✓ | 签名 |
| ClientID | cid | ✓ | 客户端ID |
| UserID | uid | ✗ | 用户ID |
| Event | en | ✓ | 事件名称 |
| EventValue | ev | ✗ | 扩展字段 |

**响应:**

| 状态码 | 说明 |
|--------|------|
| 200 | 成功 (`ok`) |
| 400 | 请求格式错误 |

## 性能优化

### 双缓冲机制

```
写入 → w_buff ──swap──▶ r_buff → MongoDB
        (写缓冲)          (读缓冲)
```

读写分离，避免锁竞争。

### 对象池

```go
pool = &sync.Pool{
    New: func() interface{} {
        return new(model.ClientEvent)
    },
}
```

复用对象，降低 GC 开销。

### 批量写入

- 队列 < 10: 直接写入
- 队列 ≥ 10: 批量缓冲 (每批 1024 条)

## 项目结构

```
buffgate/
├── example_server.go    # HTTP 服务入口
├── config/
│   ├── config.go        # 配置加载
│   └── cfg_type.go      # 配置类型
├── model/
│   └── logtype.go       # 数据模型
├── giant/
│   └── giant.go         # 核心缓冲层
└── config.json          # 配置示例
```

## 配置说明

| 参数 | 说明 | 示例 |
|------|------|------|
| port | HTTP 端口 | `:8100` |
| mgo_url | MongoDB 连接串 | `mongodb://user:pass@host:27017/db` |

## 监控

日志输出:
- `queue len: N` - 当前队列深度 (每 10s)
- `flush: M N` - 批量写入 M 条，队列剩余 N 条

## 相关文档

- [技术文档](TECHNICAL_DOC.md) - 完整架构与 API 文档

## License

[MIT](LICENSE)
