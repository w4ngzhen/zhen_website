---
layout: post
title: Linux文件系统与inode、Block笔记
date: 2018-04-08
tags: 
- Linux
- inode
- block
categories: 
- 技术
---

在Linux下一切都是文件，无论是设备还是接口，亦或是网卡等均被抽象成了文件，并且有相关的内核代码进行调度。然而，在一切都是文件的前提下，最需要进行探讨的则是文件存储的根源：文件系统。文件系统的好坏能够更加完美的解决在一个操作系统中对于文件的管理。

<!-- more -->

Linux下的文件系统是按照inode + block模式来进行了的。通俗一点讲，类似于指针一样的形式存在。即inode作为“指针”记录并指向了真正的”数据块“block。 

**环境**

为了方便后面的实际操作，我们按照如下进行环境的搭建
```shell
# 首先在test目录下创建一个名为temp的文件夹与一个普通的文件test.txt
$ pwd
/root/test
$ mkdir temp && touch test.txt
# 文件结构如下
test
├── temp
└── test.txt
1 directory, 1 file
# 向test.txt写入一句话
$ echo "Its's a test.txt" >> test.txt
```

**inode**

在Linux中，每一份文件都对应了独一无二的inode编号，通过使用命令ls加参数-i，则可以在显示的文件前显示inode编号。
```shell
$ ls -i
33933113 temp 17414066 test.txt
# 前面的数字就是inode且独一无二
```
除此之外，inode中还存储更加关键的文件元信息：权限、属性等。例如当我们使用ls -l显示文件的相关属性时，这里面的信息就存储在inode：
```shell
$ ls -l
total 4
drwxr-xr-x. 2 root root  6 Apr  8 13:05 temp
-rw-r--r--. 1 root root 16 Apr  8 13:08 test.txt
# 文件夹、文件名前面的诸如读、写、执行权限，修改时间等均存储在inode中
```
当然，对于inode来说，既然它本身能够记录这些信息，所以自身是有大小的，每个inode 大小均固定为128 bytes，虽然不大，但是记录元信息完全足够了。同时，它本身还记录此文件数据所在的block数据块的编号。inode记录一个block编号需要花掉4bytes

**block**

既然inode是记录文件的元信息的，那么一般文件本身的数据记录在何处呢？答案则是block数据块。在Linux中，block文件快大小通常选择4KB，当然我们还可以选择1KB、2KB等。这是不定的，但是如果选择的太小，那么inode需要记录block编号就要增多，如果太大，容易造成存储碎片。如何理解？其实我们可以把block看作是文件的基本单位，例如，我们现在有一个22KB大小的文件，一个简单的计算，倘若我们使用4KB的block需要6块才能完全装下，不过会有2KB是浪费了的。如果我们采用2KB的block则刚好11块可以装下。这里虽然是选择2KB是最合适的，但是在一般Linux系统中是4KB，为什么？请自行查阅相关的资料，不再赘述。

**综合inode与block进行探讨**

上面的inode与block只是大致介绍了相关的，下面才是更加实在的内容。

inode具体包含了哪些东西？
```
1、文件的字节数
2、文件拥有者的User ID
3、文件的Group ID
4、文件的读、写、执行权限
5、文件的时间戳，共有三个：ctime指inode上一次变动的时间，mtime指文件内容上一次变动的时间，atime指文件上一次打开的时间。
6、链接数，即有多少文件名指向这个inode
7、文件数据block的位置
```
1-5点不难理解，第6点在后面的软硬连接再叙，第七点这里要提一下。上面说过inode本身128 Bytes，还是能记录很多信息的，这里1-6点不至于花光128 Bytes，而对于第7点，inode本身能够记录12个block，如果采用4KB block显然，我们只能存储12 * 4KB大小的文件，这显然是不现实的。为了解决这个问题，inode中在第12个记录block编号之后，还能动态的增加二级、甚至三级间接指向，这里我们使用如下的图更为形象的说明：
![inode-block](https://static-res.zhen.wang/images/post/2018-04-08-inode-block/inode-block.png)
在上图的情况下，我们可以知道假设我们使用4KB大小的block，并且刚好使用满二次间接，能够存储的数据大小为：
```
12*4KB + 1024*4KB + 1024*1024*4KB
= 48KB + 4MB + 4GB
≈ 4GB
```
**关于文件夹的inode、block**

上面讨论inode与block我们都是以一个普通文件的角度来看待的。然而，文件夹inode与block与普通文件是有一定的差别的。对于一个文件夹来说，inode与普通文件类似，包含了关于文件夹的属性、读写执行权限、时间戳等。然而，文件夹inode中的直接block通常不会超过12个直接的。为什么呢？因为文件夹所指向的block只会存储这个文件夹拥有的文件的inode编号，并不会存储实际的文件内容。

例如，当我们拥有一个文件夹dir，这个文件夹下面只有一份文件大小为4GB的inode为1234的文件，那么实际上，文件夹inode中存储的block中只会存储类似inode=1234这样的信息。下图能够更加形象的展示：
![dir-inode-block](https://static-res.zhen.wang/images/post/2018-04-08-inode-block/dir-inode-block.png)
总结一下，文件夹的block只会存储对应文件夹下面的文件的inode。所以当我们访问某一个文件的时候，譬如我问需要查看/root/test.txt的时候，流程如下：
```
首先检查根目录下的权限“/”，符合权限
通过之后，检查根目录的block是否存储有“root/”目录以及对应inode

查找成功，找到/root/目录的inode，检查权限等信息，符合权限
通过之后，检查/root/inode下的block中是否存在test.txt以及inode

查找成功，找到/root/test.txt的inode检查权限等信息，符合
查询inode中的直接或间接block将数据读出
```
那么，通过以上的分析，我们也很容易的能够理解，如果我们没有对文件夹有写的权限，是不能够删除文件夹下面的文件或者是创建文件的。因为当我们删除一个文件的时候，是对**文件夹**inode对应的block中存储的文件信息进行删除或添加。由于root用户的特殊性，我们使用一个普通用户zhen，并在zhen用户的home目录下创建一个temp文件夹
```shell
$ cd ~
$ ls -l
drwxrwxr-x. 2 zhen zhen  6 Apr  8 13:05 temp
# temp文件夹对于zhen用户有读写以及执行的权限，所以我们（zhen）可以自由的在里面添加删除文件
$ cd ./temp
$ touch test
$ ls
test
$ rm test
$ ls
# 创建删除都没有问题
```
接下来我们首先在temp文件夹中创建一个test2文件，再回到上一目录，将temp的写权限移除
```shell
# 创建test2文件
$ touch test2
$ ls
test2
# 回到上一目录
$ cd ..
# 将temp对于zhen的写权限移除
$ chmod 500 ./temp
$ ls -l
total 4
dr-x------. 2 zhen zhen  6 Apr  8 14:52 temp
```
然后，我们再次进入temp文件夹，试图创建文件，发现失败：
```shell
$ cd ./temp
$ touch test3
touch: cannot touch ‘test3’: Permission denied
$ rm test2
rm: cannot rm ‘test3’: Permission denied
```
操作发现，无论是在该文件夹下创建还是删除文件，都是失败的。那么，我们还能不能编辑该文件夹该文件夹下面的文件呢？譬如想test2文件插入一句“Hello”？理论上来讲，是可以的，因为我们修改test2文件内容，并不会影响temp文件夹的block内容，而事实上也是如此：
```shell
$ echo "Hello" > test2
$ cat test2
Hello
# 没有问题
```
