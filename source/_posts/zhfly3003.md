---
title: "JYmusic 2.0 前台XSS漏洞"
id: zhfly3003
---

# JYmusic 2.0 前台XSS漏洞

## 一、漏洞简介

## 二、漏洞影响

## 三、复现过程

利用条件

```
1.登录会员 `2.认证音乐人` 
```

上传音乐时，抓包，修改name或者cover_url参数

值为：

```
XSS"><script>alert(document.cookie)</script>> 
```

此时提交的音乐就会存储到数据库中，由于name和cover_url没有过滤，导致在后台管理员审核的时候会触发一个XSS