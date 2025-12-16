# 统一消息管理平台（Message Hub）架构设计文档

基于 NATS + Message Hub + Docker Compose

------------------------------------------------------------------------

# 1. 背景说明

当前环境包含几十个通过 Docker Compose 部署的 Web 服务，需求包括：

1.  **CI/CD 流程中消息推送到 Mattermost**
2.  **监控预警推送到 Mattermost + Email**
3.  **容器之间通过事件/消息进行任务通知**
4.  希望建立统一的消息中枢，避免服务之间耦合

最终架构选型为：**NATS + Message Hub（自研统一路由服务）**

------------------------------------------------------------------------

# 2. 总体架构

    多个服务（Producer）
            ↓ publish
          NATS（消息总线）
            ↓ subscribe
       Message Hub（统一路由中心）
            ↓
    Mattermost / Email / 其他容器 / HTTP API / Slack / Redis

-   NATS：高速消息通道\
-   Message Hub：消息规则路由、转发、失败重试、统一日志\
-   服务：只负责往 NATS 发消息，不负责推送逻辑

------------------------------------------------------------------------

# 3. NATS 架构说明

### NATS 的优势

-   Topic（Subject）路由灵活\
-   超高性能（可替代 Redis Pub/Sub）\
-   支持 JetStream 消息持久化\
-   易于容器化、易于水平扩展

### NATS Docker Compose 示例

``` yaml
services:
  nats:
    image: nats:2.10
    command: [ "-js", "-D" ]   # 开启 JetStream
    ports:
      - "4222:4222"
      - "8222:8222"
    volumes:
      - ./nats:/data
```

------------------------------------------------------------------------

# 4. Subject 规划（建议）

    ci.cd.*
      ci.cd.deploy.start
      ci.cd.deploy.done

    alert.*
      alert.node.cpu
      alert.node.mem
      alert.app.error
      alert.critical.*

    task.*
      task.file.process
      task.job.run
      task.notify

    system.*
      system.audit
      system.log

使用"分层 Subject"可以让几十个服务清晰解耦。

------------------------------------------------------------------------

# 5. Message Hub 架构说明

Message Hub 是一个轻量服务，可用 **Go / Python / Node.js** 开发。

主要职责：

### A. 统一订阅消息

从多个 subject 订阅消息，例如：\
- `ci.cd.*`\
- `alert.*`\
- `task.*`

### B. 路由规则

根据 subject 自动转发到不同的目标：

  Subject             转发目标
  ------------------- -------------------------------
  ci.cd.\*            Mattermost
  alert.\*            Mattermost + Email
  alert.critical.\*   Mattermost + Email + 可选 SMS
  task.\*             内部 HTTP API / 再推回 NATS

### C. 提供以下功能

-   消息格式验证
-   转发失败重试
-   消息节流/限流
-   统一日志
-   可观测性 (Prometheus)

------------------------------------------------------------------------

# 6. Message Hub 示例（Node.js 实现）

``` javascript
import { connect } from "nats";
import axios from "axios";
import nodemailer from "nodemailer";

const nats = await connect({ servers: "nats:4222" });
const MM_WEBHOOK = process.env.MM_WEBHOOK;

const mailer = nodemailer.createTransport({
  host: "smtp",
  port: 587,
  auth: { user: "alert", pass: "password" },
});

async function handleAlert(msg) {
  const data = JSON.parse(msg.data.toString());

  // Mattermost 推送
  await axios.post(MM_WEBHOOK, {
    text: `🔔 ALERT: ${data.title}
${data.desc}`,
  });

  // Email 推送
  await mailer.sendMail({
    from: "alert@domain.com",
    to: "admin@domain.com",
    subject: `Alert: ${data.title}`,
    text: JSON.stringify(data, null, 2),
  });
}

const sub = nats.subscribe("alert.*");
for await (const msg of sub) {
  handleAlert(msg);
}
```

------------------------------------------------------------------------

# 7. Message Hub Docker Compose（示例）

``` yaml
services:
  message-hub:
    build: ./message-hub
    environment:
      - NATS_URL=nats://nats:4222
      - MM_WEBHOOK=https://mattermost/hooks/xxxx
      - SMTP_HOST=smtp
      - SMTP_USER=alert
      - SMTP_PASS=password
    depends_on:
      - nats
```

------------------------------------------------------------------------

# 8. CI/CD 消息推送示例

CI/CD Pipeline → NATS → Message Hub → Mattermost

### NATS Publish 示例（CI 脚本）

``` bash
nats pub ci.cd.deploy.start   '{"project":"portal","status":"start","branch":"main"}'
```

------------------------------------------------------------------------

# 9. 监控预警（Alertmanager → NATS → Hub）

Alertmanager 配置 webhook 到 Hub 或直接 Publish：

### Alertmanager → NATS 方式示例（Webhook 接收后再 publish）

``` json
{
  "receiver": "critical-alert",
  "status": "firing",
  "alerts": [
    {
      "labels": { "alertname": "CPUHigh", "instance": "node01" },
      "annotations": { "summary": "CPU 90%+" }
    }
  ]
}
```

Message Hub 发布到 subject：

    alert.critical.cpu

------------------------------------------------------------------------

# 10. 容器之间任务通知（Service A → Service B）

示例：

Service A：

``` bash
nats pub task.file.process '{"file": "a.txt"}'
```

Service B：

``` python
nc = await nats.connect("nats:4222")
sub = await nc.subscribe("task.file.process")
```

无需 HTTP 调用，即可实现强解耦。

------------------------------------------------------------------------

# 11. 最佳实践（生产环境建议）

-   **启用 NATS JetStream** 保存告警消息\
-   **Message Hub 多副本（Queue Group）提高 HA**\
-   **所有消息统一使用 JSON 格式**\
-   **用 YAML 写规则（未来支持动态路由）**\
-   **给 Hub 加上 Prometheus 指标**\
-   **邮件/Mattermost 推送带重试**\
-   **开发统一 publish SDK**

------------------------------------------------------------------------

# 12. 总结

使用 **NATS + Message Hub** 能实现：

✔ 跨容器统一消息处理\
✔ CI/CD / 监控 / 任务通知完全解耦\
✔ 高性能、可扩展、易维护\
✔ 非常适合多容器 Docker Compose 环境

------------------------------------------------------------------------
