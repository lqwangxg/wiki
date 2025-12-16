# Message Hub 需求说明书

**基于 NATS JetStream 的企业级消息中枢设计**

---

## 1. 背景与目标

### 1.1 背景

随着系统服务数量不断增加，系统间通过同步 API 方式进行交互逐渐暴露出以下问题：

- 服务之间强耦合，改动影响范围大
- 高峰期调用链路长，失败率上升
- 重试、补偿逻辑分散在各业务系统中
- 缺乏统一的消息治理与审计能力

为解决上述问题，引入统一的 **Message Hub（消息中枢）**，通过事件驱动方式实现系统解耦与异步处理。

---

### 1.2 建设目标

Message Hub 的建设目标包括：

1. 提供统一的消息消费与分发入口
2. 保证消息处理的可靠性（不丢失、可重试）
3. 降低业务系统之间的耦合度
4. 支持脚本化、自动化的消息接入方式
5. 提供可观测、可运维、可审计的运行能力

---

## 2. 系统定位与边界

### 2.1 Message Hub 的定位

Message Hub 是一个：

- 基于 **NATS JetStream** 的消息消费者服务
- 采用 **Pull + Durable Consumer** 消费模型
- 负责对消息进行校验、路由与转发

其主要职责：

- 从 JetStream 中可靠拉取消息
- 按规则进行分类与分发
- 控制 ACK / 重试 / 失败处理

---

### 2.2 非职责范围（明确不做的事情）

Message Hub 不承担以下职责：

- 不作为消息 Broker（由 NATS 承担）
- 不直接对外提供消息发布接口
- 不负责消息持久化存储
- 不包含具体业务处理逻辑

---

## 3. 总体架构

```
[ Producer (App / Script / Batch) ]
          |
          v
      (Subject)
          |
[ NATS Cluster + JetStream ]
          |
      (Stream)
          |
[ Durable Pull Consumer ]
          |
      Message Hub
          |
          v
[ Downstream Systems / Workers ]
```

---

## 4. 功能性需求

### 4.1 消息接入

#### 4.1.1 消息发布方（Producer）

消息发布由各业务系统或脚本负责，支持以下方式：

- `nats cli`
- Go / Java / Python 等 SDK
- Shell / Batch 脚本

Message Hub 本身 **不负责消息发布**。

---

#### 4.1.2 消息格式规范

**Payload 统一采用 JSON 格式：**

```json
{
  "event_type": "order.created",
  "event_id": "uuid",
  "occurred_at": "2025-12-11T10:30:00Z",
  "source": "order-service",
  "data": {}
}
```

**Header（可选）：**

| Header 名称 | 说明 |
|------------|------|
| X-Trace-ID | 链路追踪 ID |
| X-Retry | 重试次数 |

---

### 4.2 JetStream Stream 设计

| 项目 | 要求 |
|-----|------|
| Stream 类型 | JetStream |
| Subjects | `events.>` |
| Storage | file |
| Replicas | ≥ 3 |
| Retention | workqueue |
| Max Age | 可配置 |

---

### 4.3 消费模型（核心）

#### 4.3.1 消费模式

- Pull Consumer
- Durable Consumer
- Explicit ACK 模式

---

#### 4.3.2 消费处理流程

```
Fetch Message
  -> Schema 校验
    -> 路由判断
      -> 分发处理
        -> ACK / NAK / TERM
```

---

#### 4.3.3 ACK 策略

| 场景 | 处理方式 |
|-----|---------|
| 成功处理 | Ack |
| 临时失败 | Nak |
| 超过最大重试 | Term |
| 长时间处理 | InProgress |

---

### 4.4 路由与分发

#### 4.4.1 路由规则

路由可基于以下信息：

- Subject
- `event_type`
- Header 信息

示例：

```
events.order.*   -> order-worker
events.invoice.* -> invoice-worker
```

---

#### 4.4.2 分发方式

支持的下游分发方式包括：

- HTTP / Webhook
- 内部 RPC 调用
- 再次发布至其他 Subject（可选）

---

## 5. 非功能性需求

### 5.1 可用性

| 项目 | 要求 |
|-----|------|
| Hub 实例数 | ≥ 2 |
| 服务状态 | 无状态 |
| 消费幂等性 | 必须保证 |

---

### 5.2 性能要求

- 支持高并发 Pull 消费
- 支持批量 Fetch
- 可通过横向扩展提升吞吐量

---

### 5.3 可靠性

- 消息不丢失
- 至少一次投递语义
- 支持失败重试与死信处理

---

### 5.4 安全要求

- 使用 NATS Account / User 进行权限控制
- Message Hub 账号仅具备订阅权限
- 禁止 Message Hub 向业务 Subject 发布消息（除非明确授权）

---

## 6. 运维与监控

### 6.1 可观测性指标

- 消费速率（msg/s）
- Pending 消息数量
- ACK / NAK / Term 次数
- 消费延迟

---

### 6.2 日志要求

日志中至少包含以下信息：

- event_id
- subject
- consumer 名称
- 处理结果
- 错误原因（如有）

---

### 6.3 管理与查看能力

- Stream 状态查看
- Consumer 状态查看
- 消费延迟与积压情况

---

## 7. 异常处理与补偿机制

| 场景 | 处理策略 |
|-----|---------|
| 消费失败 | 自动重试 |
| 消费超时 | 自动 Redelivery |
| 无法处理消息 | 转入死信或 Term |
| Hub 宕机 | 消费进度保留，恢复后继续 |

---

## 8. 部署要求

- 容器化部署（Docker / Kubernetes）
- 所有参数配置化（环境变量 / 配置文件）
- 支持滚动升级，不影响消息消费

---

## 9. 未来扩展方向（非必须）

- Schema Registry
- 延迟消息 / 定时消息
- 消息 Replay / 回溯
- 多租户与多业务隔离

---

## 10. 验收标准（Acceptance Criteria）

- 消息在异常情况下不丢失
- Message Hub 重启后可继续消费
- Producer 与 Consumer 完全解耦
- 支持脚本化消息发布
- 消费过程可监控、可审计

---

**文档结束**
