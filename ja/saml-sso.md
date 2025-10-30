---
title: saml-sso
description: Keycloak を SAML IdP として、intra-mart Accel Platform に SSO 認証サービスを提供
published: true
date: 2025-10-29T05:17:44.223Z
tags: saml, sso, keycloak, iap
editor: markdown
dateCreated: 2025-10-29T05:14:33.027Z
---

# Keycloak (IdP) と intra-mart Accel Platform (SP) の SAML 署名設定とトラブルシューティングガイド

> 目的：Keycloak (IdP) と intra-mart Accel Platform (SP) 間で安定した SAML 2.0 SSO を確立し、AuthnRequest と Assertion の署名/検証が正しく行われ、`invalid_signature` / `SigAlg was null` などのエラーを回避する。

---

## 一、ドキュメントの対象者

* SSO 設定を担当するエンジニア/運用担当者。
* Keycloak と intra-mart がデプロイされており、両方の管理インターフェースにアクセスできること。

---

## 二、前提条件（ドメイン例）

* Keycloak: `https://keycloak.example.com`（レルム: `intra-mart`）
* intra-mart: `https://imart.example.com`
* HTTPS の使用（強く推奨）

---

## 三、全体的なフローの概要

1. intra-mart で SP メタデータ（SP EntityID、ACS、SP 公開鍵を含む）を生成またはエクスポートする。
2. Keycloak で SAML クライアントを新規作成し、SP メタデータをインポート / SP 証明書をアップロードする。
3. Keycloak で署名/暗号化ポリシー（Sign Assertions / Sign Documents / Signature Algorithm）を設定する。
4. intra-mart で Keycloak の IdP メタデータ（Keycloak の `.../protocol/saml/descriptor`）をインポートする。
5. intra-mart で AuthnRequest 署名と POST Binding（推奨）を有効にし、署名アルゴリズム（RSA-SHA256）を選択する。
6. ログインをテストし、ログに基づいて項目ごとにトラブルシューティングを行う。

---

## 四、証明書と鍵（生成例）

以下の例は、SP（intra-mart）が AuthnRequest に署名するために自己署名証明書を生成する方法を示しています。

```bash
# SP 秘密鍵の生成
openssl genpkey -algorithm RSA -out imart_sp.key -pkeyopt rsa_keygen_bits:2048

# CSR の生成
openssl req -new -key imart_sp.key -out imart_sp.csr \
  -subj "/C=JP/ST=Tokyo/L=Tokyo/O=Example/OU=IT/CN=imart.example.com"

# 自己署名証明書の生成（1 年間有効）
openssl x509 -req -days 365 -in imart_sp.csr -signkey imart_sp.key -out imart_sp.crt

# 必要に応じて証明書を PEM 形式に変換
openssl x509 -in imart_sp.crt -out imart_sp.pem -outform PEM
```

> 本番環境では、自己署名証明書ではなく、社内 CA または信頼できる CA が発行した証明書を使用することを推奨します。

---

## 五、intra-mart（SP）側の設定手順（推奨設定）

> 以下は、一般的な intra-mart 管理 UI のフィールド名（日本語インターフェースの説明）と推奨値です。お使いの intra-mart のバージョンやカスタマイズされた UI によっては、若干異なる場合があります。

1. intra-mart 管理コンソール（システム管理者）にログインします。
2. 移動先：**システム管理 → 認証 → SAML 連携設定**（または類似のパス）。
3. SP 設定を新規作成または編集します。

   * **SP Entity ID**：`https://imart.example.com/imart/sp`
   * **AssertionConsumerService (ACS) URL**：`https://imart.example.com/imart/sso/saml/SSO/alias/intra-mart-sp`（実際の URL に基づく）
   * **NameID Format**：`urn:oasis:names:tc:SAML:1.1:nameid-format:unspecified`（または `emailAddress`）
   * **SSO Binding**：**HTTP-POST**（推奨）
   * **Signature of AuthnRequest (AuthnRequest署名)**：**ON（有効）**
   * **Signature Algorithm (署名アルゴリズム)**：`RSA-SHA256`（Keycloak と一致させる）
   * **SP 証明書（公開鍵）**：`imart_sp.crt` または `imart_sp.pem` の内容をアップロード
   * **WantAuthnRequestsSigned**：`true` に設定（この項目が存在する場合）
4. SP メタデータ（通常は `metadata.xml` を生成するボタンがあります）をエクスポートまたはダウンロードし、`imart-sp-metadata.xml` として保存します。

> 注意：intra-mart のデフォルトが Redirect Binding（GET）の場合、POST に変更してください。Redirect Binding で SigAlg/Signature が含まれていない場合、`SigAlg was null` エラーが発生します。

---

## 六、Keycloak（IdP）側の設定手順

1. Keycloak 管理コンソールにログインします：`https://keycloak.example.com/auth/admin/`（パスはバージョンによって異なります）。
2. レルムを選択または作成します（例：`intra-mart`）。
3. クライアントを新規作成します。

   * **Client ID**：SP Entity ID を使用することを推奨します（例：`https://imart.example.com/imart/sp`）。
   * **Client Protocol**：`saml` を選択します。
   * **Save**。
4. クライアントを編集 → `Settings`：

   * **Enabled**：ON
   * **Client Signature Required**：ON（SP からの AuthnRequest 署名の検証を要求）
   * **Force POST Binding**：ON（HTTP-POST の強制使用）
   * **Sign Documents**：ON
   * **Sign Assertions**：ON
   * **SAML Signature Algorithm**：`RSA_SHA256`（または `RSA_SHA512`、ただし SP と一致させる）
   * **Root URL / Base URL**：`https://imart.example.com/`（リンク生成に使用）
   * **Valid Redirect URIs**：`https://imart.example.com/*` またはより正確な ACS パス
5. `Keys` または `Certificates` タブに切り替えます（Keycloak のバージョンによって UI 名が異なります）。

   * SP の公開鍵/証明書（`imart_sp.crt`）をアップロードし、Keycloak が SP の署名要求を検証できるようにします。

     * Keycloak の SAML クライアントには通常、「Import Service Provider Metadata」ボタンがあります。`imart-sp-metadata.xml` を直接アップロードすると、Keycloak は SP の EntityID、ACS、証明書などの情報を自動的にインポートします。
6. Keycloak の IdP メタデータをダウンロードします。

   * `https://keycloak.example.com/realms/<realm-name>/protocol/saml/descriptor`
   * このファイルを intra-mart の IdP 設定にインポートします（または intra-mart 管理ページでこの URL を使用してインポートさせます）。

---

## 七、metadata.xml の例（相互インポートを容易にするため）

以下に簡略化された例を示しますが、必ず実際のシステムで生成されたメタデータで置き換えてください。

### 1) intra-mart (SP) メタデータの例（imart-sp-metadata.xml）

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

### 2) Keycloak (IdP) メタデータの例（Keycloak からエクスポート）

Keycloak が自己生成する形式で、以下のようになります。

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

## 八、デバッグ手順とログ収集（重要）

`invalid_signature` または `SigAlg was null` エラーが発生した場合、以下の順序でトラブルシューティングを行います。

1. **ブラウザのネットワークリクエストをキャプチャする（最も直接的）**

   * ブラウザの DevTools → Network を開きます。
   * intra-mart から SSO ログインをクリックし、Keycloak へのリダイレクト要求を観察します。
   * Redirect Binding（GET）の場合、URL には以下が含まれている必要があります。

     * `SAMLRequest=...`
     * `RelayState=...`（オプション）
     * `SigAlg=...`（**必須**、署名されている場合）
     * `Signature=...`（**必須**、署名されている場合）
   * URL に `SigAlg` がない場合、SP が正しく署名していない（または POST Binding を使用している）ことを意味します。

2. **Keycloak のログを確認する**（すでに確認済み）

   * `SigAlg was null` → Keycloak が SigAlg なしの Redirect Binding を受信した。
   * `invalid_signature` → Keycloak の署名検証が失敗したか、証明書がインポートされていない/一致しない。

3. **一時的な検証の切り替え（テストのみ）**

   * Keycloak クライアントで `Client Signature Required` を `OFF` に設定し、もう一度テストします。

     * ログインが成功した場合、署名検証の問題であることが明確になります（テスト後に設定を元に戻すことを忘れないでください）。

4. **証明書が一致しているか検証する**

   * Keycloak クライアント → Keys で、SP の公開鍵証明書が正しくアップロードされていることを確認します（intra-mart が使用する秘密鍵とペアになっていること）。
   * intra-mart で Keycloak の証明書がインポートされていることを確認します（Assertion の検証に使用）。

5. **バインディング方式の一貫性を確認する**

   * 推奨：SP は **HTTP-POST**（AuthnRequest はフォーム POST 経由）、Keycloak は `Force POST Binding` を ON にします。
   * Redirect（GET）をどうしても使用する必要がある場合、intra-mart が URL に `SigAlg` と `Signature` パラメータを含み、Keycloak がサポートする署名アルゴリズム（RSA-SHA256）を使用していることを確認します。

6. **署名アルゴリズムの一貫性**

   * Keycloak は `RSA_SHA1`, `RSA_SHA256`, `RSA_SHA512` などをサポートしています。
   * intra-mart の署名アルゴリズムが Keycloak の設定と一致していることを確認してください（`RSA_SHA256` を推奨）。

7. **再現時に完全なリクエストを収集する**

   * 完全な HTTP リクエスト（クエリ文字列または POST ボディを含む）をキャプチャして保存し、アサーションと比較するために使用します。 tcpdump またはブラウザの cURL コピーを使用できます。

```bash
# ブラウザのネットワークで cURL を右クリックしてコピー（確認用）
# または tshark/tcpdump を使用してサーバーでパケットをキャプチャ
```

---

## 九、典型的な障害ケースと解決策

### A. エラー：`SigAlg was null`

**原因**：SP が Redirect Binding を開始したときに、URL に `SigAlg` が含まれていない。
**解決策**：

* intra-mart の SSO Binding を `HTTP-POST` に変更するか、
* intra-mart で AuthnRequest 署名を有効にし、SigAlg と Signature が URL に表示されるようにする（Redirect を使用する必要がある場合）。

### B. エラー：`invalid_signature`（Keycloak が署名検証失敗を報告）

**原因**：Keycloak にアップロードされた SP 公開鍵が、SP が実際に署名に使用した秘密鍵と一致しないか、署名アルゴリズムが一致しない。
**解決策**：

* intra-mart で SP の公開鍵をエクスポートし、Keycloak に再アップロードする（メタデータまたは証明書ファイル経由）。
* 署名アルゴリズムが一致していることを確認する（RSA-SHA256）。

### C. エラー：`Invalid Audience`、`Issuer mismatch` など

**原因**：SP EntityID（Issuer）または ACS URL が Keycloak の設定と一致しない。
**解決策**：

* Keycloak クライアントで `Client ID` / `Valid Redirect URIs` を intra-mart の SP メタデータと一致させる。

---

## 十、テストケース（迅速な検証）

1. intra-mart 管理画面で AuthnRequest 署名を無効にし、Keycloak で `Client Signature Required` をオフにする → ログインが完了するか検証する（テストのみ）。
2. 署名を有効にする：証明書を生成し、両方にアップロードし、POST Binding を使用して再度テストする。
3. ブラウザの開発者ツールを使用して、コールバック（POST フォーム）に `SAMLResponse` が存在し、ACS URL に POST されていることを検証する。
4. Keycloak のログに `type="LOGIN"`（成功）またはエラーログ（失敗）があるか確認する。

---

## 十一、付録：クイックチェックリスト（Checklist）

* [ ] intra-mart は HTTP-POST Binding を使用している
* [ ] intra-mart は AuthnRequest 署名（RSA-SHA256）を有効にしている
* [ ] intra-mart の SP 公開鍵はエクスポートされ、Keycloak にアップロードされている
* [ ] Keycloak クライアントの `Client Signature Required` = ON
* [ ] Keycloak の `Force POST Binding` = ON
* [ ] Keycloak の対応するクライアントの `Valid Redirect URIs` に ACS URL が含まれている
* [ ] Keycloak IdP メタデータは intra-mart にインポートされている
* [ ] NameID Format は両者で一致している
* [ ] 証明書の有効期限/ペアリングチェックが通過している

---
