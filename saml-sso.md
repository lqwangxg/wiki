---
title: saml-sso 
description: keycloak 作为saml IdP，为intra-mart accel platfrom 提供sso认证服务
published: true
date: 2025-10-29T05:17:44.223Z
tags: saml, sso, keycloak, iap
editor: markdown
dateCreated: 2025-10-29T05:14:33.027Z
---

# Keycloak 作为 IdP 与 intra-mart Accel Platform 作为 SP 的 SAML 签名配置与排查手册

> 目标：在 Keycloak（IdP）与 intra-mart Accel Platform（SP）间建立稳定的 SAML 2.0 SSO，确保 AuthnRequest 与 Assertion 的签名/验签正确，避免 `invalid_signature` / `SigAlg was null` 等错误。

---

## 一、文档适用对象

* 负责 SSO 配置的工程师/运维。
* 已部署 Keycloak 和 intra-mart，能够访问两者管理界面。

---

## 二、前提假设（示例域名）

* Keycloak: `https://keycloak.example.com`（Realm: `intra-mart`）
* intra-mart: `https://imart.example.com`
* 使用 HTTPS（强烈建议）

---

## 三、总体流程概览

1. 在 intra-mart 生成或导出 SP Metadata（包含 SP EntityID、ACS、SP 公钥）。
2. 在 Keycloak 新建 SAML Client 并导入 SP Metadata / 上传 SP 证书。
3. 在 Keycloak 配置签名/加密策略（Sign Assertions / Sign Documents / Signature Algorithm）。
4. 在 intra-mart 导入 Keycloak 的 IdP Metadata（Keycloak 的 `.../protocol/saml/descriptor`）。
5. 在 intra-mart 启用 AuthnRequest 签名与 POST Binding（推荐），并选择签名算法（RSA-SHA256）。
6. 测试登录并根据日志逐项排查。

---

## 四、证书与密钥（生成示例）

下面示例演示如何为 SP（intra-mart）生成自签名证书以用于签名 AuthnRequest。

```bash
# 生成 SP 私钥
openssl genpkey -algorithm RSA -out imart_sp.key -pkeyopt rsa_keygen_bits:2048

# 生成 CSR
openssl req -new -key imart_sp.key -out imart_sp.csr \
  -subj "/C=JP/ST=Tokyo/L=Tokyo/O=Example/OU=IT/CN=imart.example.com"

# 生成自签名证书（1 年有效）
openssl x509 -req -days 365 -in imart_sp.csr -signkey imart_sp.key -out imart_sp.crt

# 将证书转为 PEM（如果需要）
openssl x509 -in imart_sp.crt -out imart_sp.pem -outform PEM
```

> 在生产环境建议使用公司内部 CA 或受信任 CA 签发证书，不使用自签名。

---

## 五、intra-mart（SP）端设置步骤（推荐配置）

> 以下为常见 intra-mart 管理 UI 的字段名（日文界面说明）和推荐值。你的 intra-mart 版本或定制化 UI 可能会略有差异。

1. 登录 intra-mart 管理控制台（系统管理员）。
2. 前往：**システム管理 → 認証 → SAML 連携設定**（或相似路径）。
3. 新建或编辑 SP 配置：

   * **SP Entity ID**：`https://imart.example.com/imart/sp`
   * **AssertionConsumerService (ACS) URL**：`https://imart.example.com/imart/sso/saml/SSO/alias/intra-mart-sp`（以实际 URL 为准）
   * **NameID Format**：`urn:oasis:names:tc:SAML:1.1:nameid-format:unspecified`（或 `emailAddress`）
   * **SSO Binding**：**HTTP-POST**（推荐）
   * **Signature of AuthnRequest (AuthnRequest署名)**：**ON（有効）**
   * **Signature Algorithm (署名アルゴリズム)**：`RSA-SHA256`（与 Keycloak 保持一致）
   * **SP 証明書（公開鍵）**：上传 `imart_sp.crt` 或 `imart_sp.pem` 内容
   * **WantAuthnRequestsSigned**：设置为 `true`（如果存在此项）
4. 导出或下载 SP metadata（通常有按钮生成 `metadata.xml`），保存为 `imart-sp-metadata.xml`。

> 注意：若 intra-mart 的默认为 Redirect Binding（GET），请改为 POST。Redirect 绑定下若不包含 SigAlg/Signature 会触发 `SigAlg was null` 错误。

---

## 六、Keycloak（IdP）端设置步骤

1. 登录 Keycloak 管理控制台：`https://keycloak.example.com/auth/admin/`（路径根据版本略有差异）。
2. 选择或创建 Realm（示例：`intra-mart`）。
3. 新建 Client：

   * **Client ID**：建议使用 SP Entity ID，如 `https://imart.example.com/imart/sp`。
   * **Client Protocol**：选择 `saml`。
   * **Save**。
4. 编辑 Client → `Settings`：

   * **Enabled**：ON
   * **Client Signature Required**：ON（要求验证 SP 发来的 AuthnRequest 签名）
   * **Force POST Binding**：ON（强制使用 HTTP-POST）
   * **Sign Documents**：ON
   * **Sign Assertions**：ON
   * **SAML Signature Algorithm**：`RSA_SHA256`（或 `RSA_SHA512`，但与 SP 保持一致）
   * **Root URL / Base URL**：`https://imart.example.com/`（用于生成链接）
   * **Valid Redirect URIs**：`https://imart.example.com/*` 或更精确的 ACS 路径
5. 切换到 `Keys` 或 `Certificates` 选项卡（Keycloak 版本 UI 名称不同）:

   * 上传 SP 的公钥/证书（`imart_sp.crt`），以便 Keycloak 能验证 SP 的签名请求。

     * 在 Keycloak 的 SAML Client 中通常有“Import Service Provider Metadata”按钮：可直接上传 `imart-sp-metadata.xml`，Keycloak 会自动导入 SP 的 EntityID、ACS、证书等信息。
6. 下载 Keycloak 的 IdP Metadata：

   * `https://keycloak.example.com/realms/<realm-name>/protocol/saml/descriptor`
   * 将此文件导入到 intra-mart 的 IdP 配置中（或让 intra-mart 管理页面使用该 URL 导入）。

---

## 七、metadata.xml 示例（便于互相导入）

下面给出简化的示例，切记用真实系统生成的 metadata 替换示例内容。

### 1) intra-mart (SP) metadata 示例（imart-sp-metadata.xml）

```xml
<EntityDescriptor entityID="https://imart.example.com/imart/sp" xmlns="urn:oasis:names:tc:SAML:2.0:metadata">
  <SPSSODescriptor AuthnRequestsSigned="true" WantAssertionsSigned="true" protocolSupportEnumeration="urn:oasis:names:tc:SAML:2.0:protocol">
    <AssertionConsumerService Binding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST" Location="https://imart.example.com/imart/sso/saml/SSO/alias/intra-mart-sp" index="1" />
    <SingleLogoutService Binding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect" Location="https://imart.example.com/imart/sso/saml/SLO"/>
    <KeyDescriptor use="signing">
      <KeyInfo xmlns="http://www.w3.org/2000/09/xmldsig#">
        <X509Data>
          <X509Certificate>MIID...BASE64...QAB</X509Certificate>
        </X509Data>
      </KeyInfo>
    </KeyDescriptor>
  </SPSSODescriptor>
</EntityDescriptor>
```

### 2) Keycloak (IdP) metadata 示例（从 Keycloak 导出）

Keycloak 自行生成，形如：

```xml
<EntityDescriptor entityID="https://keycloak.example.com/realms/intra-mart" xmlns="urn:oasis:names:tc:SAML:2.0:metadata">
  <IDPSSODescriptor protocolSupportEnumeration="urn:oasis:names:tc:SAML:2.0:protocol">
    <SingleSignOnService Binding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect" Location="https://keycloak.example.com/realms/intra-mart/protocol/saml"/>
    <SingleLogoutService Binding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-Redirect" Location="https://keycloak.example.com/realms/intra-mart/protocol/saml"/>
    <KeyDescriptor use="signing">
      <KeyInfo xmlns="http://www.w3.org/2000/09/xmldsig#">
        <X509Data>
          <X509Certificate>MIIF...BASE64...==</X509Certificate>
        </X509Data>
      </KeyInfo>
    </KeyDescriptor>
  </IDPSSODescriptor>
</EntityDescriptor>
```

---

## 八、调试步骤与日志收集（重点）

当遇到 `invalid_signature` 或 `SigAlg was null` 错误，按下面顺序排查：

1. **捕获浏览器网络请求（最直接）**

   * 打开浏览器 DevTools → Network。
   * 从 intra-mart 点击 SSO 登录，观察跳转到 Keycloak 的请求。
   * 如果是 Redirect Binding（GET），URL 应包含：

     * `SAMLRequest=...`
     * `RelayState=...`（可选）
     * `SigAlg=...`（**必须**，若签名）
     * `Signature=...`（**必须**，若签名）
   * 如果 URL 中没有 `SigAlg`，说明 SP 没有正确签名（或使用 POST 绑定）。

2. **查看 Keycloak 日志**（你已看到）

   * `SigAlg was null` → Keycloak 收到 Redirect Binding 无 SigAlg。
   * `invalid_signature` → Keycloak 验签失败或证书未导入/不匹配。

3. **临时开关验证（仅测试）**

   * 在 Keycloak Client 中把 `Client Signature Required` 设为 `OFF`，再测试一次：

     * 若登录成功，明确是签名验证问题（记得测试后恢复设置）。

4. **验证证书是否匹配**

   * 在 Keycloak Client → Keys 中确保已正确上传 SP 的公钥证书（与 intra-mart 使用的私钥配对）。
   * 在 intra-mart 中确保已导入 Keycloak 的证书（用于验证 Assertion）。

5. **确认绑定方式一致**

   * 推荐：SP 使用 **HTTP-POST**（AuthnRequest 通过表单 POST），Keycloak `Force POST Binding` ON。
   * 若必须使用 Redirect（GET），确保 intra-mart 在 URL 中包含 `SigAlg` 与 `Signature` 参数且使用 Keycloak 支持的签名算法（RSA-SHA256）。

6. **签名算法一致性**

   * Keycloak 支持 `RSA_SHA1`, `RSA_SHA256`, `RSA_SHA512` 等。
   * 请确保 intra-mart 签名算法与 Keycloak 设置一致（推荐 `RSA_SHA256`）。

7. **重现时采集完整请求**

   * 抓取完整 HTTP 请求（包括查询字符串或 POST body）并保存，用于断言与比对。可用 tcpdump 或浏览器复制 cURL：

```bash
# 在浏览器 Network 中右键复制 cURL（作为检查）
# 或使用 tshark/tcpdump 在服务器上抓包
```

---

## 九、典型故障案例与解决方法

### A. 错误：`SigAlg was null`

**原因**：SP 发起 Redirect Binding 时没有在 URL 中带 `SigAlg`。
**解决**：

* 把 intra-mart SSO Binding 改为 `HTTP-POST`，或
* 在 intra-mart 中开启 AuthnRequest 签名并确保 SigAlg 与 Signature 出现在 URL（如果必须用 Redirect）。

### B. 错误：`invalid_signature`（Keycloak 报验签失败）

**原因**：上传到 Keycloak 的 SP 公钥与 SP 实际用于签名的私钥不匹配，或签名算法不一致。
**解决**：

* 在 intra-mart 导出并重新上传 SP 的公钥至 Keycloak（通过 metadata 或证书文件）。
* 确认签名算法一致（RSA-SHA256）。

### C. 错误：`Invalid Audience`、`Issuer mismatch` 等

**原因**：SP EntityID（Issuer）或 ACS URL 与 Keycloak 中配置不一致。
**解决**：

* 在 Keycloak Client 中把 `Client ID` / `Valid Redirect URIs` 与 intra-mart 的 SP Metadata 一致化。

---

## 十、测试用例（快速验证）

1. 在 intra-mart 管理界面禁用 AuthnRequest 签名，Keycloak 中关闭 `Client Signature Required` → 验证能否完成登录（仅测试）。
2. 启用签名：生成证书，上传到双方，使用 POST Binding，再次测试。
3. 使用浏览器开发者工具验证回调（POST 表单）中 `SAMLResponse` 存在并且 POST 到 ACS URL。
4. 查看 Keycloak 日志是否有 `type="LOGIN"`（成功）或错误日志（失败）。

---

## 十一、附录：快速检查清单（Checklist）

* [ ] intra-mart 使用 HTTP-POST Binding
* [ ] intra-mart 启用 AuthnRequest 签名（RSA-SHA256）
* [ ] intra-mart 的 SP 公钥已导出并上传到 Keycloak
* [ ] Keycloak Client `Client Signature Required` = ON
* [ ] Keycloak `Force POST Binding` = ON
* [ ] Keycloak 对应 Client 的 `Valid Redirect URIs` 包含 ACS URL
* [ ] Keycloak IdP metadata 已导入到 intra-mart
* [ ] NameID Format 双方一致
* [ ] 证书有效期/配对检查通过

---
