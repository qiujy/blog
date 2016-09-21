---
title: curl 模拟ajax请求
date: 2016-09-20 16:36:23
tags: curl
---

使用 `curl` 可以模拟各种http请求:

``` bash
curl -i \
-H "Content-Type: application/json" \
--data '{username:"test010",password:"123456"}' \
http://192.168.1.2:q/login
```
