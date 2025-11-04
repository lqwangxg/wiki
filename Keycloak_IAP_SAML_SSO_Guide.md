# Keycloak + intra-mart Accel Platform (IAP)  SAML SSO 认证完整配置手册与异常排查指南

---

## 📘 一、概述

本文档详细说明如何在 **Keycloak** 与 **intra-mart Accel Platform (IAP)** 之间实现 **SAML SSO（单点登录）** 集成，涵盖：

- SAML 配置步骤
- 证书生成方法
- IAP 与 Keycloak 的集成设定
- 常见异常及排查方法

---

## 🏗️ 二、前提条件

| 项目 | 要求 |
|------|------|
| intra-mart 版本 | Accel Platform 2019+ |
| Keycloak 版本 | 17 及以上（WildFly 或 Quarkus 版均可） |
| 协议 | SAML 2.0 |
| 网络连通性 | IAP 可以访问 Keycloak 的 `/auth/realms/{realm}/protocol/saml` |
| 时钟同步 | IAP 与 Keycloak 服务器需保持时间同步（误差 ≤ 1 分钟） |

---

## 🔑 三、SAML 证书生成

IAP 作为 Service Provider (SP)，需提供自签名证书给 Keycloak。

```bash
# 1. 生成私钥与自签名证书
openssl req -newkey rsa:2048 -nodes -keyout imart_sp.key -x509 -days 3650 -out imart_sp.crt

# 2. 合并为 PEM 文件（供 IAP 使用）
cat imart_sp.crt imart_sp.key > imart_sp.pem
```

生成后将文件放置于：
```
/imart/conf/saml/
  ├── imart_sp.crt #公钥
  ├── imart_sp.key #私钥
  └── imart_sp.pem #公钥pem版
```

---
## ⚙️ 四、Keycloak 配置

### 1️⃣ 创建 Realm 与用户

- 登录 Keycloak 管理控制台
- 创建新的 Realm（例如：`intra-mart`）
  - ![image](/uploads/554300e91c213d5812a248987e87f647/image.png)
- 在该 Realm 下创建用户（例如：`wangxg`）
- 为用户设置密码并启用账户
  - ![image](/uploads/c92e7422772547d8f916980b105ad905/image.png)
### 2️⃣ 导出IdP metadata xml： `keycloak_saml_metadata.xml` 
- ![image](/uploads/d90b9773d4d54bacce7b94d93a93a6d7/image.png)
- 示例：SAML Metadata（IdP）
```xml
<md:EntityDescriptor xmlns="urn:oasis:names:tc:SAML:2.0:metadata" xmlns:md="urn:oasis:names:tc:SAML:2.0:metadata" xmlns:saml="urn:oasis:names:tc:SAML:2.0:assertion" xmlns:ds="http://www.w3.org/2000/09/xmldsig#" entityID="https://keycloak.its2.mbpsmartec.co.jp/realms/intra-mart">
  <md:IDPSSODescriptor WantAuthnRequestsSigned="true" protocolSupportEnumeration="urn:oasis:names:tc:SAML:2.0:protocol">
    <md:KeyDescriptor use="signing">
      <ds:KeyInfo>
        <ds:KeyName>xxxxxx</ds:KeyName>
        <ds:X509Data>
          <ds:X509Certificate>xxxx</ds:X509Certificate>
        </ds:X509Data>
      </ds:KeyInfo>
    </md:KeyDescriptor>
```
---

## 🧩 五、intra-mart 配置

### 1️⃣ 启用 SAML 认证

登录 intra-mart 管理控制台：  
`https://sample.its2.mbpsmartec.co.jp/imart/system/login`

进入：  
**系统管理 > SAML认证设定 > IdP一览 **
![image](/uploads/57a1e8cd0b36df8edb4b0b209b8e4702/image.png)

### 2️⃣ 注册 IdP (Keycloak)

| 项目 | 内容 |
|------|------|
| IdP 名称 | Keycloak-for-aws|
| 证书导入 | 将 `keycloak_saml_metadata.xml`内容拷入IdPメタデータ  |
![image](/uploads/542d0da2e3f105e450988277b0c7db3e/image.png)

---
### 3️⃣ 注册 SP 信息
- 以下红框部分需要设定，其他项目保持默认值，公钥/私钥证书的内容分别拷贝到输入框中，保存。
  ![image](/uploads/107ea3a681ca9655f37714deaee75bdb/image.png)
- 下载SP meta xml imart-sp.xml
  ![image](/uploads/aeef8e354f1aa9443d8ca844625b6786/image.png)

## ⚙️ 六、Keycloak 导入client 信息： `imart-sp.xml`
- ![image](/uploads/f43b3f060c7d14746a1911271c519abf/image.png)
- ![image](/uploads/4f5511551c568a965d1375ef1f5b0fcf/image.png)
- ![image](/uploads/b522ebf0a5e33e9b966d1d23d0a0849e/image.png)
## ⚙️ 七、Keycloak 导入公钥证书信息.
- ![image](/uploads/78b407d0a06798fc1804878f3fefc097/image.png)
- ![image](/uploads/0af4b54051c36dc69b0ae0b5242504f8/image.png)
- `注意`： 证书导入后，自动更新keycloak,不要回到setting页面点保存. 保存按钮会重新生成新证书。
- 其他保持默认值

## ⚙️ 八、intar-mart端设定iap user和saml user的映射关系：
- ![image](/uploads/449513f383616f8403f7db89f7a8b84f/image.png)
- ![image](/uploads/d02d33577ea14c3c662e1b90ae087ab4/image.png)
- ![image](/uploads/8757623785b6de1e3ac0296d96b45d7d/image.png)
- ![image](/uploads/23d8c03efcff8e7dd980d5fa3d3193c6/image.png)
- **注意**： `两边必须都存在此用户`
### 4️⃣ 测试登录
- ![image](/uploads/88e8bec158b4193e6188badbd63e95a2/image.png)
- ![image](/uploads/ca3ba7000d4dabee40c13fe1dced151e/image.png)
- ![image](/uploads/782eae2e5e2609de160c2a7ed626c460/image.png)
---

## 🚨 九、常见异常与排查

| 异常现象 | log 信息 | 原因 | 对策 |
|-----------|-----------|------|------|
| 登录后出现 “Invalid Signature” | `org.opensaml.xmlsec.signature.support.SignatureException` | Keycloak 与 IAP 签名证书不一致 | 确认双方使用相同的公钥，重新导入 IdP Metadata |
| 登录后无限重定向 | 浏览器中不断重定向至 `/im_login/saml2/post/acs` | SAML Session 未保存或 Cookie 丢失 | 检查 IAP Session 配置和 Cookie 域名 |
| 出现 “Cannot parse SAMLResponse” | IAP log 中 `SAXParseException` | Keycloak 响应压缩格式不兼容 | 在 IAP SAML 设定中关闭“Response Compression” |
| 登录成功后用户名为空 | `No NameID found in Assertion` | Keycloak 未设置 NameID 格式 | 在 Client 设置中设定 NameID Format 为 `username` |
| 时间戳不一致错误 | `Invalid Conditions: NotBefore/NotOnOrAfter` | 服务器时间不同步 | 同步 NTP 或手动调整服务器时间 |
| “unknown entityID” 错误 | IAP 日志 `unknown entityID` | IAP 配置中的 EntityID 与 Keycloak Client ID 不一致 | 保证 Entity ID 与 Keycloak Client ID 一致 (`imart-sp`) |

---

## ✅ 十、验证结果

| 测试项 | 结果 |
|--------|------|
| 跳转至 Keycloak 登录页 | ✅ |
| 登录成功后返回 IAP | ✅ |
| IAP 获取到用户属性 | ✅ |
| 用户登出同步登出 Keycloak | ✅ |
