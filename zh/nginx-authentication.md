---
title:  🚀 使用 Nginx 为 Web 页面添加登录认证
description:  🚀 使用 Nginx 为没有login的 Web 页面，比如smtp4dev  添加登录认证
published: 1
date: 2024-12-09T07:19:06.393Z
tags: nginx, login, authentication, smtp4dev
editor: markdown
dateCreated: 2024-12-09T07:19:01.448Z
---

# 🚀 使用 Nginx 为 `smtp4dev` Web 页面添加登录认证

[English](/nginx-authentication.md) | [Japanese](/ja/nginx-authentication.md) | [Chinese](/zh/nginx-authentication.md)

以下是通过 Docker 和 Nginx 为 `smtp4dev` 的 Web 页面设置用户认证的步骤。

---
## 1️⃣ 向`Nginx.conf`追加`auth_basic`和`auth_basic_user_file`
```conf
http {
    server {
        listen 80;

        location / {
            proxy_pass http://smtp4dev:8025; # 代理 smtp4dev Web 页面
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;

            auth_basic "Restricted Access";            #👈️ 开启基本身份验证 🔒
            auth_basic_user_file /etc/nginx/htpasswd;  #👈️ 指定认证文件
        }
    }
}
```
## 2️⃣ 创建 `htpasswd` 文件
- 使用 htpasswd 工具生成用户名和密码：
```bash
# 安装 htpasswd 工具
#apt-get install apache2-utils
apk add apache2-utils

# 创建 .htpasswd 文件并添加用户
htpasswd -c ./htpasswd mailuser
# 输入密码并确认
```
- 这会生成一个 `htpasswd` 文件，内容类似于：
```plaintext
mailuser:$apr1$xyz12345$abcdEFGhijkLmnopQrstUV
```
- 将`htpasswd` 文件放在`/etc/nginx/`下, 重新load nginx 
```bash
nginx -s reload
```
- 重新打开mail页面，会提示输入user/password
