---
title: ' 使用Let’s encrypt部署HTTPS站点'
date: "2019-11-08T00:01:07+08:00"
tags:
- network
draft: false
---

### 1. 使用`certbort`生成有效期三个月的证书

```bash
certbot certonly --webroot -w /usr/share/nginx/html
```
会要求用户输入域名。

`--webroot -w`参数的意义在于让Let’s Encrypt可以从外部根据输入的域名，试图获取`-w`中由`certbot`生成并存放的临时文件，从而确认该域名确实在用户的控制之下。

详细参考：[User Guide — Certbot 0.34.0.dev0 documentation](https://certbot.eff.org/docs/using.html#webroot)

### 2. 部署certificates到Nginx

执行`certbort`之后，在`/etc/letsencrypt/live`下会生成certificates。
配置nginx添加如下内容启用：
```
listen 443 ssl default_server;
listen [::]:443 ssl default_server;
ssl_certificate /etc/letsencrypt/live/your-domain-name/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/your-domain-name/privkey.pem;
```
注意这里使用`fullchain.pem`作为证书文件，而不是`cert.pem`。否则会出现客户端无法获取证书的错误。

### 3. 重启nginx

```bash
nginx -s reload
```
