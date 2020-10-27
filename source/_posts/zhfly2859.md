---
title: "（CVE-2018-7662）Couchcms 2.0 存在路径泄露漏洞"
id: zhfly2859
---

# （CVE-2018-7662）Couchcms 2.0 存在路径泄露漏洞

## 一、漏洞简介

Couch through 2.0存在路径泄露漏洞，当访问特定url时系统返回的报错信息中暴露物理路径信息。Couch through是一个在github上开源的系统，漏洞发现者已经将漏洞信息通过[issues](https://github.com/CouchCMS/CouchCMS/issues/46)告知作者。

## 二、漏洞影响

Couch through 2.0

## 三、复现过程

### poc

访问如下页面，报错信息中显示完整物理路径信息。

```
http://www.0-sec.org/includes/mysql2i/mysql2i.func.php
http://www.0-sec.org/addons/phpmailer/phpmailer.php 
```