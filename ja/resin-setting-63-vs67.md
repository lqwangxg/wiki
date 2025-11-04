---
title: resin setting 4.063 vs 4.067
description: 
published: true
date: 2025-11-04T08:44:11.101Z
tags: resin
editor: markdown
dateCreated: 2025-11-04T08:44:11.101Z
---

# Resin-Pro 4.0.63 vs 4.0.67 の 設定変更レポート

---

## 1. `watchdog-port` の追加について

### 変更内容
```diff
- <server-multi id-prefix="web-" address-list="${web_servers}" port="6810" />
+ <server-multi id-prefix="web-" address-list="${web_servers}" port="6810" watchdog-port="${watchdog_port}"/>
```

### 概要
`watchdog-port` は **Resin の Watchdog プロセス**（JVM の起動・監視・再起動を行う軽量プロセス）と  
**実際の Resin サーバープロセス（JVM）** 間で通信を行うための制御ポートである。

### 役割
- Watchdog ⇔ Resin JVM の **心拍・状態監視・制御命令** 用ポート  
- クラスタ構成 (`<server-multi>`) において、各サーバーごとに一意なポートが必要。

### 設定有無の影響
| 状況 | 動作 |
|------|------|
| **設定あり（推奨）** | Watchdog と Resin JVM が確実に紐づき、クラスタ管理や再起動制御が安定する。 |
| **設定なし** | 自動割り当てだが、ポート衝突や複数ノード起動時の不安定動作の可能性がある。 |

**結論：** Resin 4.0.67 では `watchdog-port` の明示設定が推奨される。

---

## 2. `expire-timeout`（health.xml）の追加について

### 変更内容
```diff
- <health:LogService level="${health_log_level?:info}" />
+ <health:LogService level="${health_log_level?:info}" expire-timeout="${health_log_expire_timeout?:'14D'}"/>
```

### 概要
`expire-timeout` は **Health Log の保持期間**を指定する設定。  
Health Log は Resin の内部状態（スレッド数・CPU負荷・接続プール状態など）を継続記録する。

### 設定有無の影響
| 状況 | 動作 |
|------|------|
| **設定あり（推奨）** | 指定期間を過ぎた Health Log を自動削除。ディスク肥大を防止。 |
| **設定なし** | デフォルトではログが無期限保持され、長期運用時に容量肥大の可能性がある。 |

**結論：** `expire-timeout` は長期運用時のディスク肥大防止を目的としており、設定が推奨される。

---

## 3. `access_log_format` の優先順位

### 設定例
```properties
# resin.properties
access_log_format : %h %l %u %t "%r" %s %b "%{Referer}i" "%{User-Agent}i"
```

```xml
# resin.xml
<access-log path='/opt/im/log/resin-log/access.log'>
  <rollover-period>1D</rollover-period>
  <rollover-count>60</rollover-count>
  <format>%{X-Forwarded-For}i %h %D %l %u %t "%r" %>s %b "%{Referer}i" "%{User-Agent}i"</format>
</access-log>
```

### 優先順位
| 優先順位 | 設定場所 | 備考 |
|-----------|-----------|------|
| ① | **resin.xml の `<access-log>` 内に明示記述された `<format>`** | 最優先 |
| ② | **resin.properties の `access_log_format`** | resin.xml で参照された場合のみ有効 |
| ③ | **Resin 組み込みデフォルトフォーマット** | 設定なし時に使用 |

**結論：**  
`resin.xml` に `<format>` が直接記述されている場合、`resin.properties` の値は無視される。  
したがって、現構成では **resin.xml の設定が優先**される。

---

## 総括

| 項目 | 新設定 | 意味 | 設定有無の影響 | 推奨 |
|------|----------|------|------------------|------|
| `watchdog-port` | Watchdog ⇔ JVM 通信用ポート | ポート競合防止・安定化 | 明示設定推奨 |
| `expire-timeout` | Health Log の保持期間 | ディスク肥大防止 | 明示設定推奨（例：14D） |
| `access-log format` | resin.xml に直書き | resin.properties より優先 | 現状で問題なし |