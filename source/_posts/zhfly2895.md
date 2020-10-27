---
title: "（CVE-2019-6339）Drupal 远程代码执行漏洞"
id: zhfly2895
---

# （CVE-2019-6339）Drupal 远程代码执行漏洞

## 一、漏洞简介

Drupal core是Drupal社区所维护的一套用PHP语言开发的免费、开源的内容管理系统。 Drupal core 7.62之前的7.x版本、8.6.6之前的8.6.x版本和8.5.9之前的8.5.x版本中的内置phar stream wrapper（PHP）存在远程代码执行漏洞。远程攻击者可利用该漏洞执行任意的php代码。

## 二、漏洞影响

Drupal core 7.62之前的7.x版本、8.6.6之前的8.6.x版本和8.5.9之前的8.5.x版本

## 三、复现过程

### 漏洞环境

执行如下命令启动drupal 8.5.0的环境：

```
docker-compose up -d 
```

环境启动后，访问 `http://your-ip:8080/` 将会看到drupal的安装页面，一路默认配置下一步安装。因为没有mysql环境，所以安装的时候可以选择sqlite数据库。

### 漏洞复现

如下图所示，先使用管理员用户上传头像，头像图片为构造好的 PoC，参考[ianxtianxt/PoC](https://github.com/ianxtianxt/PoC)的PoC

![image](../img/543b8fcbefe5b4e1a34623ae8266120b.png)

Drupal 的图片默认存储位置为 `/sites/default/files/pictures//`，默认存储名称为其原来的名称，所以之后在利用漏洞时，可以知道上传后的图片的具体位置。

访问 `http://127.0.0.1:8080/admin/config/media/file-system`，在 `Temporary directory` 处输入之前上传的图片路径，示例为 `phar://./sites/default/files/pictures/2019-06/blog-ZDI-CAN-7232-cat_0.jpg`，保存后将触发该漏洞。如下图所示，触发成功。

![image](../img/eb418c35697a29106ecce08fb9a3a10c.png)

## 参考链接

> https://vulhub.org/#/environments/drupal/CVE-2019-6339/