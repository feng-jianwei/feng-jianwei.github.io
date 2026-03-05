---
layout: post
title: "openVPN+openssl建立私有网络"
subtitle: ""
date: 2021-07-03
author: "fjw"
header-img: "img/contact-bg.jpg"
tags: [网络,vpn]
---

# server

```shell
mkdir .CA
# 生成 CA 密钥
openssl genpkey -algorithm Ed25519 -out ca.key

# 生成自签名的 CA 证书
openssl req -x509 -key ca.key -days 3650 -out ca.crt -subj "/C=CN/ST=State/L=City/O=fjw/CN=PriVateCA"

# 生成服务器的 Ed25519 密钥和证书
openssl genpkey -algorithm Ed25519 -out server.key
openssl req -new -key server.key -out server.csr -subj "/C=CN/ST=State/L=City/O=fjw/CN=PriVateServercsr"
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt -days 365
```

# client

```shell
openssl genpkey -algorithm Ed25519 -out client.key
openssl req -new -key client.key -out client.csr -subj "/C=CN/ST=State/L=City/O=fjw/CN=MAC"
openssl x509 -req -in client.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out client.crt -days 365
```
