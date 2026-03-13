# BuffGate 技术文档

> MongoDB 高性能前端缓冲层

---

## 目录

- [项目概述](#项目概述)
- [技术架构](#技术架构)
- [核心组件](#核心组件)
- [数据流](#数据流)
- [安装部署](#安装部署)
- [配置说明](#配置说明)
- [API 接口](#api-接口)
- [性能优化策略](#性能优化策略)

---

## 项目概述

BuffGate 是一个高性能的 MongoDB 前端缓冲服务，主要用于：

- **客户端事件收集** - 接收来自各客户端的事件日志
- **批量写入优化** - 通过双缓冲机制批量写入 MongoDB
- **高并发处理** - 使用内存池和异步处理提升吞吐量

### 技术栈

| 组件 | 技术 |
|------|------|
| 语言 | Go |
| Web 框架 | Gin |
| 数据库 | MongoDB (mgo driver) |
| 日志 | glog |
| 配置 | JSON |

---

## 技术架构

```
┌─────────────────────────────────────────────────────────────────┐
│                        Client Requests                           │
│                     (POST /collect)                              │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                     Gin HTTP Server                              │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────────┐  │
│  │   Recovery  │ -> │    CORS     │ -> │   /collect Handler  │  │
│  └─────────────┘    └─────────────┘    └─────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Giant Buffer Layer                          │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    sync.Pool                              │   │
│  │            (ClientEvent 对象复用池)                        │   │
│  └──────────────────────────────────────────────────────────┘   │
│                          │                                       │
│                          ▼                                       │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                   stomach Channel                         │   │
│  │              (容量: 10240 的缓冲通道)                       │   │
│  └──────────────────────────────────────────────────────────┘   │
│                          │                                       │
│              ┌───────────┴───────────┐                          │
│              ▼                       ▼                          │
│  ┌─────────────────┐     ┌─────────────────┐                   │
│  │  直接写入模式    │     │  批量写入模式    │                   │
│  │  (队列 < 10)    │     │  (队列 >= 10)   │                   │
│  └─────────────────┘     └─────────────────┘                   │
│                                  │                               │
│                                  ▼                               │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                  Double Buffer                            │   │
│  │        ┌─────────┐           ┌─────────┐                 │   │
│  │        │ r_buff  │ <─swap──> │ w_buff  │                 │   │
│  │        │ (读取)   │           │ (写入)  │                 │   │
│  │        └─────────┘           └─────────┘                 │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                       MongoDB                                    │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Database: gsensor                                        │   │
│  │  Collection: clientlog                                    │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 核心组件

### 1. Config 模块 (`config/`)

配置管理模块，负责加载和存储应用配置。

**配置结构 (`cfg_type.go`):**

```go
type CfgType struct {
    Port   string `json:"port"`     // HTTP 服务端口
    MgoUrl string `json:"mgo_url"`  // MongoDB 连接字符串
}
```

**配置加载 (`config.go`):**

- 自动从可执行文件目录加载 `config/config.json`
- 使用 JSON 反序列化配置

### 2. Model 模块 (`model/`)

数据模型定义。

**ClientEvent 结构 (`logtype.go`):**

| 字段 | JSON Key | 类型 | 必填 | 说明 |
|------|----------|------|------|------|
| Version | v | string | ✓ | 协议版本 |
| AppName | an | string | ✓ | 应用名称 |
| DataSource | ds | string | ✓ | 数据来源 |
| AppVersion | av | string | ✓ | 应用版本 |
| Type | t | string | ✓ | 事件类型 |
| Nonce | z | string | ✓ | 随机数 |
| Sign | sgn | string | ✓ | 签名 |
| ClientID | cid | string | ✓ | 客户端 ID |
| UserID | uid | string | ✗ | 用户 ID |
| Event | en | string | ✓ | 事件名称 |
| EventValue | ev | map | ✗ | 事件附加值 |
| Timestamp | - | time.Time | 自动 | 服务器时间戳 |

### 3. Giant 模块 (`giant/`)

核心缓冲层，实现双缓冲和批量写入。

**Giant 结构:**

```go
type Giant struct {
    stomach  chan *ClientEvent     // 输入通道 (容量 10240)
    close    chan int              // 关闭信号
    session  *mgo.Session          // MongoDB 会话

    r_buff   []*ClientEvent        // 读缓冲区 (初始容量 128)
    w_buff   []*ClientEvent        // 写缓冲区 (初始容量 128)

    rw       *sync.RWMutex         // 读写锁
    pool     *sync.Pool            // 对象池
}
```

**核心方法:**

| 方法 | 功能 |
|------|------|
| `Live()` | 主循环，接收事件并分发 |
| `flush()` | 后台刷新，批量写入 MongoDB |
| `swap()` | 交换读写缓冲区 |
| `Start()` | 启动 Giant 服务 |
| `GetEvent()` | 从对象池获取 ClientEvent |
| `Send()` | 发送事件到缓冲通道 |

### 4. HTTP Server (`example_server.go`)

Gin 框架实现的 HTTP 服务。

**中间件:**

- `gin.Recovery()` - 异常恢复
- `Cors()` - CORS 跨域支持

**路由:**

- `POST /collect` - 接收客户端事件

---

## 数据流

### 请求处理流程

```
1. 客户端 POST /collect
        │
        ▼
2. handler() 解析 JSON
        │
        ▼
3. 从 sync.Pool 获取 ClientEvent
        │
        ▼
4. 绑定 JSON 到 ClientEvent
        │
        ▼
5. 设置服务器时间戳 (UTC+8)
        │
        ▼
6. 发送到 stomach 通道
        │
        ▼
7. 返回 "ok" 响应
```

### 写入策略

**直接写入模式 (队列长度 < 10):**

```go
// 立即写入 MongoDB，适合低负载
c.Insert(event)
pool.Put(event)  // 归还对象池
```

**批量写入模式 (队列长度 >= 10):**

```go
// 写入缓冲区，由 flush() 后台批量写入
w_buff = append(w_buff, event)
```

### 批量刷新流程

```
flush() goroutine (持续运行)
        │
        ▼
检查 r_buff 是否有数据
        │
        ├── 空 → 检查 w_buff → 有数据 → swap()
        │                    └── 无数据 → sleep 100ms
        │
        ▼
批量插入 (每批 1024 条)
        │
        ▼
bulk.Insert(r_buff[i:idx_end])
bulk.Run()
        │
        ▼
对象归还 pool
清空 r_buff
```

---

## 安装部署

### 环境要求

- Go 1.16+
- MongoDB 3.6+

### 构建步骤

```bash
# 克隆项目
git clone https://github.com/lightlfyan/buffgate.git
cd buffgate

# 下载依赖
go mod download

# 构建
go build -o buffgate example_server.go

# 创建配置目录
mkdir -p config
```

### 配置文件

创建 `config/config.json`:

```json
{
  "port": ":8100",
  "mgo_url": "mongodb://username:password@localhost:27017/collect"
}
```

### 启动服务

```bash
# 直接运行
./buffgate

# 带日志参数
./buffgate -log_dir=./logs -v=2
```

### Docker 部署

```dockerfile
FROM golang:1.21-alpine AS builder
WORKDIR /app
COPY . .
RUN go build -o buffgate example_server.go

FROM alpine:latest
WORKDIR /app
COPY --from=builder /app/buffgate .
COPY config config
EXPOSE 8100
CMD ["./buffgate"]
```

---

## 配置说明

### config.json

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| port | string | ":8100" | HTTP 监听端口 |
| mgo_url | string | - | MongoDB 连接字符串 |

### MongoDB 连接字符串格式

```
mongodb://[username:password@]host1[:port1][,host2[:port2],...]/database[?options]
```

**示例:**

```json
// 无认证
"mgo_url": "mongodb://localhost:27017/collect"

// 带认证
"mgo_url": "mongodb://admin:password@localhost:27017/collect"

// 副本集
"mgo_url": "mongodb://host1:27017,host2:27017,host3:27017/collect?replicaSet=myReplicaSet"
```

---

## API 接口

### POST /collect

提交客户端事件日志。

**请求头:**

```
Content-Type: application/json
```

**请求体:**

```json
{
  "v": "1.0",
  "an": "my-app",
  "ds": "web",
  "av": "2.1.0",
  "t": "track",
  "z": "abc123",
  "sgn": "signature_hash",
  "cid": "client_001",
  "uid": "user_123",
  "en": "page_view",
  "ev": {
    "page": "/home",
    "referrer": "https://example.com"
  }
}
```

**响应:**

| 状态码 | 说明 | 响应体 |
|--------|------|--------|
| 200 | 成功 | `ok` |
| 400 | 请求格式错误 | `payload error\n` |

**cURL 示例:**

```bash
curl -X POST http://localhost:8100/collect \
  -H "Content-Type: application/json" \
  -d '{
    "v": "1.0",
    "an": "test-app",
    "ds": "mobile",
    "av": "1.0.0",
    "t": "event",
    "z": "random123",
    "sgn": "sign456",
    "cid": "client789",
    "en": "click",
    "ev": {"button": "submit"}
  }'
```

---

## 性能优化策略

### 1. 对象池 (sync.Pool)

复用 `ClientEvent` 对象，减少 GC 压力：

```go
pool = &sync.Pool{
    New: func() interface{} {
        return new(model.ClientEvent)
    },
}
```

### 2. 双缓冲机制

读写分离，避免锁竞争：

```
写入线程 → w_buff (写缓冲)
              ↓ swap()
flush线程 ← r_buff (读缓冲)
```

### 3. 批量写入

使用 MongoDB Bulk API，每批 1024 条：

```go
bulk := collection.Bulk()
for _, event := range batch {
    bulk.Insert(event)
}
bulk.Run()
```

### 4. 自适应写入策略

- 低负载 (< 10 条队列): 直接写入
- 高负载 (>= 10 条队列): 批量缓冲

### 5. 异步处理

- `Live()` 主循环处理请求
- `flush()` goroutine 后台批量写入

---

## 监控指标

### 日志输出

| 日志 | 含义 |
|------|------|
| `queue len: N` | 当前队列长度 (每 10 秒) |
| `flush: M N` | 刷新 M 条，队列剩余 N 条 |
| `Error: {...}` | 写入失败的事件 |

### 关键指标

- 队列深度 (`len(stomach)`)
- 缓冲区大小 (`len(r_buff)`, `len(w_buff)`)
- 写入延迟
- MongoDB 连接状态

---

## 目录结构

```
buffgate/
├── example_server.go    # HTTP 服务入口
├── config/
│   ├── config.go        # 配置加载逻辑
│   └── cfg_type.go      # 配置类型定义
├── model/
│   └── logtype.go       # ClientEvent 数据模型
├── giant/
│   └── giant.go         # 核心缓冲层
├── config.json          # 配置文件 (示例)
└── README.md            # 项目说明
```

---

## 扩展建议

1. **健康检查接口** - 添加 `/health` 或 `/ready` 端点
2. **指标暴露** - 集成 Prometheus metrics
3. **优雅关闭** - 实现 SIGTERM 信号处理，刷新缓冲区后退出
4. **重试机制** - MongoDB 写入失败时的重试策略
5. **配置热更新** - 支持运行时更新配置

---

## License

MIT License
