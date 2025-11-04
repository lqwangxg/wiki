# Keycloak + intra-mart Accel Platform (IAP) 实现 SAML SSO 认证  
## 完整配置手册与异常排查指南  

---

## 🧩 1. 环境概述  

| 项目 | 说明 |
|------|------|
| IdP | Keycloak (SAML 2.0 Provider) |
| SP | intra-mart Accel Platform (IAP) |
| 协议 | SAML 2.0 |
| 目标 | 使用 Keycloak 作为统一认证源，实现 intra-mart 的单点登录 (SSO) |

---

## ⚙️ 2. 前提准备  

### 2.1 证书生成  

使用 OpenSSL 生成自签证书（用于 SP 签名）：

```bash
openssl req -newkey rsa:2048 -nodes -keyout imart_sp.key -x509 -days 3650 -out imart_sp.crt
cat imart_sp.crt imart_sp.key > imart_sp.pem
```
生成文件说明：

| 文件名          | 说明                      |
| ------------ | ----------------------- |
| imart_sp.key | SP 私钥                   |
| imart_sp.crt | SP 公钥证书                 |
| imart_sp.pem | PEM 格式证书（供 Keycloak 导入） |

🏗️ 3. 配置步骤
3.1 Keycloak 端（作为 IdP）

创建新 Realm：intra-mart

打开 Realm Settings → SAML 2.0 Identity Provider Metadata，导出 descriptor.xml

记录 Keycloak 的 IdP 元数据 URL（如 https://idp.example.com/realms/intra-mart/protocol/saml/descriptor）

3.2 IAP 端（作为 SP）

新建 IdP，导入上一步导出的 descriptor.xml

创建 SP 设置（示例）：

| 参数                           | 值                                       |
| ---------------------------- | --------------------------------------- |
| `profInfo_signFlag`          | `true`                                  |
| `profInfo_encryptFlag`       | `false`                                 |
| `Request Binding`            | `HTTP-POST` 或 `HTTP-Redirect`           |
| `sigalg`                     | `SHA256withRSA`                         |
| `Assertion Consumer Service` | `https://iap.example.com/imart/sso/acs` |

上传证书：

私钥：imart_sp.key

公钥：imart_sp.pem

保存后导出 imart-sp-metadata.xml

3.3 Keycloak 端导入 IAP 元数据

在 Keycloak Clients → Create Client → Import → 上传 imart-sp-metadata.xml

核对以下项目：
| 设置项                        | 值                            |
| -------------------------- | ---------------------------- |
| Client ID                  | 与 IAP 中 EntityID 一致          |
| Client Protocol            | saml                         |
| Client Signature Required  | ✅ true                       |
| Valid Redirect URIs        | IAP 的 Assertion Consumer URL |
| IDP Initiated SSO URL Name | 可选（自定义名称）                    |
| Master SAML Processing URL | 留空或与 Redirect URI 一致         |

导入 SP 证书 (imart_sp.pem) 到 Client → “SAML Keys” → “Add Key”

注意：导入后不要切换到 “Settings” 再点 Save，否则 Keycloak 会重新生成新证书覆盖旧值。

🔍 4. 常见错误与排查
❌ invalid_signature / SigAlg was null

原因：请求未签名或签名算法为空。
对策：

确认 IAP 端 profInfo_signFlag=true

IAP 的签名算法必须为 SHA256withRSA

确认 IAP 的 imart_sp.pem 与 Keycloak 中导入的证书一致

❌ It is forbidden to use algorithm xmldsig#sha1

原因：IAP 使用了 SHA1 签名算法。
对策：
在 IAP 中将签名算法修改为 SHA256withRSA。

❌ invalid_signature (Invalid query param signature)

原因：IAP 发送的 Redirect 请求签名与 Keycloak 存储的不一致。
对策：

重新导入 IAP 的 metadata.xml 到 Keycloak

确认未误保存覆盖证书（Keycloak 自动导入后不需再手动保存）


❌ intra-martのユーザコードを取得できませんでした

原因：IAP 无法将 SAML Attribute 映射到本地用户。
对策：

在 Keycloak → Client → Mappers 中添加：
Name: usercd
Mapper Type: User Property
Property: username
Friendly Name: usercd
SAML Attribute Name: usercd

确保 IAP 中的 usercd 与 Keycloak 用户名一致（如 wangxg:wangxg）

❌ Non-secure context detected / CORS not allowed

原因：IAP 在 HTTP 本地访问 Keycloak HTTPS。
对策：

在本地测试时使用 http://localhost 会触发跨域与 Cookie 限制；

可在 Keycloak 启动参数中加入：

-Dkeycloak.hostname.fixed.hostname=localhost
-Dkeycloak.hostname.fixed.http=true


或配置反向代理（nginx）让 IAP 与 Keycloak 通信保持同协议。

🧠 5. 成功验证检查点

IAP 登录跳转到 Keycloak 登录页面；

登录成功后返回 IAP 主页面；

Keycloak 日志中无 invalid_signature；

SAMLResponse 中包含 <Attribute Name="usercd">；

IAP 可正确解析并匹配用户。

✅ 6. 进阶建议

在生产环境使用正式 CA 证书；

Keycloak 与 IAP 建议同属 HTTPS 域；

保留导出的双方 metadata.xml 以便追踪；

每次修改证书后必须重新导入到对方系统。

📘 附录：证书快速检查命令
```bash
# 查看 PEM 内容
openssl x509 -in imart_sp.pem -noout -text

# 验证签名算法
openssl x509 -in imart_sp.pem -text | grep "Signature Algorithm"

# 验证 Key 是否匹配
openssl x509 -noout -modulus -in imart_sp.crt | openssl md5
openssl rsa -noout -modulus -in imart_sp.key | openssl md5
```