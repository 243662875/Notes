**优化清单：**

基本操作

添加普通用户

系统配置

内核配置

# **一、企业面试题**

1.1 如何选择linux系统发行版本？

1.2 32位系统和64位系统的区别？

1.3 ssh连接不上如何排查？

1.4 linux的7种运行级别，及其应用？

1.5 linux从系统开机到登陆之前的启动流程。

# **二、基础环境**

你的系统是什么版本的？

```bash
[root@oldboyedu ~]# cat /etc/redhat-release  #或者centos-release 或os-release
CentOS release 6.7 (Final) ç这是系统版本信息
[root@oldboyedu ~]# uname -r
2.6.32-573.el6.x86_64     ç这是内核kernel的版本号
[root@oldboyedu ~]# uname -m
x86_64 这表示为64位系统
```

# **三、添加普通用户账号**

linux/unix是一个多用户、多任务的操作系统。

超级管理员(root)： root默认在unix/linux操作系统中拥有最高的管理权限。

普通用户：管理员或者具备管理权限的用户创建的。权限：系统管理仅可以读、看，不能增、删、改。

权限越大，责任越大。

可使用如下命令添加一个普通用户账号，并为其设置口令：

```bash
[root@oldboyedu ~]# useradd oldboy
[root@oldboyedu ~]# passwd oldboy **###****问你新的密码，然后你输入  交互设置密码**
Changing password for user oldboy.
New password:
BAD PASSWORD: it is too simplistic/systematic **#****ç提示密码太简单了，但可以不理会。**
BAD PASSWORD: is too simple
Retype new password:
passwd: all authentication tokens updated successfully.
```

提示：

- 一般情况下，在企业生产环境中应尽量避免直接到root用户下操作，除非有超越普通用户权限的系统维护需求，使用完后立刻退回到普通用户。
- 非交互式设置密码：还可通过下面的命令一步到位地设置密码（其中，oldboy为用户名，密码为123456）。

```bash
echo "123456"|passwd --stdin oldboy && history -c
```

尝试切换用户角色，命令如下：

```bash
[root@oldboyedu ~]# su - oldboy <==由root管理员，切换到普通用户oldboy

[oldboy@oldboyedu ~]$ whoami <==查看当前用户是什么

oldboy

[oldboy@oldboyedu ~]$ su - root <==切回到root用户

Password:
```

说明：

1）超级用户root切换到普通用户下面，无需输入对应用户密码，这相当于“皇帝”去“大臣”家里。

2）普通用户切换到root或其他普通用户下，需要输入切换的对应用户密码。

3）普通用户的权限比较小，只能进行基本的系统信息查看等操作，无法更改系统配置和管理服务。

4）$符号是普通用户的命令行提示符，#符号是超级管理员的提示符。示例如下：

```bash
[oldboy@oldboyedu ~]$ #ç普通用户oldboy对应的提示符
[root@oldboyedu ~]# #ç超级管理员root对应的提示符
```

5）提示符@前面的字符代表当前用户（可用whoami查询），后面的为主机名（可用hostname查询），~所在的位置是窗口当前用户所在的路径。示例如下：

```bash
[oldboy@oldboyedu ~]$  #çoldboy为当前用户,oldboyedu为主机名，~表示当前目录，即家目录。
```

6）Linux命令提示符由PS1环境变量控制。示例如下：

```bash
[root@oldboyedu ~]# echo $PS1
[\u@\h \W]$
```

这里的PS1='[\u@\h \W]$ '，可以通过全局变量配置/etc/profile文件调整PS1='[\u@\h \W\t]$ '。

注意：PS1必须大写的。

PS1---变量的名字------藏经阁里面的武功秘籍（葵花宝典）  秘籍名字（书名）

$PS1---查看变量里面的内容---手端着书（葵花宝典）  看书的内容（读书）

PS1=新的内容 ---向变量里面放入东西----修改书的内容（升级书）

欲练此功，必先自宫，若不自宫，也能成功。

linux变量名字（书名）大写的一般是自己用（linux环境变量），在哪里都可以用的变量。

# **四、使用阿里云镜像做yum源**

默认国外的yum源(软件仓库)比较慢，所以换成国内的。

```bash
[root@oldboyedu ~]# cd /etc/yum.repos.d/
[root@oldboyedu yum.repos.d]# ls
CentOS-Base.repo CentOS-Debuginfo.repo CentOS-fasttrack.repo CentOS-Media.repo CentOS-Vault.repo
[root@oldboyedu yum.repos.d]#cp CentOS-Base.repo CentOS-Base.repo.ori #改配置文件要备份
[root@oldboyedu yum.repos.d]# wget -O CentOS-Base.repo [http://mirrors.aliyun.com/repo/Centos-6.repo](http://mirrors.aliyun.com/repo/Centos-6.repo)
#安装必要的软件包
#补充yum命令
#yum -y install tree nmap sysstat lrzsz dos2unix telnet
yum install -y tree lrzsz
```

# **五、关闭SELinux功能**

SELinux（Security-Enhanced  Linux）是美国国家安全局（NSA）对于强制访问控制的实现，这个功能让系统管理员又爱又恨，这里我们还是把它给关闭了吧，至于安全问题，后面通过其他手段来解决，这也是大多数生产环境的做法，如果非要开启也是可以的。关闭方式如下。

**第一个脚印-永久关闭selinux**

第一个里程碑-操作前备份

```bash
[root@oldboyedu ~]# cp /etc/selinux/config /etc/selinux/config.bak
[root@oldboyedu ~]#
[root@oldboyedu ~]#
[root@oldboyedu ~]#
```

第二个里程碑-sed修改，看看结果，不加-i

**第三个里程碑-确认并使用 sed -i 修改文件内容**

```bash
[root@oldboyedu ~]# sed -i 's#SELINUX=enforcing#SELINUX=disabled#g' /etc/selinux/config
```

**第四个里程碑-检查结果**

```bash
[root@oldboyedu  ~]# grep "disabled" /etc/selinux/config
# disabled - No SELinux policy is loaded.
SELINUX=disabled
```

**第二个脚印-临时关闭selinux** 

临时关闭SElinux，可在命令行执行如下命令： ###****命令行修改，重启失效

```bash
[root@oldboyedu ~]# setenforce
usage: setenforce [ Enforcing | Permissive | 1 | 0 ]
```

ç数字0表示Permissive，即给出警告提示，但不会阻止操作，相当于disabled。

数字1表示Enforcing，即表示SElinux为开启状态。

```bash
[root@oldboyedu ~]# setenforce 0  ##<==临时将selinux调成Permissive状态。
[root@oldboyedu ~]# getenforce    ##<==查看selinux当前状态
Permissive
```

命令说明：

- setenforce：用于命令行管理SELinux的级别，后面的数字表示设置对应的级别。
- getenforce：查看SELinux当前的级别状态。

提示：修改配置SElinux后，要想使其生效，必须要重启系统。因此，可配合使用setenforce 0这个临时使其关闭的命令，这样在重启前后都可以使得SElinux关闭生效了，也就是说无须立刻重启服务器了，在生产场景下Linux机器是不能随意重启的(不要给自己找任何理由重启)。

总结selinux关闭：

1. 第一个里程碑-操作前备份cat,cp,sed -i.bak
2. 第二个里程碑-永久的修改

a) cat/vim 先看一眼-找目标

b) sed 's#SELINUX=enforcing#SELINUX=disabled#g'  /etc/selinux/config

c) 执行成功了 再加上-i参数

d) 再查看，cat /grep

1. 第三个里程碑-临时生效

a) setenforce

# **六、设定运行级别为3（文本模式）**

设定运行级别（runlevel）为3，即表示使用文本命令行模式管理Linux，如果按照前面要求完成的系统安装，则无需设置，检查一下即可。检查命令如下：

```bash
[root@web01 ~]# cat /etc/inittab

# For information on how to write upstart event handlers, or how
# upstart works, see init(5), init(8), and initctl(8).
**#linux不同状态，用不同数字表示。**
# Default runlevel. The runlevels used are:
# 0 - halt (Do NOT set initdefault to this)
# 1 - Single user mode    #单用户模式
# 2 - Multiuser, without NFS (The same as 3, if you do not have networking) #多用户模式，没有NFS。
# 3 - Full multiuser mode #完全的多用户模式，命令行模式
# 4 - unused                   #没有使用
# 5 - X11                   #图形界面模式，桌面模式
# 6 - reboot (Do NOT set initdefault to this) #重启
#
id:3:initdefault:

[root@oldboyedu ~]# runlevel #<==查看当前系统运行级别
N 3
```

**切换运行级别：**

```bash
init 6 ##重启电脑
init 0 ##关机
[root@oldboyedu ~]# grep 3:initdefault /etc/inittab
id:3:initdefault: <==这里的3就是Linux默认的运行级别，如果有需求可以将其修改为其他级别。工作中常用3级别，即文本模式。
[root@oldboyedu ~]# init 5 <==切换运行级别为5，只有在安装了桌面程序时才可以执行startx命令切换。
```

**命令说明：**

```bash
q runlevel：查看当前系统运行级别。
q init：切换运行级别，后面接对应级别的数字，例如:init 6就是重启linux服务器了。
```

**关机命令**

```bash
shutdown -h now
halt
```

**重启**

```bash
shutdown -r now
reboot
```

# **七、关闭iptables防火墙**

```bash
关闭防火墙的目的是为了让初学者学习更方便，将来在学了iptables****技术后可再统一开启。
在企业环境中，一般只有配置外网****IP的linux服务器才需要开启防火墙，但即使是有外网IP，对于高并发高流量的业务服务器仍是不能开的，因为会有较大性能损失，导致网站访问很慢，这种情况下只能在前端加更好的硬件防火墙了。
关闭防火墙的具体操作过程如下：
临时关闭
[root@oldboyedu ~]# /etc/init.d/iptables stopiptables: Setting chains to policy ACCEPT: filter [ OK ]iptables: Flushing firewall rules: [ OK ]iptables: Unloading modules: [ OK ]

[root@oldboyedu ~]# /etc/init.d/iptables stop   #<==重复执行下确认已关闭。[root@oldboyedu ~]# /etc/init.d/iptables status ###查看是否关闭iptables: Firewall is not running.

[root@oldboyedu ~]# chkconfig iptables off     #<==关闭开机自启动命令，前面已经关闭这里就无需执行。[root@oldboyedu ~]# chkconfig --list|grep iptiptables 0:off 1:off 2:off 3:off 4:off 5:off 6:off
彻底的让一个服务，不再运行
```

1. `关闭当前正在运行的进程（服务）=====>/etc/init.d/iptables stop`

2. `让他在开机不启动======>chkconfig iptables off`

   ```bash
   [root@oldboyedu37 ~]# chkconfig |grep ipt
   iptables  0:off 1:off 2:on 3:on 4:on 5:on 6:off
   [root@oldboyedu37 ~]# ##我在执行了chkconfig iptables off 之后 ，对当前正在运行iptables 会有什么影响
   [root@oldboyedu37 ~]#
   [root@oldboyedu37 ~]# /etc/init.d/iptables stop
   iptables: Setting chains to policy ACCEPT: filter [ OK ]
   iptables: Flushing firewall rules:                       [ OK ]
   iptables: Unloading modules: [ OK ]
   [root@oldboyedu37 ~]# /etc/init.d/iptables stop
   [root@oldboyedu37 ~]# ###chkconfig 管理的是 开机的时候 某个软件是否启动 （开机自启动）
   ```

# **八、Linux中文显示设置（如何防止显示中文乱码）**

```bash
此项优化为可选项，即调整Linux系统的字符集设置，那么，什么是字符集呢？
简单的说，**<u>字符集就是一套文字符号及其编码</u>**。目前Linux下常用的字符集有：
```

- `GBK：定长，双字节，不是国际标准，支持的系统不少，实际企业用的不多。`

- `UTF-8：非定长，1~4字节，广泛支持，MYSQL也使用UTF-8，企业广泛使用。`

  `可通过快捷的命令方式在/etc/sysconfig/i18n中添加如下内容，使其支持中文显示：`

- ```bash
  [root@oldboyedu ~]# echo $LANG               ###查看系统当前的字符集
  en_US.UTF-8
  [root@oldboyedu ~]# cat /etc/sysconfig/i18n   #####系统字符集配置文件的位置
  LANG="en_US.UTF-8"
  SYSFONT="latarcyrheb-sun16"
  [root@oldboyedu ~]# cp /etc/sysconfig/i18n /etc/sysconfig/i18n.ori  ####备份
  [root@oldboyedu ~]# echo 'LANG="zh_CN.UTF-8"' >/etc/sysconfig/i18n ####修改配置文件
  ```

<!-- →相当于用vi /etc/sysconfig/i18n 添加LANG="zh_CN.UTF-8"内容 -->

```bash
[root@oldboyedu ~]# source /etc/sysconfig/i18n #→使上文修改生效 ###让配置文件生效
[root@oldboyedu ~]# echo $LANG ###****查看系统当前的字符集
zh_CN.UTF-8
提示：
q 乱码的核心解决方法：
```

1. `系统字符集(utf-8)`
2. `xshell软件的字符集保持一致(utf-8)`
3. `文件使用的字符集一致`

```bash
zh_CN.GBK
q 注意“zh_CN.UTF-8”的大小写字母。
q 这个中文显示配置要跟你自己的SSH客户端的配置一致。
q 调整SSH客户端CRT的字符集，使其和Linux服务端一致（UTF-8）。
查看字符集设置的结果。
[root@oldboyedu ~]# cat /etc/sysconfig/i18n

LANG="zh_CN.UTF-8"
登录Linux系统查看中文字符正常显示的步骤如下：
1）将服务器端字符集（/etc/sysconfig/i18n）改为LANG="zh_CN.UTF-8"。
2）将客户端字符集（CRT）调整为UTF-8。
3）命令行执行setup命令看到原来中文乱码的窗口不乱了，正确显示中文字符了（如图所示）。
```

# **九、linux特殊变量===调整历史记录以及终端超时**

**9.1 命令行临时生效：**

```bash
export TMOUT=300      #连接超时时间控制变量
export TMOUT=3      #3秒之后，没有任何输入，则自动退出。timed out waiting for input: auto-logout
export HISTSIZE=5 #history size命令行历史记录数量 history 命令显示的条数
export HISTFILESIZE=5  #命令行命令对应的历史记录文件，文件中命令记录数 ~/.bash_history
```

**9.2 永久生效:** 

```bash
放入到/etc/profile
source 生效
了解如何配置即可，我们学习环境可以不配置。
270 shutdown -h 10
271 tree
272 history
273 history -d 270
274 history
[root@oldboyedu-30 ~]# !271
tree
.
├── anaconda-ks.cfg
├── a.txt
├── ett.txt
├── install.log
├── install.log.syslog
└── test.txt
0 directories, 6 files
```

# **十、隐藏Linux版本信息显示**

```bash
<v:shape id="图片_x0020_7" o:spid="_x0000_i1025" type="#_x0000_t75" style="width:294.6pt;
height:53.4pt;visibility:visible;mso-wrap-style:square"><v:imagedata src="file:///C:/Users/lee/AppData/Local/Temp/msohtmlclip1/01/clip_image009.png" o:title=""></v:imagedata></v:shape>
在登录到Linux主机本地（非CRT连接的窗口）前，会显示系统的版本和内核，如图所示。
登录后执行如下命令上产上述登录Linux前的终端内容显示的实际存放文件。
[root@oldboyedu ~]# cat /etc/issue
CentOS release 6.7 (Final)
Kernel \r on an \m
[root@oldboyedu ~]# cat /etc/issue.net
CentOS release 6.7 (Final)
Kernel \r on an \m
执行如下命令清除Linux系统版本及内核信息
[root@oldboyedu ~]# > /etc/issue
[root@oldboyedu ~]# > /etc/issue.net
[root@oldboyedu ~]# cat /etc/issue
[root@oldboyedu ~]# cat /etc/issue.net
```