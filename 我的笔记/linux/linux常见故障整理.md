[TOC]

# 一. 文件和目录类

## **1.1 File exist 文件已经存在**

```bash
[root@oldboyedu59 ~]# mkdir   /data   /lidao  
[root@oldboyedu59 ~]# mkdir   /data   /lidao  
mkdir: cannot create directory ‘/data’: File exists
mkdir: cannot create directory ‘/lidao’: File exists
```

> mkdir: cannot create directory ‘/lidao’: File exists
> 无法   创建    目录   因为这个目录已经存在

## **1.2 No such file or directory 没有这个文件或目录（这个东西不存在）**

> 没有这个目录:文件或路径书写错误

```bash
[root@oldboyedu59 ~]# mkdir  /oldboy
[root@oldboyedu59 ~]# cd oldboy
-bash: cd: oldboy: No such file or directory
```

> mkdir命令本身问题:mkdir 命令默认只能创建1层目录 创建多层报错
> -p解决

```bash
[root@oldboyedu59 ~]# mkdir  /data/oldboy/lidao/
mkdir: cannot create directory ‘/data/oldboy/lidao/’: No such file or directory
```

> touch命令只能创建文件,目录不存在则会报错
> 解决:先创建目录,再创建文件

```bash
[root@oldboyedu59 ~]# ls /oldboy/
oldboy.txt
[root@oldboyedu59 ~]# touch /oldboy/lidao/alex/oldboy.txt
touch: cannot touch ‘/oldboy/lidao/alex/oldboy.txt’: No such file or directory
```

**排错思路：**

1、ls命令检查对应的目录是否存在?

2、目录不存在 先创建目录在创建文件/

## **1.3 command not found 命令找不到（没有这个命令）**

```bash
[root@oldboyedu59 ~]# mkdiy
-bash: mkdiy: command not found
```

> 1.书写错误
> 2.没有安装

## **1.4 invalid option   无效的参数（不可用的参数）**

```bash
[root@oldboyedu59 ~]# touch -p /oldboy/oldboy.txt
touch: invalid option -- 'p'
Try 'touch --help' for more information.
```

## **1.5 overwrite  覆盖**

> cp复制如果已经存在这个文件会提示是否覆盖

```bash
[root@oldboyedu59 ~]# cp  /oldboy/oldboy.txt   /tmp/
cp: overwrite ‘/tmp/oldboy.txt’? 
```

## **1.6 remove regular empty file 是否删除普通文件（空的）**

```bash
[root@oldboyedu59 ~]# rm   /oldboy/oldboy.txt
rm: remove regular empty file ‘/oldboy/oldboy.txt’?
```

## **1.7 is a directory xxx是一个目录**

> rm默认无法删除目录
> 解决:加上-r 或-rf

```bash
[root@oldboyedu59 ~]# rm /data/
rm: cannot remove ‘/data/’: Is a directory
```

> vi命令中 使用vi编辑目录也会报错

```bash
"/oldboy"
E502: "/oldboy" is a directory
Press ENTER or type command to continue
```

## **1.8 descend into directory 是否进入目录**

```bash
[root@oldboyedu59 ~]# rm -r /data/
rm: descend into directory ‘/data/’? y
rm: remove regular empty file ‘/data/oldboy01.txt’? n
rm: remove regular empty file ‘/data/oldboy02.txt’? n
rm: remove regular empty file ‘/data/oldboy03.txt’? n
rm: remove regular empty file ‘/data/oldboy04.txt’? n
rm: remove regular empty file ‘/data/oldboy05.txt’? n
rm: remove regular empty file ‘/data/oldboy06.txt’? n
rm: remove regular empty file ‘/data/oldboy07.txt’? n
rm: remove regular empty file ‘/data/oldboy08.txt’? n
rm: remove regular empty file ‘/data/oldboy09.txt’? n
rm: remove regular empty file ‘/data/oldboy10.txt’? n
rm: remove directory ‘/data/’? n
1.9 Invalid level 无效的层数,层数必须大于0
```

> 注意参数位置

```bash
[root@oldboyedu59 ~]# tree  -L -F 2 / 
tree: Invalid level, must be greater than 0.
```

## **1.10 Can't open file for writing 无法打开这个文件**

> vi中 如果目录不存在就会提示

```bash
"/oldbyo/oldboy.txt"
"/oldbyo/oldboy.txt" E212: Can't open file for writing
Press ENTER or type command to continue
```

## **1.11 No write since last change**

```bash
E37: No write since last change (add ! to override)
     粘包赖(你修改了内容就无法使用:q退出 需要使用:q! 
```

## **1.12 xx column window is too narrow  窗口只有xx列太窄了 无法完全显示**

> 这是w的坑 空间太小施展不开.

```bash
[root@oldboyedu60-lnb ~]# w
w: 39 column window is too narrow
```

# **二. 网络连接类**

## **2.1 远程连接错误 Connection Failed 连接失败**

> 使用Xshell远程连接失败提示,检查端口是否开启或正确

```bash
[c:\~]$ 

Connecting to 10.0.0.200:233...
Could not connect to '10.0.0.200' (port 233): Connection failed.

Type `help' to learn how to use Xshell prompt.
```

> 使用telnet测试端口是否打开

```bash
[c:\~]$ telnet 10.0.0.200 233 

Connecting to 10.0.0.200:233...
Could not connect to '10.0.0.200' (port 233): Connection failed.   #233端口没有开启

Type `help' to learn how to use Xshell prompt.
```

> 端口开启

```bash
[c:\~]$ telnet 10.0.0.200 22
Connecting to 10.0.0.200:22...
Connection established.            #端口开启
To escape to local shell, press 'Ctrl+Alt+]'.
SSH-2.0-OpenSSH_7.4

Protocol mismatch.

Connection closed by foreign host.

Disconnected from remote host(10.0.0.200:22) at 12:22:54.

Type `help' to learn how to use Xshell prompt.
[c:\~]$ 
```

## **2.2 yum安装软件故障提示 Could not resolve host无法解析主机**

> Could not resolve host无法解析主机
> 主要是系统能否上网和DNS问题.

```bash
http://mirrors.tuna.tsinghua.edu.cn/centos/7.6.1810/updates/x86_64/repodata/repomd.xml: [Errno 14] curl#6 - 
"Could not resolve host: mirrors.tuna.tsinghua.edu.cn; Unknown error"
Trying other mirror. 
```

## **2.3 yum安装软件提示:Nothing to do (没事做)**

> 有两种情况:
> 情况1:软件已经安装并且最新如下:

```bash
Package tree-1.6.0-10.el7.x86_64 already installed and latest version
tree软件包已经安装并且是最新版本
Package 2:vim-enhanced-7.4.160-5.el7.x86_64 already installed and latest version
Package 1:bash-completion-2.1-6.el7.noarch already installed and latest version
Nothing to do
```

> 情况2:软件名字写错或没有配置yum源导致找不到这个软件包

```bash
[root@oldboyedu60-lnb ~]# yum install treea -y
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
 * base: mirror.lzu.edu.cn
 * extras: mirrors.nwsuaf.edu.cn
 * updates: mirrors.nwsuaf.edu.cn
base                                                                                              | 3.6 kB  00:00:00     
extras                                                                                            | 3.4 kB  00:00:00     
updates                                                                                           | 3.4 kB  00:00:00     
No package treea available. 
#没有找到叫treea的软件包
Error: Nothing to do
```

## **2.4 Name or service not known 域名无法识别（无法上网）**

**原因：**

1：DNS配置错误原因

2：Linux无法上网原因

```bash
[root@oldboyedu59 ~]# ping baidu.com 
ping: baidu.com: Name or service not known
  域名无法识别（无法将域名---->ip地址)
```

# **三. 修改系统基础配置类**

## **3.1 重启网卡报错 device not present**

```bash
[root@oldboyusd ~]# systemctl restart network 
Job for network.service failed because the control process exited with error code.
 See "systemctl status network.service" and "journalctl -xe" for details.
```

> 查看详细错误原因
> ·journalctl -xe·

```bash
Apr 01 15:31:05 oldboyusd.1 network[7816]: Bringing up interface etho:  ERROR     : [/etc/sysconfig/network-scripts/ifup-eth] Device  does not seem to be present, delaying initialization.

Apr 01 15:31:05 oldboyusd.1 /etc/sysconfig/network-scripts/ifup-eth[8019]: Device  does not seem to be present, delaying initializatio
```

## **3.2 修改主机名过程中,命令行中主机名没有变化**

### 1 hostname命令修改主机名（临时 重启服务器之后失效）

```bash
[root@oldboyedu59 ~]# hostname
oldboyedu59
[root@oldboyedu59 ~]# hostname oldboyedu59-lnb 
```

### 2 修改文件内容（写合同 永久 重启服务器之后生效）

```bash
vim /etc/hostname 
 oldboyedu59-lnb
```

### 3 检查

```bash
[root@oldboyedu59 ~]# hostname
oldboyedu59-lnb
[root@oldboyedu59 ~]# cat /etc/hostname 
oldboyedu59-lnb
```

### 命令行中的主机名部分没有改变？

> 解决：重新登录下即可（断开连接，重新连接）

```bash
[root@oldboyedu59-lnb ~]# 
```