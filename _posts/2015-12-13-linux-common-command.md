---
layout: post
title: Linux常用命令
date:       2015-12-13
author:     xue
catalog:    true
tags:
    - linux
---

# Linux常用命令总结


&emsp;&emsp;虽然使用Linux较长时间了，但是还没有对Linux系统的学习过，导致遇到一些问题常常要去网上查Linux命令，因此花了近一周的时间看了《鸟哥的Linux私房菜》前十二章，这里记录一些常用了Linux命令，一方面是为了加强记忆，一方面也是为了方便以后的查询。



>\# 立即关机  
>$ shutdown -h now  
>$ init 0

>\# 延迟10S后关机  
>$ shutdown -h +10

>\# 立即重启  
>$ shutdown -r now

>\# 向其他用户发警告消息，并不是真正的要关机  
>$ shutdown -k now 'This system will reboot'

>\# 从给定的包含绝对路径的文件名中去除文件名（非目录的部分），然后返回剩下的路径（目录的部分）    
>$ dirname 绝对路径

>\# 一次创建多个文件夹  
>$ mkdir -p test1/test2/test3

>\# 由第一行开始显示文件内容  
>$ cat filenmae    

>\# 显示文件内容并显示行号  
>$ cat -n filename

>\# 以覆盖的方式将file1的内容写入file2  
>$ cat file1>file2

>\# 以追加的方式将file1的内容写入file2  
>$ cat file1>>file2

>\# 从最后一行开始显示  
>$ tac filename

>\# 查看多页文件  
>$ more filename  
>\# Space代表向下翻一页  
>\# Enter代表向下滚动一行 

>\# 向下搜索String    
>$ /String

>\# 向上搜索String    
>$ ?String

>\# 取出文件最后n行内容  
>$ tail -n filename

>\# 获取默认创建文件的user,group,other group权限  
>$ umask  
>\# 如：输出为0022，后面三位数字的意思是，0：user具有所有权限，2:group成员被拿掉了w权限，2：other group被拿掉了w权限

>\# 文件的隐藏属性chattr    
>$ chattr +i filename  
>\# 使得文件“不能被删除，改名，设置连接也无法写入或添加数据，只有root能设置此权限。

>\# 取消文件的i属性  
>$ chattr -i filename 

>\# 显示文件的隐藏属性  
>$ lsattr [-adR] 文件或目录  
>\# a:将隐藏文件的属性也秀出来;d:如果连接的是目录，仅列出目录本身的属性而非目录内的文件名；R:连同子目录的数据也一并列出来

>\# 查看文件的类型    
>$ file filename

>\# 列出查询到的目录（只列出第一个）    
>$ which filenmae

>\# 列出又有查询到的目录    
>$ which -a filename
>\# which 是寻找“执行文件”

>\# 寻找特定文件    
>$ whereis filename

>\# 根据文件的部分名称查找  
>$ locate [part of filename]

>\# find命令  
>$ find pathname -options [-print -exec -ok ...]

>\# 以人们较容易阅读的GB、MB、KB等格式查看分区空间信息  
>$ df -h

>\# 评估文件系统的磁盘使用量(常用语评估目录)  
>$ du directory

>\# 查看磁盘使用情况  
>$ fdisk -l

>\# 磁盘格式化    
>$ mkfs [options] device  
>\# 示例：mkfs -t ext3 /dev

>\# 挂载设备  
>$ mount [-t vfstype] [-o options] device dir  
>\# 示例：mount -t iso9660 device dir

>\# 变量的显示与设置  
>$ echo

>\# 取消变量  
>$ unset 变量名

>\# 变量赋值  
>$ name=xue(等号左右不能有空格)

>\# 变量累加  
>\# 如：PATH=$PATH:/home/dmtsai/bin

>\# 历史命令  
>$ history [n]  
>\# n:数字，是要列出最近的n条命令

>\# 改变文件或目录时间  
>$ touch fileA  
>\# 如果fileA存在，使用touch指令可更改这个文件或目录的日期时间，包括存取时间和更改时间；  
>\# 如果fileA不存在，touch指令会在当前目录下新建一个空白文件fileA。

>\# 选取命令  
>$ grep  
>\# 示例：ps -aux | grep java

## Vi编辑器常用命令

>\# 向下移动一页  
>$ ctrl+f

>\# 向上一动一页    
>$ ctrl+b

>\# 向下移动半夜 
>$ ctrl+d

>\# 向上移动半页  
>$ ctrl+u

>\# 移动到这一行的最前面字符处  
>$ “0” 或者 [Home]

>\# 移动到这一行的最后面字符处  
>$ “$” 或者 [End]

>\# 移动到文件的最后一行  
>$ G

>\# 移动到文件的第一行  
>$ gg

>\# 光标向下移动n行  
>$ n[Enter]

>\# 向下搜索String  
>$ /String

>\# 向上搜索String  
>$ ?String  

>\# 向前删一个字符  
>$ x 

>\# 向后删一个字符  
>$ X

>\# 删除一行  
>$ dd

>\# 删n行  
>$ ndd

>\# 复制光标所在的那一行  
>$ yy

>\# 复制光标所在的向下n行  
>$ nyy

>\# 将复制的数据粘贴在光标的下一行  
>$ p

>\# 将复制的数据粘贴在光标的上一行    
>$ P

>\# 复制块数据  
>$ v，然后通过左右箭头选择列;ctrl+v,然后通过上下箭头选择行；y（复制刚才选择的数据）

>\# 一个vim编辑器打开多个文件  
>$ vim file1 file2  
>$ n(切换到下一个文件)

>\#　打开多个vim编辑器窗口　　  
>$ 在第一个file的编辑器窗口中     ：sp file2（：也要输入）  
>\# 切换到上方的编辑器：ctrl+w+↑  
>\# 切换到下方的编辑器：ctrl+w+↓



## 通配符与特殊符号

- *：代表0个到无穷多个任意字符  
- ?：代表一定有一个任意字符  
- []：同样代表一定有一个在中括号内的字符（非任意字符）。例如[abcd]代表一定有一个字符，可能是a,b,c,d这四个任何一个  
- [-]：若有减号在中括号内时，代表在编码顺序内的所有字符。例如[0-9]代表0到9之间的数字，因为数字的语系编码是连续的  
- [^]：若中括号中的第一个字符为指数符号(^)，那表示原向选择，例如[^abc]代表一定有一个字符，只要是非a,b,c的其他字符就接受的意思

示例一：找出/etc/下面以cron为开头的文件名  
$ ll -d /etc/cron*

示例二：找出/etc/下面文件名刚好是五个字母的文件名  
$ ll -d /etc/?????

示例三：找出/etc/下面文件名含有数字的文件名  
$ ll -d /etc/\*[0-9]\*

示例四：找出/etc/下面文件名开头不为小写字母的文件名  
$ ll -d /etc/[^a-z]*

示例五：将示例四兆道德文件复制到/tmp中  
$ cp -a /etc/[^a-z]* /tmp
