---
title: 🚀Nginx を使用してログイン認証を追加
description: Nginx を使用して smtp4dev のウェブページにログイン認証を追加する
published: true
date: 2025-10-31T05:06:12.957Z
tags: authentication, login, nginx, smtp4dev
editor: markdown
dateCreated: 2025-09-25T02:53:30.572Z
---

# 🚀 Nginx を使用してログイン認証を追加する

[English](/nginx-authentication.md) | [Japanese](/ja/nginx-authentication.md) | [Chinese](/zh/nginx-authentication.md)

以下は、Docker と Nginx を使用して smtp4dev のウェブページにユーザー認証を設定する手順です。

---
## 1️⃣ `Nginx.conf` に `auth_basic` と `auth_basic_user_file` を追加する
```conf
http {
    server {
        listen 80;

        location / {
            proxy_pass http://smtp4dev:8025; # smtp4dev をプロキシ
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;

            auth_basic "Restricted Access";            #👈️ 基本認証を有効にする 🔒
            auth_basic_user_file /etc/nginx/htpasswd;  #👈️ 認証ファイルを指定
        }
    }
}
```
## 2️⃣  `htpasswd` ファイルを作成する
- htpasswd ツールを使用してユーザー名とパスワードを生成します：
```bash
# htpasswd ツールをインストール
#apt-get install apache2-utils
apk add apache2-utils

# htpasswd ファイルを作成し、ユーザーを追加
htpasswd -c ./htpasswd mailuser
# パスワードを入力し、確認
```
- これにより、次のような内容の htpasswd ファイルが生成されます：
```plaintext
mailuser:$apr1$xyz12345$abcdEFGhijkLmnopQrstUV
```
- `htpasswd` ファイルを `/etc/nginx/` に置き、nginx を再読み込みします：
```bash
nginx -s reload
```
- メールページを再度開くと、ユーザー名とパスワードの入力を求められます。
