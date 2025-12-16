# 公司级 Linux 边界防护规范

> **版本**：v1.0  
> **适用范围**：公网/混合云 Linux 主机（Nginx / API / Web）  
> **目标**：在最早网络阶段阻断恶意流量，降低应用与 TLS 层负载，消灭日志噪音，形成可审计、可运维、可扩展的边界防护体系。

---

## 1. 设计原则

### 1.1 防护分层（从外到内）

| 层级 | 技术 | 目标 |
|----|----|----|
| L3/L4 最早期 | **nftables PREROUTING(raw)** | 直接丢弃恶意 IP，最高性能 |
| L7 前置 | **Fail2Ban** | 动态封禁扫描/攻击源 |
| 应用层 | **Nginx + Lua** | 精准识别异常行为 |
| 业务层 | Backend | 只接收“干净流量” |

### 1.2 核心原则

- **Drop 越早越好**（PREROUTING > INPUT）
- **识别在应用层，拦截在网络层**
- **Fail2Ban 只负责“决策 + ban”，不做重逻辑**
- **日志必须“可控、可分类、可关闭”**

---

## 2. 总体架构

```text
[ Internet ]
      │
      ▼
[nftables PREROUTING(raw)]  ← Fail2Ban 动态 set
      │ (drop)
      ▼
[ TCP / TLS 栈 ]
      │
      ▼
[ Nginx + Lua 检测 ]
      │ (log tag)
      ▼
[ Fail2Ban jail ]
```

---

## 3. nftables（最终拦截层）

### 3.1 表与链规范

- 使用 **inet family**（同时支持 IPv4/IPv6）
- 使用 **PREROUTING + raw priority**

```nft
table inet f2b-preraw {
    chain prerouting {
        type filter hook prerouting priority raw; policy accept;
    }
}
```

### 3.2 每个 jail 一个独立 set

```nft
set f2b-nginx-http-auth { type ipv4_addr; flags timeout; }
set f2b-nginx-badbot   { type ipv4_addr; flags timeout; }
set f2b-nginx-lua-scan { type ipv4_addr; flags timeout; }
```

### 3.3 Drop 规则

```nft
chain prerouting {
    type filter hook prerouting priority raw; policy accept;

    ip saddr @f2b-nginx-http-auth drop
    ip saddr @f2b-nginx-badbot   drop
    ip saddr @f2b-nginx-lua-scan drop
}
```

> 说明：
> - 使用 **set timeout**，由 nft 内核自动过期
> - 不依赖 fail2ban 定时扫描，性能最优

---

## 4. Fail2Ban 设计规范

### 4.1 运行模式

- **容器运行（host network）**
- 不依赖 iptables，仅使用 nftables

### 4.2 action 规范（示例）

`action.d/nft-preraw.conf`

```ini
[Definition]

actionstart = nft add table inet f2b-preraw
              nft 'add chain inet f2b-preraw prerouting { type filter hook prerouting priority raw; policy accept; }'

actionban   = nft add element inet f2b-preraw <setname> { <ip> timeout <bantime> }

actionunban = nft delete element inet f2b-preraw <setname> { <ip> }
```

### 4.3 jail 规范

```ini
[nginx-lua-scan]
enabled   = true
filter    = nginx-lua-scan
logpath   = /var/log/nginx/lua_scan.log
bantime   = 30d
findtime  = 5m
maxretry  = 1
banaction = nft-preraw
```

---

## 5. Nginx + Lua（检测层）

### 5.1 原则

- **Lua 只做判断，不做封禁**
- 封禁统一交给 Fail2Ban

### 5.2 Lua 示例（rewrite_by_lua_block）

```lua
-- 非法 Host / TLS 探测 / 协议异常
if ngx.var.ssl_server_name == "" then
    ngx.log(ngx.ERR, "F2B_LUA_SCAN ip=", ngx.var.remote_addr, " reason=no_sni")
    return ngx.exit(444)
end
```

### 5.3 独立日志

```nginx
error_log /var/log/nginx/lua_scan.log error;
```

---

## 6. Fail2Ban filter 规范

`filter.d/nginx-lua-scan.conf`

```ini
[Definition]
failregex = F2B_LUA_SCAN ip=<HOST>
ignoreregex =
```

---

## 7. TLS-only 扫描防护（低误伤）

### 7.1 Nginx

```nginx
error_page 497 = @bad_tls;

location @bad_tls {
    access_log /var/log/nginx/tls_scan.log;
    return 444;
}
```

### 7.2 对应 Fail2Ban jail

```ini
[nginx-tls-scan]
filter    = nginx-tls-scan
logpath  = /var/log/nginx/tls_scan.log
bantime  = 7d
maxretry = 1
banaction = nft-preraw
```

---

## 8. 日志治理（必须遵守）

- SSL 握手噪音 → 独立日志或直接 drop
- 禁止在主 error.log 输出扫描噪音
- Lua / TLS / Auth 各自独立 log

---

## 9. 运维检查清单

### 9.1 nftables

```bash
nft list table inet f2b-preraw
nft list chain inet f2b-preraw prerouting
```

### 9.2 Fail2Ban

```bash
fail2ban-client status
fail2ban-client status nginx-lua-scan
```

---

## 10. 安全基线结论

✔ 攻击 IP 在 **进入 TCP 栈前被 drop**  
✔ Nginx / TLS 几乎无扫描噪音  
✔ Fail2Ban 规则可审计、可维护  
✔ 架构支持长期运行与规模扩展

---

> 本规范为公司级 Linux 边界防护基线，任何新上线公网服务 **必须遵守本规范**。

