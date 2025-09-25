---
title: 🚀 Adding Login Authentication to the Web Page Using Nginx
description: Adding Login Authentication to the smtp4dev Web Page Using Nginx
published: 1
date: 2024-12-09T07:42:25.573Z
tags: authentication, login, nginx, smtp4dev, auth_basic, auth_basic_user_file
editor: markdown
dateCreated: 2024-12-09T07:41:55.725Z
---

# 🚀 Adding Login Authentication to Web Page Using Nginx

[English](/nginx-authentication.md) | [Japanese](/ja/nginx-authentication.md) | [Chinese](/zh/nginx-authentication.md)

The following are the steps to set up user authentication for the smtp4dev web page using Docker and Nginx.
---
## 1️⃣ Add `auth_basic` and `auth_basic_user_file` to `Nginx.conf`
```conf
http {
    server {
        listen 80;

        location / {
            proxy_pass http://smtp4dev:8025; # Proxy to smtp4dev web page
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;

            auth_basic "Restricted Access";            #👈️ Enable basic authentication 🔒
            auth_basic_user_file /etc/nginx/htpasswd;  #👈️ Specify the authentication file
        }
    }
}
```
## 2️⃣ Create the `htpasswd` File
- Use the `htpasswd` tool to generate a username and password:
```bash
# Install the htpasswd tool
#apt-get install apache2-utils
apk add apache2-utils

# Create the .htpasswd file and add a user
htpasswd -c ./htpasswd mailuser
# Enter and confirm the password
```
- This will generate an `htpasswd` file with content similar to:
```plaintext
mailuser:$apr1$xyz12345$abcdEFGhijkLmnopQrstUV
```
- Place the `htpasswd` file under `/etc/nginx/`, and reload nginx:
```bash
nginx -s reload
```
- When you reopen the mail page, you will be prompted to enter the username and password.
