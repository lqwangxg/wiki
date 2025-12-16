# 📜 公司级 HTTP 身份 Header 规范

**Company-wide HTTP Identity Header Specification**

| 属性 | 值 |
| :--- | :--- |
| **Version** | 1.0 |
| **Last Update** | 2025-01 |
| **Scope** | Nginx / oauth2-proxy / Backend Services |
| **Auth Source** | OAuth2 / OpenID Connect (Keycloak) |

---

## 1. 目的（Purpose）

本规范用于统一公司内部 HTTP 请求中**用户身份信息的 Header 传递方式**，确保：

* ✅ 后端系统无需解析 Token
* ✅ 防止敏感 Token 泄露
* ✅ 避免 Header 过大（431）
* ✅ 支持微服务 / WebSocket / API / Legacy 系统
* ✅ 可长期维护、可审计

---

## 2. 身份传递总体原则（Design Principles）



* **OAuth / OIDC 仅在边界层**（`oauth2-proxy`）处理。
* **后端只信任 Nginx 注入的 Header**。
* **禁止**向后端传递 `access_token` / `id_token`。
* 身份 Header 使用固定前缀 `X-User-*`。
* 所有 Header 均视为「**已认证用户**」。

---

## 3. Header 命名规范（Naming Convention）

| 类型 | 规则 |
| :--- | :--- |
| **前缀** | `X-User-` |
| **分隔符** | `-`（kebab-case） |
| **字符集** | ASCII |
| **编码** | UTF-8 |
| **可选性** | 明确标注 `REQUIRED` / `OPTIONAL` |

---

## 4. 标准身份 Header 一览（Standard Headers）

### 4.1 必须 Header（REQUIRED）

| Header 名称 | 来源 | 示例 | 说明 |
| :--- | :--- | :--- | :--- |
| **`X-User-Id`** | `preferred_username` / `sub` | `wangxg` | 公司内唯一用户 ID |
| **`X-User-Email`** | `email` | `wangxg@company.com` | 主邮箱 |
| **`X-User-Auth-Provider`** | 固定值 | `keycloak` | 认证来源 |

### 4.2 推荐 Header（RECOMMENDED）

| Header 名称 | 示例 | 说明 |
| :--- | :--- | :--- |
| **`X-User-Name`** | `王小刚` | 展示用姓名 |
| **`X-User-Groups`** | `rpa-admin,finance` | 逗号分隔的权限组 |
| **`X-User-Tenant`** | `jp` | 子公司 / 租户 |

### 4.3 可选 Header（OPTIONAL）

| Header 名称 | 示例 | 说明 |
| :--- | :--- | :--- |
| **`X-User-Dept`** | `IT部门` | 部门信息 |
| **`X-User-Role`** | `admin` | 主角色 |
| **`X-User-Locale`** | `ja-JP` | 语言/地区设置 |

---

## 5. 明确禁止的 Header（FORBIDDEN）

> 🚫 **任何后端系统禁止依赖以下 Header**

| Header | 原因 |
| :--- | :--- |
| `Authorization` | Token 泄露风险 |
| `X-Access-Token` | Header 过大 / 安全风险 |
| `X-ID-Token` | 含敏感 Claims |
| `Cookie` | 后端不应解析 Session |

---

## 6. Nginx / oauth2-proxy 标准注入示例

```nginx
# OAuth2 Proxy auth_request
auth_request /oauth2/auth;
error_page 401 = /oauth2/start?rd=$scheme://$host$request_uri;

# 从 oauth2-proxy 读取身份
auth_request_set $user_id   $upstream_http_x_auth_request_preferred_username;
auth_request_set $user_mail $upstream_http_x_auth_request_email;
auth_request_set $groups    $upstream_http_x_auth_request_groups;

# 注入到后端
proxy_set_header X-User-Id            $user_id;
proxy_set_header X-User-Email         $user_mail;
proxy_set_header X-User-Groups        $groups;
proxy_set_header X-User-Auth-Provider keycloak;
```

---

## 7. Backend 身份信任模型（Trust Model）

后端系统**必须**遵循：

1.  **仅信任**来自 Nginx 的请求
2.  Header 存在即视为**已认证**
3.  **不自行校验 Token**
4.  **不**从 `Cookie` / `Query` 取身份

**Java / Spring 示例**

```java
String userId = request.getHeader("X-User-Id");
if (userId == null) {
    throw new UnauthorizedException();
}
// 此时 userId 已经过 Nginx 认证，可信
```

---

## 8. WebSocket / API 兼容性

* **WebSocket**：Header 在握手阶段传递 ✅
* **REST API**：标准支持 ✅
* **Legacy CGI**：无需 OAuth 支持，仅读取 Header ✅

---

## 9. 日志与审计建议（Logging）

| 推荐记录 | 禁止记录 |
| :--- | :--- |
| `X-User-Id` | `Cookie` |
| `X-User-Email` | `Token` |
| `X-User-Groups` | `Authorization Header` |

---

## 10. 版本演进策略（Versioning）

* **Header 名称**：禁止变更。
* **新字段**：只能**新增**，不删除。
* **废弃字段**：至少保留 12 个月。

---

## 11. 责任划分（Responsibility）

| 组件 | 责任 |
| :--- | :--- |
| `oauth2-proxy` | 认证 / Token 校验 |
| **Nginx** | Header 注入 / 防伪 |
| **Backend** | Header 使用 / 授权 |

---

## 12. 附录：最小可用 Header 集合（MVP）

* `X-User-Id`
* `X-User-Email`
* `X-User-Auth-Provider`