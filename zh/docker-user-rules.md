---
title: DOCKER-USER 网络路由规则的追加与删除
description: DOCKER-USER 网络路由规则的追加与删除`
published: 1
date: 2024-12-04T10:16:46.275Z
tags: docker, iptables, network, route, user
editor: markdown
dateCreated: 2024-12-04T09:25:44.609Z
---

## DOCKER-USER 链规则的增加与删除说明 🛠️

Docker 使用 iptables 来管理网络流量。`DOCKER-USER` 链是一个特殊的链，用于添加与 Docker 容器相关的自定义规则。这个链在 Docker 生成的规则之前被评估。

---

### 1. 规则的增加 ➕ 

要向 `DOCKER-USER` 链添加规则，可以使用 `iptables` 命令。添加规则的一般格式如下：

```bash
sudo iptables -I DOCKER-USER -s <source_ip> -j <target_action>
```

##### 例子：
添加一个拒绝特定 IP 地址流量的规则：
```bash
sudo iptables -I DOCKER-USER -s 192.168.1.100 -j DROP
```
该命令将拒绝来自 192.168.1.100 的所有流量。

### 2. 规则的删除 ➖
要从 DOCKER-USER 链中删除规则，可以使用 iptables 命令的 -D 选项。删除规则时需要指定规则编号。

##### 显示规则 📜
首先，显示当前 DOCKER-USER 链中的规则：

```bash
sudo iptables -L DOCKER-USER --line-numbers
```

##### 删除规则
要删除特定规则，请指定其规则编号：
```bash
sudo iptables -D DOCKER-USER <rule_number>
```

##### 例子：
要删除规则编号为 1 的规则：
```bash
sudo iptables -D DOCKER-USER 1
```
还可以通过匹配条件删除规则：
```bash
# 删除DOCKER-USER 从10.13.13.0/24到 10.0.0.0/24的双向通信规则
sudo iptables -D DOCKER-USER -s 10.13.13.0/24 -d 10.0.0.0/24 -j ACCEPT
sudo iptables -D DOCKER-USER -s 10.0.0.0/24 -d 10.13.13.0/24 -j ACCEPT
```


### 总结 📝
- DOCKER-USER 链: 用于添加 Docker 容器流量的自定义规则的链。
- 规则的增加:
  - 命令: sudo iptables -I DOCKER-USER -s <source_ip> -j <target_action>
- 规则的删除:
  - 显示规则: sudo iptables -L DOCKER-USER --line-numbers
  - 删除命令: sudo iptables -D DOCKER-USER <rule_number>
  
## IP Route Rules 削除方法 🚦

### 1. Current Routes表示 📜

Before deleting a route, you may want to view the current routing table:

```bash
ip route show
```
### 2. 削除指定 Route ➖
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

## 如何删除 iptables NAT 规则 🚫

要删除 `iptables` 中的 NAT 规则，您需要知道特定的链（如 `PREROUTING`、`POSTROUTING` 或 `OUTPUT`）以及您想要移除的规则。以下是操作步骤：

### 1. 显示当前 NAT 规则 📜

首先，您可以使用以下命令查看当前的 NAT 规则：

```bash
sudo iptables -t nat -L -n --line-numbers
```
### 2. 删除特定 NAT 规则 ➖
要删除特定的 NAT 规则，请使用以下命令格式：

```bash
sudo iptables -t nat -D <chain> <rule_number>
#ex:
#sudo iptables -t nat -L -n --line-numbers
#>21   MASQUERADE  all  --  10.13.13.0/24        172.27.0.0/16
sudo iptables -t nat -D POSTROUTING  21
```
