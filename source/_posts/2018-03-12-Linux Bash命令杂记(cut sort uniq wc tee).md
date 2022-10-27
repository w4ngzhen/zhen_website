---

layout: post
title: Linux Bash命令杂记(cut sort uniq wc tee)
date: 2018-03-12
tags: 
- Linux
- Bash

---

<!-- more -->

**数据流重定向**

```
标准输入（stdin）：代码为0，使用<或<<；
标准输出（stdout）：代码为1，使用>或>>；
标准错误输出（stderr）：代码为2，使用2>或2>>；
>：覆盖的方式，>>：追加的方式
```
如果想要一般输出与错误输出同时输入到某一个文件，如果采取如下的方式进行输出是错误的：
```shell
输出数据  1> list 2> list
```
如果按照上面的方式输出到list文件中时而没有采用特殊的语法，会因为两个输出进程的同步问题，导致正确的数据与错误的数据可能会交叉的输入到list文件中。正确的方式应该如下：
```shell
输出数据 > list 2>&1
# 或者是
输出数据 &> list
```
**命令执行&& ||**
```
cmd1 && cmd2
若cmd1执行完毕且正确执行($?==0)，则执行cmd2
若cmd1执行完毕且错误执行($?!=0)，则不执行cmd2

cmd1 || cmd2
若cmd1执行完毕且执行正确($?==0)，则不执行cmd2
若cmd1执行完毕且执行错误($?!=0)，则执行cmd2
```
**cut命令**

cut命令按行数据进行处理，常用的方式如下：
```shell
#参数 -d -f（组合使用）
输出数据 | cut -d '分个字符' -f fields
# 例
str=ni:hao:ma:?
echo $str | cut -d ':' -f 2
表示将echo出的str字符串按照':'字符分割，且取第2个字段
得到的结果是
hao
# 补充
-f 1,3 代表取第1和第3字段，输出 ni:man
-f 1-3 取1到3字段，输出 ni:hao:ma

# 参数 -c
输出数据 | cut -c 字符范围
# 例
str=hello
echo $str | cut -c 1
输出
h
# 补充
-c 1-，输出 hello
-c 1-3，输出 hel
```
**sort命令**
```shell
head -4 /etc/passswd
# output
root:x:0:0:root:/root:/bin/bash
bin:x:1:1:bin:/bin:/sbin/nologin
daemon:x:2:2:daemon:/sbin:/sbin/nologin
adm:x:3:4:adm:/var/adm:/sbin/nologin
我们可以看到并没有按照首字母排序

head -4 /etc/passwd | sort
# output
adm:x:3:4:adm:/var/adm:/sbin/nologin
bin:x:1:1:bin:/bin:/sbin/nologin
daemon:x:2:2:daemon:/sbin:/sbin/nologin
root:x:0:0:root:/root:/bin/bash
我们可以看到已经按照首字母排序了

同样，我们可以指定想按照哪一个字段来排序，
head /etc/passwd | sort -t ':' -k 3
# 不看前4行了，准备输出所有行
# 将输出按照类型':'分割(-t ':')，并且取第3个字段(-k 3)
# 然而此时的字段依然是按照字符进行，如本测试机上输出的结果注意看第二行：
root:x:0:0:root:/root:/bin/bash
operator:x:11:0:operator:/root:/sbin/nologin
bin:x:1:1:bin:/bin:/sbin/nologin
daemon:x:2:2:daemon:/sbin:/sbin/nologin
# 注意第三个字段，11跑到了2前面去了，因为字符串11排在2前面
此时我们需要加上 -n 参数提示按照数字进行
head /etc/passwd | sort -t ':' -k 3 -n
```
**last命令**
```shell
# 该命令用来列出目前与过去登录系统的用户相关信息
last
# output
root     tty1                          Mon Mar 12 18:33   still logged in   
reboot   system boot  3.10.0-693.17.1. Mon Mar 12 18:33 - 19:02  (00:29)    
root     tty1                          Sat Mar 10 20:18 - 20:18  (00:00)    
reboot   system boot  3.10.0-693.17.1. Sat Mar 10 20:18 - 20:18  (00:00)    
root     tty1                          Fri Mar  9 19:10 - 20:50  (01:40)    
...
其中：
第一列：用户名
第二列：终端位置。(pts/0通过ssh或者telnet远程连接的用户，tty：直接连接到计算机或者本地用户)
第三列：登陆IP或者内核（看到0.0或者什么都没有，意味着用户通过本地终端连接，除了重启，内核版本会显示在状态中）

第四列：开始时间(如：sun apr 3 ：四月三号星期天)
第五列：结束时间（still login in 还未退出，down：直到正常关机，crash：直到强制关机）
第六列:持续时间
```
**uniq命令**

```shell
last | cut -d ' ' -f 1 | sort | uniq
# 先取用户名，然后排序，最后去重
# output
reboot
root
wtmp
zhen
# 加上 -c 显示统计
last | cut -d ' ' -f 1 | sort | uniq -c
# output
1
27 reboot
26 root
1 wtmp
3 zhen
```
务必注意，uniq命令是通过叠加去重**相邻**的字符串，如果你不首先进行排序，那么会出现下面的情况：
```shell
      1 root
      1 reboot
      1 root
      1 reboot
      1 root
      1 reboot
      1 root
      1 reboot
      1 zhen
      1 root
      1 reboot
...
```
**wc命令**
```shell
wc [-lwm]
-l: 仅列出行
-w: 仅列出多少个英文单词
-m: 仅列出多少个字符
head /etc/passwd | wc
# output
  10  10  385
# 分别代表行数，词数，字符数（这里10个“词”应该是因为每一行没有空格的原因，wc统计是按空格来分词的）
```
**tee双向重定向**

由前面的数据流我们可以知道，我们在将数据定向时，如果不采取特殊的操作，数据要么输出到屏幕，要么输出到文件或者是设备中，没有办法，既输出到屏幕有输出到文件中；又或者是，我们想要对数据进行处理存放到一个文件中，但是同时对原始数据又存到另一个文件中。使用tee命令，我们就可以做到。

例如，我们使用last命令首先要把数据存放到last.log中，同时要对用户去重并输出到屏幕上：
```shell
last | tee [-a 追加] last.log | cut -d ' ' -f 1 | sort | uniq
# output

reboot
root
wtmp
zhen
# 同时我们打开last.log文件可以看到没有做任何处理的原始数据
root     tty1                          Mon Mar 12 18:33   still logged in   
reboot   system boot  3.10.0-693.17.1. Mon Mar 12 18:33 - 19:29  (00:56)    
root     tty1                          Sat Mar 10 20:18 - 20:18  (00:00)    
reboot   system boot  3.10.0-693.17.1. Sat Mar 10 20:18 - 20:18  (00:00)    
root     tty1                          Fri Mar  9 19:10 - 20:50  (01:40)    
...
```
