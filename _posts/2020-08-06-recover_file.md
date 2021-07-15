---
layout: post
title: Linux上恢复误删除的文件或目录
date: 2020-08-06
tags: 系统知识
---

Linux不像windows有那么显眼的回收站，不是简单的还原就可以了。 linux删除文件还原可以分为两种情况，一种是删除以后在进程存在删除信息，一种是删除以后进程都找不到，只有借助于工具还原，这里分别检查介绍下。

# 一、误删除文件进程还在的情况
这种一般是有活动的进程存在持续标准输入或输出，到时文件被删除后，进程PID还是存在。这也就是有些服务器删除一些文件但是磁盘不释放的原因。比如当前举例说明： 通过一个shell终端对一个测试文件做cat追加操作：

```
[root@huyouba1 ~]# echo  "hello  py" > testdelete.py

[root@huyouba1 ~]# cat  >> testdelete.py 
hello delete
```
另外一个终端查看这个文件可以清楚看到内容：

```
[root@huyouba1 ~]# cat testdelete.py 
hello  py
hello delete
```
此时，在当前服务器删除文件**rm -f ./testdelete.py**

命令查看这个目录，文件已经不存在了，那么现在我们将其恢复出来。

### 1.lsof查看删除的文件进程是否还存在
这里用到一个命令lsof，如没有安装请自行yum或者apt-get。类似这种情况，我们可以先lsof查看删除的文件 是否还在：

```
[root@huyouba1 ~]# lsof | grep deleted

mysqld     1512   mysql    5u      REG              252,3          0    6312397 /tmp/ibzW3Lot (deleted)
cat       20464    root    1w      REG              252,3         23    1310722 /root/testdelete.py (deleted)
```
幸运的是这种情况进程还存在 ，那么开始进行恢复 操作。

### 2.恢复
> 恢复命令：

```
cp /proc/pid/fd/1  /指定目录/文件名
```
> 进入 进程目录，一般是进入/proc/pid/fd/,针对当前情况：


```
[root@huyouba1 ~]# cd   /proc/20464/fd

[root@huyouba1 fd]# ll
total 0
lrwx------ 1 root root 64 Nov 15 18:12 0 > /dev/pts/1
l-wx------ 1 root root 64 Nov 15 18:12 1 > /root/testdelete.py (deleted)
lrwx------ 1 root root 64 Nov 15 18:12 2 > /dev/pts/1
```
> 恢复操作：

```
cp 1 /tmp/testdelete.py
```
> 查看文件：

```
[root@huyouba1 fd]# cat  /tmp/testdelete.py
hello  py
hello delete
```
> 恢复完成

# 二、误删除的文件进程已经不存在，借助于工具还原
> 创建准备删除的目录并echo一个 带有内容的文件：

```
[root@huyouba1 huyouba1]# tree
.
├── deletetest
│   └── mail
│       └── test.py
├── lost+found
└── passwd
3 directories, 2 files

[root@huyouba1 huyouba1]# cat /21yunwei/deletetest/mail/test.py 
hello Dj

[root@huyouba1 huyouba1]# tail  -2  passwd 
haproxy:x:500:502::/home/haproxy:/bin/bash
tcpdump:x:72:72::/:/sbin/nologin
```

> 执行删除操作：

```
[root@huyouba1 huyouba1]# rm  -rf    ./*

[root@huyouba1 huyouba1]# ll
total 0
```

> 现在开始进行误删除文件的恢复。这种情况一般是没有守护进程或者后台进程对其持续输入，所以删除就删除了，lsof也看不到。就要借助于工具。这里我们采用的工具是extundelete第三方工具。恢复步骤如下：

### 1.停止对当前分区做任何操作，防止inode被覆盖。inode被覆盖基本就告别恢复了。比如停止所在分区的服务，卸载目录所在的设备，有必要的情况下都可以断网。

### 2.通过dd命令对当前分区进行备份，防止第三方软件恢复失败导致数据丢失。适合数据非常重要的情况，这里测试，就没有备份，如备份可以考虑如下方式：

```
dd if=/path/filename of=/dev/vdc1
```

### 3.通过umount命令，对当前设备分区卸载。或者fuser 命令

```
umount /dev/vdb1 或者 umount /huyouba1
```

> 如果提示设备busy，可以用fuser命令强制卸载：

```
fuser -m -v -i -k /huyouba1
```

### 4.下载第三方工具extundelete安装，搜索误删除的文件进行还原。

```
wget  http://nchc.dl.sourceforge.net/project/extundelete/extundelete/0.2.4/extundelete-0.2.4.tar.bz2

tar jxvf extundelete-0.2.4.tar.bz2

cd  extundelete-0.2.4

./configure 

make

make  install
```

> 扫描误删除的文件：

```
[root@huyouba1 extundelete-0.2.4]# extundelete  --inode 2 /dev/vdb1
NOTICE: Extended attributes are not restored.
Loading filesystem metadata ... 8 groups loaded.
Group: 0
Contents of inode 2:
.
.省略N行
File name                                       | Inode number | Deleted status
.                                                 2
..                                                2
lost+found                                        11             Deleted
deletetest                                        12             Deleted
passwd                                            14             Deleted
```
通过扫描发现了我们删除的文件夹，现在执行恢复操作。

### 5.恢复单一文件passwd

```
[root@huyouba1 /]# extundelete /dev/vdb1 --restore-file passwd   
NOTICE: Extended attributes are not restored.
Loading filesystem metadata ... 8 groups loaded.
Loading journal descriptors ... 46 descriptors loaded.
Successfully restored file passwd
```
> 恢复文件是放到了当前目录RECOVERED_FILES。 查看恢复的文件：

```
[root@huyouba1 /]# tail  -5  RECOVERED_FILES/passwd 
mysql:x:497:500::/home/mysql:/bin/false
nginx:x:496:501::/home/nginx:/sbin/nologin
zabbix:x:495:497:Zabbix Monitoring System:/var/lib/zabbix:/sbin/nologin
haproxy:x:500:502::/home/haproxy:/bin/bash
tcpdump:x:72:72::/:/sbin/nologin
```

### 6.恢复目录deletetest

```
[root@huyouba1 /]# extundelete /dev/vdb1 --restore-directory  deletetest 
NOTICE: Extended attributes are not restored.
Loading filesystem metadata ... 8 groups loaded.
Loading journal descriptors ... 46 descriptors loaded.
Searching for recoverable inodes in directory deletetest ... 
5 recoverable inodes found.
Looking through the directory structure for deleted files ... 

[root@huyouba1 /]# cat  RECOVERED_FILES/deletetest/mail/test.py 
hello Dj
```
### 7.恢复所有

```
[root@huyouba1 /]# extundelete /dev/vdb1 --restore-all
NOTICE: Extended attributes are not restored.
Loading filesystem metadata ... 8 groups loaded.
Loading journal descriptors ... 46 descriptors loaded.
Searching for recoverable inodes in directory / ... 
5 recoverable inodes found.
Looking through the directory structure for deleted files ... 
0 recoverable inodes still lost. 

[root@huyouba1 /]# cd RECOVERED_FILES/

[root@huyouba1 RECOVERED_FILES]# tree
.
├── deletetest
│   └── mail
│       └── test.py
└── passwd
2 directories, 2 files
```

### 8.恢复指定inode

```
[root@huyouba1 /]# extundelete /dev/vdb1 --restore-inode 14
NOTICE: Extended attributes are not restored.
Loading filesystem metadata ... 8 groups loaded.
Loading journal descriptors ... 46 descriptors loaded.

[root@huyouba1 /]# tail  -5   /RECOVERED_FILES/file.14 
mysql:x:497:500::/home/mysql:/bin/false
nginx:x:496:501::/home/nginx:/sbin/nologin
zabbix:x:495:497:Zabbix Monitoring System:/var/lib/zabbix:/sbin/nologin
haproxy:x:500:502::/home/haproxy:/bin/bash
tcpdump:x:72:72::/:/sbin/nologin
```
> 注意恢复inode的时候，恢复 出来的文件名和之前不一样，需要单独进行改名。内容是没问题的。
> 
> 更多的extundelete用法请参考extundelete –help选项参数说明，当前恢复所有的操作完成。
