---
title: "（CVE-2018-14574）Django < 2.0.8 任意URL跳转漏洞"
id: zhfly2885
---

# （CVE-2018-14574）Django < 2.0.8 任意URL跳转漏洞

## 一、漏洞简介

Django默认配置下，如果匹配上的URL路由中最后一位是/，而用户访问的时候没加/，Django默认会跳转到带/的请求中。（由配置项中的`django.middleware.common.CommonMiddleware`、`APPEND_SLASH`来决定）。

在path开头为`//example.com`的情况下，Django没做处理，导致浏览器认为目的地址是绝对路径，最终造成任意URL跳转漏洞。

该漏洞利用条件是目标`URLCONF`中存在能匹配上`//example.com`的规则。

## 二、漏洞影响

Django < 2.0.8

## 三、复现过程

访问`http://your-ip:8000//www.example.com`，即可返回是301跳转到`//www.example.com/`：

![image](../img/8d562ad3f362ab7ebcb09971d15e17e2.png)