---
title: Docker user route rules of network 
description: docker user route rules of network 
published: 1
date: 2024-12-04T10:19:04.679Z
tags: docker, iptables, network, route, user
editor: markdown
dateCreated: 2024-12-04T09:29:57.323Z
---

## DOCKER-USER チェーンのルールの追加と削除

Docker は、iptables を使用してネットワークトラフィックを管理します。`DOCKER-USER` チェーンは、Docker コンテナに関連するトラフィックに対するカスタムルールを追加するための特別なチェーンです。このチェーンは、Docker が生成したルールの前に評価されます。

---

### 1. ルールの追加

`DOCKER-USER` チェーンにルールを追加するには、`iptables` コマンドを使用します。以下は、ルールを追加する一般的な形式です：

```bash
sudo iptables -I DOCKER-USER -s <source_ip> -j <target_action>
```


##### 例:
特定の IP アドレスからのトラフィックを拒否するルールを追加する場合：
```bash
sudo iptables -I DOCKER-USER -s 192.168.1.100 -j DROP
```
このコマンドは、`192.168.1.100 `からのすべてのトラフィックを拒否します。

### 2. ルールの削除
`DOCKER-USER` チェーンからルールを削除するには、`iptables` コマンドの` -D` オプションを使用します。ルールを削除するには、削除したいルールの番号を指定する必要があります。

##### ルールの一覧表示
まず、現在の `DOCKER-USER` チェーンのルールを表示します：

```bash
sudo iptables -L DOCKER-USER --line-numbers
```

##### ルールの削除
特定のルールを削除するには、そのルールの番号を指定します：
```bash
sudo iptables -D DOCKER-USER <rule_number>
```

##### 例:
ルール番号が 1 のルールを削除する場合：
```bash
sudo iptables -D DOCKER-USER 1
```

### まとめ 📝
- DOCKER-USER チェーン: Docker コンテナのトラフィックに対するカスタムルールを追加するためのチェーン。
- ルールの追加:
  - Command: `sudo iptables -I DOCKER-USER -s <source_ip> -j <target_action>`
- ルールの削除:
  - ルールの一覧表示: `sudo iptables -L DOCKER-USER --line-numbers`
  - 削除コマンド: `sudo iptables -D DOCKER-USER <rule_number>`
  
## IP Route Rules 削除方法 🚦

### 1. Current Routesを表示 📜

Before deleting a route, you may want to view the current routing table:

```bash
ip route show
```
### 2. 指定 Routeを削除 ➖
To delete a specific route, use the following command format:
```bash
sudo ip route del <destination> [via <gateway>] [dev <interface>]
```
##### Example:
To delete a route to the network 192.168.1.0/24:
```bash
sudo ip route del 192.168.1.0/24
sudo ip route del 10.13.13.0/24 dev docker0 scope link
```

## iptables NAT ルールを削除 🚫

`iptables` NAT　Rules,(例: `PREROUTING`、`POSTROUTING` 或 `OUTPUT`）を `-L -n --line-numer`で取得

### 1. NAT ルールを表示 📜

```bash
sudo iptables -t nat -L -n --line-numbers
```
### 2. 特定 NAT を削除 ➖

```bash
sudo iptables -t nat -D <chain> <rule_number>
#ex:
#sudo iptables -t nat -L -n --line-numbers
#>21   MASQUERADE  all  --  10.13.13.0/24        172.27.0.0/16
sudo iptables -t nat -D POSTROUTING  21
```
