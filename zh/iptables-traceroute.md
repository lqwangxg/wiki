---
title: 🚀 iptables 使用方法的综合介绍，及路由路径介绍
description: 🚀 iptables 综合介绍，及路由路径介绍
published: 1
date: 2024-12-10T14:10:03.831Z
tags: iptables, network, route, traceroute, linux, windows, tracert
editor: markdown
dateCreated: 2024-12-05T01:14:27.972Z
---

# 🚀 `iptables` 使用方法的综合介绍

[English](/en/iptables-traceroute.md) | [Japanese](/ja/iptables-traceroute.md) | [Chinese](/zh/iptables-traceroute.md)

`iptables` 是 Linux 上一种强大的防火墙工具，用于管理和配置网络流量的规则。通过它可以设置允许、拒绝或转发的流量规则，保护系统网络安全。

---

## 🛠️ 基本概念

- **表（Table）**
  - `filter`：✨ 默认表，用于设置允许或禁止的流量规则。
  - `nat`：🔄 用于网络地址转换（如端口转发）。
  - `mangle`：🛠️ 用于修改数据包内容或标记。
  - `raw`：⏩ 用于设置不被跟踪的流量规则。

- **链（Chain）**
  - `INPUT`：📥 处理进入本地系统的数据包。
  - `OUTPUT`：📤 处理本地系统发送的数据包。
  - `FORWARD`：🔀 处理转发到其他网络的数据包。
  - `PREROUTING`：🚦 路由前处理数据包（用于 `nat`）。
  - `POSTROUTING`：🏁 路由后处理数据包（用于 `nat`）。

- **规则（Rule）**
  每条链包含多条规则，每条规则定义了条件和对应的动作。

---

## 🖥️ 常用命令

- **查看当前规则**：
  ```bash
  iptables -L -v -n
  ```
  - 参数解释：
    - -L：列出规则。
    - -v：显示详细信息。
    - -n：不解析主机名，提高效率。
    - 添加规则：
    ```bash
    #iptables -A <链名> -p <协议> --dport <端口> -j <动作>
    iptables -A INPUT -p tcp --dport 22 -j ACCEPT
    ```
    - 删除规则：
    ```bash
    #iptables -D <链名> <规则编号>
    iptables -L --line-numbers 
    ```
    - 插入规则：
    ```bash
    #iptables -I <链名> <规则编号> -p <协议> --dport <端口> -j <动作>
    #插入允许 80 端口的 HTTP 流量到第一条规则：
    iptables -I INPUT 1 -p tcp --dport 80 -j ACCEPT
    ```
    - 清空规则：
    ```bash
    #清空全部规则：
    iptables -F
    #清空指定链：
    iptables -F <链名>
    ```
    - 保存规则：
    ```bash
    service iptables save
    #或者
    iptables-save > /etc/iptables/rules.v4
    ```
    
    - 恢复规则：
    ```bash
    iptables-restore < /etc/iptables/rules.v4
    ```
 
## 🎯 动作（Target）

  -  ACCEPT：✅ 允许数据包通过。
  -  DROP：❌ 丢弃数据包，不发送任何响应。
  -  REJECT：🚫 拒绝数据包，并向发送方返回错误消息。
  -  LOG：📋 记录日志，通常与其他动作配合使用。
## 📚 实际例子
1. 设置基本防火墙规则
    - 允许本地回环流量：
    ```bash
    iptables -A INPUT -i lo -j ACCEPT
    ```
    - 允许已建立连接：
    ```bash
    iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
    ```
    - 拒绝其他流量：
    ```bash
    iptables -A INPUT -j DROP
    ```
    - 拒绝特定IP的INPUT访问443
    ```bash
    #deny sourceIP 192.168.1.100, destPort:443/tcp, action:DROP
    iptables -A INPUT -s 192.168.1.100 -p tcp --dport 443 -j DROP
    #deny IPRange 
    iptables -A INPUT -s 192.168.1.0/24 -p tcp --dport 443 -j DROP
    #save rules
    service iptables save  # 对于某些系统
    iptables-save > /etc/iptables/rules.v4  # 永久保存规则
    
    #- 使用firewalld 的拒绝例
    firewall-cmd --permanent --add-rich-rule="rule family='ipv4' source address='192.168.1.100' port protocol='tcp' port='443' reject"
    firewall-cmd --permanent --add-rich-rule="rule family='ipv4' source address='192.168.1.0/24' port protocol='tcp' port='443' reject"
    #reload
    firewall-cmd --reload
    ```
    - 多个IP，可以加入ipset， 针对ipset配置规则
    ```bash
    ipset create blocklist hash:ip
    ipset add blocklist 192.168.1.100
    ipset add blocklist 192.168.1.101
    ipset add blocklist 192.168.2.0/24
    iptables -A INPUT -m set --match-set blocklist src -p tcp --dport 443 -j DROP
    
    #save rules 
    ipset save > /etc/ipset.conf
    iptables-save > /etc/iptables/rules.v4
    #重启时加载 ipset 在 /etc/rc.local
    ipset restore < /etc/ipset.conf
    ```
    
2. 端口转发
   - 将流量从 80 端口转发到 8080：
    ```bash
    iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-port 8080
    ```
3. 限制流量
    - 限制每分钟允许 10 个 SSH 连接：
    ```bash
    iptables -A INPUT -p tcp --dport 22 -m limit --limit 10/min -j ACCEPT
    ```
## 📝 注意事项

   - 修改规则不会自动保存，需手动执行保存命令。
   - 大规模部署建议使用 iptables-save 和 iptables-restore 工具。
   - 如果希望简化配置，可选择 ufw 或 firewalld 等高级工具。
   
   
# 🌐 取得发送端到目标的全路由路径的命令介绍

---

## 🛠️ Linux 系统

### 使用 `traceroute` 工具
- **命令格式**：
  ```bash
  traceroute <目标地址或域名>
  ```
- 常见用法：
  - 跟踪到 example.com 的路由：
  ```bash
  traceroute example.com
  ```
- 指定使用 ICMP 数据包：
  ```bash
  traceroute -I example.com
  ```
- 使用 UDP 数据包并指定端口（默认行为）：
  ```bash
  traceroute -p 33434 example.com
  ```
- 限制最大跳数：
  ```bash
  traceroute -m 20 example.com
  ```
  
## 💻 Windows 系统
### 使用 tracert 工具
- **命令格式**：
  ```bash
  tracert <目标地址或域名>
  ```
  - 示例：
  ```bash
  tracert example.com
  ```
  
## ip route 与 iptables 的区别

| 功能方面    | ip route                     | iptables                   |
|---------|------------------------------|----------------------------|
| 主要功能    | 配置系统的路由表，控制数据包的路径选择。         | 配置防火墙规则，控制数据包的访问权限。        |
| 作用层     | 第三层网络层：决定数据包的转发路径。           | 第三层及以上：决定数据包是否允许通过，以及如何处理。 |
| 工作机制    | 指定如何到达某个目标地址。                | 定义数据包允许、拒绝、转发或修改等行为。       |
| 典型命令    | ip route add、ip route del 等。 | iptables -A、iptables -D 等。 |
| 是否支持NAT | 不支持。                         | 支持网络地址转换（NAT）。             |
| 使用场景    | - 配置静态路由。                    | - 设置防火墙。                   |
|         | - 手动优化网络路径。                  | - 实现端口映射。                  |

### 对比示例
1. ip route 设置路由
    ```bash
    # 数据包会通过接口 eth1，经过网关 10.0.0.1，转发到目标网段 192.168.1.0/24。
    ip route add 192.168.1.0/24 via 10.0.0.1 dev eth1
    ```
   

2. iptables 控制访问
    ```bash
    # 允许从源地址 192.168.1.0/24 发往目标地址 10.0.0.0/24 的数据包。 
    iptables -A FORWARD -s 192.168.1.0/24 -d 10.0.0.0/24 -j ACCEPT
    ```
### 使用建议
- **使用 ip route**：
   - 需要手动配置静态路由。
   - 优化网络路径或解决无法路由的情况。

- **使用 iptables**：
   - 实现访问控制（如防火墙规则）。
   - 配置 NAT 或数据包过滤。

- **简单记忆**：
  - ip route 是 路由规则，决定 "走哪条路"。
  - iptables 是 行为规则，决定 "能否通过以及如何处理"。
  
### `ifconfig.me `检查 自身外网IP 地址
```bash
curl ifconfig.me
```
### `tcpdump` 抓包检查流量路径：
```bash
tcpdump -i eth0 host <client_public_ip>
```