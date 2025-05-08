---
layout: post
title: Linux Bash命令杂记(tr col join paste expand)
date: 2018-03-13
tags: 
- Linux
- Bash
categories: 
- 技术
---

<!-- more -->

**tr命令**

tr命令可以将输入的数据中的某些字符做替换或者是作删除
```shell
tr [-ds] STR
d: 删除输入数据的中的STR
s: 替换重复的字符
# 例
last | tr '[a-z]' 'A-Z'
将last输出的数据中的所有小写字符替换为大写字符
SPPU     UUZ1                          TVF MBS 13 18:45   TUJMM MPHHFE JO   
SFCPPU   TZTUFN CPPU  3.10.0-693.17.1. TVF MBS 13 18:45 - 18:47  (00:01)    
SPPU     UUZ1                          TVF MBS 13 10:55 - 13:15  (02:20)    
SFCPPU   TZTUFN CPPU  3.10.0-693.17.1. TVF MBS 13 10:54 - 18:47  (07:52)    
SPPU     UUZ1                          MPO MBS 12 18:33 - 19:35  (01:02)  
...

cat /etc/passwd | tr -d ':'
将cat /etc/passwd输出的数据中的':'全部删除
# output
rootx00root/root/bin/bash
binx11bin/bin/sbin/nologin
daemonx22daemon/sbin/sbin/nologin
admx34adm/var/adm/sbin/nologin
lpx47lp/var/spool/lpd/sbin/nologin
...
```
**col命令**

```shell
col [-xb]
-x: 将tab键替换为等长的空个
-b: 在文字内由反斜杠时，仅保留反斜杠后接的那个字符

cat -A ~/.bashrc
# 使用cat -A可以讲输出中所有的特殊按键
# output
...
# Source global definitions$
if [ -f /etc/bashrc ]; then$
^I. /etc/bashrc$
fi$
# 注意这里有个^I就是tab字符。

cat -A ~/.bashrc | col -x
# output
# Source global definitions$
if [ -f /etc/bashrc ]; then$
    . /etc/bashrc$
# tab字符不再出现
```
**join命令**

用于对两个文件按照某一个字符或者字段进行按行连接
```shell
join [-ti12] file1 file2
-t: 选择分割字符，并且对比“第一个字段”的数据，如果两个文件相同，则将两条数据连成一行，并将第一个字段放在最前
-i: 忽略大小写
-1: 表示第一个文件
-2: 表示第二个文件

# 例1
head -n 3 /etc/passwd /etc/shadow
# 先查看这两个文件前三行数据
==> /etc/passwd <==
root:x:0:0:root:/root:/bin/bash
bin:x:1:1:bin:/bin:/sbin/nologin
daemon:x:2:2:daemon:/sbin:/sbin/nologin

==> /etc/shadow <==
root:<密码太长，我忽略了方便查看>:17593:0:99999:7:::
bin:*:17110:0:99999:7:::
daemon:*:17110:0:99999:7:::

join -t ':' /etc/passwd /etc/shadow | head -n 3
# output
root:x:0:0:root:/root:/bin/bash:<密码太长，我忽略了方便查看>:17593:0:99999:7:::
bin:x:1:1:bin:/bin:/sbin/nologin:*:17110:0:99999:7:::
daemon:x:2:2:daemon:/sbin:/sbin/nologin:*:17110:0:99999:7:::
#  我们可以看到，按照':'分割，并且默认一第一个字段进行连接

# 例2
我们知道/etc/passwd中第四个字段是GID，而/etc/group中第三个字段是GID，我们就可以像如下进行整合：
join -t ':' -1 4 /etc/passwd -2 3 /etc/group | head -n 3
# output
0:root:x:0:root:/root:/bin/bash:root:x:
1:bin:x:1:bin:/bin:/sbin/nologin:bin:x:
2:daemon:x:2:daemon:/sbin:/sbin/nologin:daemon:x:
# 我们可以看到，将我们需要的字段提到了最前
```
**paste命令**
```shell
直接讲两个文件中的数据按行连接
paste [-d] file1 file2
-d: 设定每行数据连接的字符，默认为tab
paste /etc/passwd /etc/group | head -n 3
# output
root:x:0:0:root:/root:/bin/bash	root:x:0:
bin:x:1:1:bin:/bin:/sbin/nologin	bin:x:1:
daemon:x:2:2:daemon:/sbin:/sbin/nologin	daemon:x:2:
```
**expand命令**
```shell
expand [-t] file
-t: 后面接数字，代表了将一个tab替换为多少个空格键
# 例
cat -A ~/.bashrc
# 使用cat -A可以讲输出中所有的特殊按键
# output
...
# Source global definitions$
if [ -f /etc/bashrc ]; then$
^I. /etc/bashrc$
fi$
注意看有个^I，是tab符号
cat -A ~/.bashrc | expand -t 10 -(标准输入) | cat -A
# output
...
# Source global definitions$
if [ -f /etc/bashrc ]; then$
          . /etc/bashrc$
fi$
我们可以看到原先的tab变为了10个空格
```
