---
title: saml-sso
description: Keycloak as SAML IdP, providing SSO authentication service for intra-mart Accel Platform
published: true
date: 2025-10-29T05:17:44.223Z
tags: saml, sso, keycloak, iap
editor: markdown
dateCreated: 2025-10-29T05:14:33.027Z
---

# SAML Signature Configuration and Troubleshooting Guide for Keycloak as IdP and intra-mart Accel Platform as SP

> Objective: Establish stable SAML 2.0 SSO between Keycloak (IdP) and intra-mart Accel Platform (SP), ensuring correct signing/verification of AuthnRequest and Assertion, and avoiding errors such as `invalid_signature` / `SigAlg was null`.

---

## I. Target Audience for this Document

* Engineers/operations responsible for SSO configuration.
* Those who have Keycloak and intra-mart deployed and can access both administration interfaces.

---

## II. Prerequisites (Example Domains)

* Keycloak: `https://keycloak.example.com` (Realm: `intra-mart`)
* intra-mart: `https://imart.example.com`
* Use HTTPS (strongly recommended)

---

## III. Overall Flow Overview

1. Generate or export SP Metadata in intra-mart (including SP EntityID, ACS, SP Public Key).
2. Create a new SAML Client in Keycloak and import SP Metadata / upload SP Certificate.
3. Configure signing/encryption policies in Keycloak (Sign Assertions / Sign Documents / Signature Algorithm).
4. Import Keycloak's IdP Metadata into intra-mart (Keycloak's `.../protocol/saml/descriptor`).
5. Enable AuthnRequest signing and POST Binding (recommended) in intra-mart, and select the signature algorithm (RSA-SHA256).
6. Test login and troubleshoot item by item based on logs.

---

## IV. Certificates and Keys (Generation Example)

The following example demonstrates how to generate a self-signed certificate for the SP (intra-mart) to sign AuthnRequest.

```bash
# Generate SP private key
openssl genpkey -algorithm RSA -out imart_sp.key -pkeyopt rsa_keygen_bits:2048

# Generate CSR
openssl req -new -key imart_sp.key -out imart_sp.csr \
  -subj "/C=JP/ST=Tokyo/L=Tokyo/O=Example/OU=IT/CN=imart.example.com"

# Generate self-signed certificate (valid for 1 year)
openssl x509 -req -days 365 -in imart_sp.csr -signkey imart_sp.key -out imart_sp.crt

# Convert certificate to PEM (if needed)
openssl x509 -in imart_sp.crt -out imart_sp.pem -outform PEM
```

> In a production environment, it is recommended to use certificates issued by an internal CA or a trusted CA, not self-signed certificates.

---

## V. intra-mart (SP) Side Setup Steps (Recommended Configuration)

> The following are common intra-mart administration UI field names (Japanese interface descriptions) and recommended values. Your intra-mart version or customized UI may vary slightly.

1. Log in to the intra-mart administration console (system administrator).
2. Go to: **システム管理 → 認証 → SAML 連携設定** (System Management → Authentication → SAML Federation Settings) (or similar path).
3. Create or edit SP configuration:

   * **SP Entity ID**: `https://imart.example.com/imart/sp`
   * **AssertionConsumerService (ACS) URL**: `https://imart.example.com/imart/sso/saml/SSO/alias/intra-mart-sp` (based on actual URL)
   * **NameID Format**: `urn:oasis:names:tc:SAML:1.1:nameid-format:unspecified` (or `emailAddress`)
   * **SSO Binding**: **HTTP-POST** (recommended)
   * **Signature of AuthnRequest (AuthnRequest署名)**: **ON (有効)** (Enabled)
   * **Signature Algorithm (署名アルゴリズム)**: `RSA-SHA256` (consistent with Keycloak)
   * **SP Certificate (Public Key)**: Upload the content of `imart_sp.crt` or `imart_sp.pem`
   * **WantAuthnRequestsSigned**: Set to `true` (if this item exists)
4. Export or download SP metadata (there is usually a button to generate `metadata.xml`), save as `imart-sp-metadata.xml`.

> Note: If intra-mart's default is Redirect Binding (GET), please change it to POST. If SigAlg/Signature is not included in Redirect Binding, a `SigAlg was null` error will be triggered.

---

## VI. Keycloak (IdP) Side Setup Steps

1. Log in to the Keycloak administration console: `https://keycloak.example.com/auth/admin/` (path may vary depending on the version).
2. Select or create a Realm (example: `intra-mart`).
3. Create a new Client:

   * **Client ID**: It is recommended to use the SP Entity ID, such as `https://imart.example.com/imart/sp`.
   * **Client Protocol**: Select `saml`.
   * **Save**.
4. Edit Client → `Settings`:

   * **Enabled**: ON
   * **Client Signature Required**: ON (requires verification of AuthnRequest signature from SP)
   * **Force POST Binding**: ON (forces the use of HTTP-POST)
   * **Sign Documents**: ON
   * **Sign Assertions**: ON
   * **SAML Signature Algorithm**: `RSA_SHA256` (or `RSA_SHA512`, but consistent with SP)
   * **Root URL / Base URL**: `https://imart.example.com/` (used to generate links)
   * **Valid Redirect URIs**: `https://imart.example.com/*` or a more precise ACS path
5. Switch to the `Keys` or `Certificates` tab (UI name varies by Keycloak version):

   * Upload the SP's public key/certificate (`imart_sp.crt`) so that Keycloak can verify the SP's signed requests.

     * In Keycloak's SAML Client, there is usually an "Import Service Provider Metadata" button: you can directly upload `imart-sp-metadata.xml`, and Keycloak will automatically import the SP's EntityID, ACS, certificate, and other information.
6. Download Keycloak's IdP Metadata:

   * `https://keycloak.example.com/realms/<realm-name>/protocol/saml/descriptor`
   * Import this file into intra-mart's IdP configuration (or have the intra-mart administration page import it using this URL).

---

## VII. metadata.xml Examples (for easy mutual import)

Below are simplified examples; remember to replace the example content with metadata generated by your actual system.

### 1) intra-mart (SP) metadata example (imart-sp-metadata.xml)

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

### 2) Keycloak (IdP) metadata example (exported from Keycloak)

Keycloak generates it itself, in the form of:

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

## VIII. Debugging Steps and Log Collection (Key Points)

When encountering `invalid_signature` or `SigAlg was null` errors, troubleshoot in the following order:

1. **Capture browser network requests (most direct)**

   * Open browser DevTools → Network.
   * Click SSO login from intra-mart and observe the request redirected to Keycloak.
   * If it is Redirect Binding (GET), the URL should contain:

     * `SAMLRequest=...`
     * `RelayState=...` (optional)
     * `SigAlg=...` (**required**, if signed)
     * `Signature=...` (**required**, if signed)
   * If `SigAlg` is not in the URL, it means the SP did not sign correctly (or used POST binding).

2. **Check Keycloak logs** (you have already seen)

   * `SigAlg was null` → Keycloak received Redirect Binding without SigAlg.
   * `invalid_signature` → Keycloak signature verification failed or certificate not imported/mismatched.

3. **Temporary switch for verification (test only)**

   * In Keycloak Client, set `Client Signature Required` to `OFF` and test again:

     * If login is successful, it is clearly a signature verification issue (remember to restore settings after testing).

4. **Verify certificate matching**

   * In Keycloak Client → Keys, ensure that the SP's public key certificate has been correctly uploaded (paired with the private key used by intra-mart).
   * In intra-mart, ensure that Keycloak's certificate has been imported (used to verify Assertion).

5. **Confirm consistent binding method**

   * Recommended: SP uses **HTTP-POST** (AuthnRequest via form POST), Keycloak `Force POST Binding` ON.
   * If Redirect (GET) must be used, ensure that intra-mart includes `SigAlg` and `Signature` parameters in the URL and uses a signature algorithm supported by Keycloak (RSA-SHA256).

6. **Signature algorithm consistency**

   * Keycloak supports `RSA_SHA1`, `RSA_SHA256`, `RSA_SHA512`, etc.
   * Please ensure that the intra-mart signature algorithm is consistent with the Keycloak settings (RSA-SHA256 recommended).

7. **Collect full request when reproducing**

   * Capture and save the complete HTTP request (including query string or POST body) for assertion and comparison. You can use tcpdump or copy cURL from the browser:

```bash
# Right-click and copy cURL in browser Network (for inspection)
# Or use tshark/tcpdump to capture packets on the server
```

---

## IX. Typical Failure Cases and Solutions

### A. Error: `SigAlg was null`

**Cause**: SP initiated Redirect Binding without `SigAlg` in the URL.
**Solution**:

* Change intra-mart SSO Binding to `HTTP-POST`, or
* Enable AuthnRequest signing in intra-mart and ensure SigAlg and Signature appear in the URL (if Redirect must be used).

### B. Error: `invalid_signature` (Keycloak reports signature verification failure)

**Cause**: The SP public key uploaded to Keycloak does not match the private key actually used by the SP for signing, or the signature algorithms are inconsistent.
**Solution**:

* Export and re-upload the SP's public key to Keycloak from intra-mart (via metadata or certificate file).
* Confirm signature algorithm consistency (RSA-SHA256).

### C. Error: `Invalid Audience`, `Issuer mismatch`, etc.

**Cause**: SP EntityID (Issuer) or ACS URL is inconsistent with the configuration in Keycloak.
**Solution**:

* In Keycloak Client, make `Client ID` / `Valid Redirect URIs` consistent with intra-mart's SP Metadata.

---

## X. Test Cases (Quick Verification)

1. Disable AuthnRequest signing in the intra-mart administration interface, and turn off `Client Signature Required` in Keycloak → Verify if login can be completed (test only).
2. Enable signing: Generate certificates, upload to both parties, use POST Binding, and test again.
3. Use browser developer tools to verify that `SAMLResponse` exists in the callback (POST form) and is POSTed to the ACS URL.
4. Check Keycloak logs for `type="LOGIN"` (success) or error logs (failure).

---

## XI. Appendix: Quick Checklist

* [ ] intra-mart uses HTTP-POST Binding
* [ ] intra-mart enables AuthnRequest signing (RSA-SHA256)
* [ ] intra-mart's SP public key has been exported and uploaded to Keycloak
* [ ] Keycloak Client `Client Signature Required` = ON
* [ ] Keycloak `Force POST Binding` = ON
* [ ] Keycloak's corresponding Client's `Valid Redirect URIs` include the ACS URL
* [ ] Keycloak IdP metadata has been imported into intra-mart
* [ ] NameID Format is consistent on both sides
* [ ] Certificate validity/pairing check passed

---
