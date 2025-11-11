---
title: bash下jq对json操作
description: jq操作json语法简介
published: true
date: 2025-11-11T05:29:25.627Z
tags: json, jq
editor: markdown
dateCreated: 2025-11-11T05:29:25.627Z
---

# 🧩 jq 命令速查表

> jq 是命令行下处理 JSON 的强大工具。  
> 语法类似 JavaScript，支持字段访问、筛选、遍历、重组等。

---

## 🟢 基本结构

```bash
jq <过滤表达式> <file.json>
cat file.json | jq <过滤表达式>
```

选项：

| 参数 | 说明 |
|------|------|
| `-r` | 输出原始值（去掉引号） |
| `-c` | 输出单行紧凑 JSON |
| `-M` | 禁用彩色输出 |
| `-s` | 合并多个 JSON 输入为数组 |

---

## 🟢 1. 读取字段

假设：
```json
{
  "name": "Alice",
  "age": 25,
  "skills": ["bash", "python", "terraform"]
}
```

| 操作 | 命令 | 输出 |
|------|------|------|
| 整个 JSON | `jq '.' data.json` | 格式化 JSON |
| name | `jq '.name' data.json` | `"Alice"` |
| 原始 name | `jq -r '.name' data.json` | `Alice` |
| 第 1 个技能 | `jq '.skills[0]' data.json` | `"bash"` |
| 数组长度 | `jq '.skills | length' data.json` | `3` |

---

## 🟢 2. 组合输出

```bash
jq '.name, .age' data.json
# 输出两行: "Alice" 和 25

jq '{Name: .name, Skill1: .skills[0]}'
# 输出: {"Name":"Alice","Skill1":"bash"}
```

---

## 🟢 3. 字符串插值

```bash
jq -r '"Name=\(.name), Age=\(.age)"' data.json
# 输出: Name=Alice, Age=25
```

---

## 🟢 4. 遍历数组

假设：
```json
{
  "users": [
    {"name": "Alice", "age": 25},
    {"name": "Bob", "age": 30}
  ]
}
```

| 目标 | 命令 | 输出 |
|------|------|------|
| 遍历数组 | `jq '.users[]' data.json` | 每个对象 |
| 取每个名字 | `jq -r '.users[].name' data.json` | Alice<br>Bob |
| 筛选年龄 > 25 | `jq '.users[] | select(.age > 25)'` | {"name":"Bob","age":30} |
| 只输出名字 | `jq -r '.users[] | select(.age > 25) | .name'` | Bob |

---

## 🟢 5. 管道与过滤器

`|` 表示前一个结果传入后一个过滤器：

```bash
jq '.users[] | .name' data.json
# 等价于 for user in users: print(user.name)
```

---

## 🟢 6. 生成/修改 JSON

追加元素：
```bash
jq '.users += [{"name":"Charlie","age":28}]' data.json
```

提取并生成新结构：
```bash
jq '{people: [.users[].name]}' data.json
# 输出: {"people":["Alice","Bob"]}
```

---

## 🟢 7. 多文件合并

```bash
jq -s '[.[][]]' file1.json file2.json
# 把两个文件的数组合并成一个大数组
```

---

## 🟢 8. 实用示例

| 任务 | 命令 | 说明 |
|------|------|------|
| 获取键名 | `jq -r 'keys[]' data.json` | 列出顶层键 |
| 删除字段 | `jq 'del(.age)'` | 删除 age 字段 |
| 判断字段是否存在 | `jq 'has("age")'` | true / false |
| 格式化压缩 | `jq -c '.' data.json` | 单行 JSON |
| 美化打印 | `jq '.' data.json` | 彩色缩进输出 |

---

## 🟢 9. 在 Shell 脚本中使用

```bash
#!/bin/bash
json=$(cat data.json)
name=$(echo "$json" | jq -r '.name')
age=$(echo "$json" | jq -r '.age')
echo "Name: $name, Age: $age"
```

---

## 🟢 10. jq 在线测试工具

- [https://jqplay.org/](https://jqplay.org/)
- [https://jqterm.com/](https://jqterm.com/)

---

🧠 **提示**：jq 支持完整的条件判断、循环、函数定义等高级功能，可用于自动化配置解析、API 响应处理等场景。
