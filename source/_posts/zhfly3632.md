---
title: "Powershell基础命令"
id: zhfly3632
---

#powershell递归寻址注册表

```
$key = Get-Item HKLM:\Software\Microsoft\PowerShell\1
$key.Name
HKEY_LOCAL_MACHINE\Software\Microsoft\PowerShell\1 `$key | Format-List ps*` 
```