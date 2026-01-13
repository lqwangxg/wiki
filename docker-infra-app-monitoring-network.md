# Docker Infra + App + Monitoring 网络规划与 Compose 模板

> 适用场景：多套 docker-compose、反向代理（nginx）、认证（oauth2-proxy）、业务服务、数据库、监控（Prometheus + Grafana）  
> 目标：**网络可控、可扩展、可维护、可观测**

---

## 一、总体设计目标

- **网络先行**：所有网络手动创建，`docker-compose down` 不影响网络
- **职责隔离**：infra / app / monitoring 网络分离
- **安全可控**：最小互通原则（nginx 作为入口）
- **监控统一**：Prometheus 可采集全部容器

---

## 二、网络规划总览

```
                        Internet
                            |
                         [ 80/443 ]
                            |
                        ┌─────────┐
                        │  nginx  │
                        └────┬────┘
                             |
        ┌────────────────────┼────────────────────┐
        |                    |                    |
   infra_net             app_net_xxx          monitoring_net
  172.30.0.0/16        172.31.10.0/24        172.32.0.0/16
        |                    |                    |
 oauth2-proxy        app / worker        prometheus / grafana
 postgres / redis
```

---

## 三、Docker Network 详细规划

### 1. infra_net（基础设施）

| 项目 | 值 |
|----|----|
| Network | infra_net |
| Driver | bridge |
| Subnet | 172.30.0.0/16 |
| Gateway | 172.30.0.1 |
| 用途 | nginx / oauth2-proxy / DB / 共享组件 |

```bash
docker network create \
  --driver bridge \
  --subnet 172.30.0.0/16 \
  --gateway 172.30.0.1 \
  infra_net
```

---

### 2. app_net_example（业务系统示例）

| 项目 | 值 |
|----|----|
| Network | app_net_example |
| Subnet | 172.31.10.0/24 |
| Gateway | 172.31.10.1 |
| 用途 | 单一业务系统 |

```bash
docker network create \
  --driver bridge \
  --subnet 172.31.10.0/24 \
  --gateway 172.31.10.1 \
  app_net_example
```

---

### 3. monitoring_net（监控）

| 项目 | 值 |
|----|----|
| Network | monitoring_net |
| Subnet | 172.32.0.0/16 |
| Gateway | 172.32.0.1 |
| 用途 | Prometheus / Grafana |

```bash
docker network create \
  --driver bridge \
  --subnet 172.32.0.0/16 \
  --gateway 172.32.0.1 \
  monitoring_net
```

---

## 四、Infra Compose 模板（nginx + oauth2-proxy + DB）

```yaml
version: "3.9"

services:
  nginx:
    image: nginx:alpine
    container_name: infra-nginx
    ports:
      - "80:80"
      - "443:443"
    networks:
      - infra_net
      - monitoring_net
    depends_on:
      - oauth2-proxy

  oauth2-proxy:
    image: quay.io/oauth2-proxy/oauth2-proxy
    container_name: infra-oauth2
    networks:
      - infra_net

  postgres:
    image: postgres:15
    container_name: infra-postgres
    environment:
      POSTGRES_PASSWORD: example
    networks:
      - infra_net

networks:
  infra_net:
    external: true
  monitoring_net:
    external: true
```

---

## 五、App Compose 模板（示例业务）

```yaml
version: "3.9"

services:
  app:
    image: myorg/myapp:latest
    container_name: app-example
    expose:
      - "8080"
    networks:
      - app_net
      - infra_net
      - monitoring_net

networks:
  app_net:
    external: true
    name: app_net_example
  infra_net:
    external: true
  monitoring_net:
    external: true
```

---

## 六、Monitoring Compose 模板（Prometheus + Grafana）

```yaml
version: "3.9"

services:
  prometheus:
    image: prom/prometheus
    container_name: monitor-prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"
    networks:
      - monitoring_net

  grafana:
    image: grafana/grafana
    container_name: monitor-grafana
    ports:
      - "3000:3000"
    networks:
      - monitoring_net
      - infra_net

networks:
  monitoring_net:
    external: true
  infra_net:
    external: true
```

---

## 七、Prometheus 基础配置示例

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "docker"
    static_configs:
      - targets:
        - "infra-nginx:80"
        - "app-example:8080"
```

---

## 八、最佳实践 & 注意事项

- ❌ 不使用 compose 自动创建网络
- ✅ nginx 连接 infra + app
- ✅ 监控组件只读访问业务
- ❌ 业务服务不暴露端口（仅 expose）
- ✅ 网络命名即文档

---

## 九、扩展建议

- 每新增一个系统：新建一个 `app_net_xxx`
- 结合 iptables / docker-proxy 做东西向隔离
- 后续可平滑迁移到 Kubernetes / CNI

---

**文件用途**：  
- 可直接提交 Git  
- 作为团队 Docker 网络规范文档  
- 新项目复制即用
