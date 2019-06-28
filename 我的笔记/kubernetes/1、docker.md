[TOC]

# 第一章：docker基础

参考：https://mp.weixin.qq.com/s/bcafrpR21PAr71mBJb6iFw

**docker的优点**

1、资源利用率比传统虚拟机高

2、支持跨节点部署

3、版本可控，组件可服用

4、共享镜像

5、轻量级，易维护

**docker的缺点**

1、宿主机资源没有完全做到隔离

2、语言不成熟



## 1、安装docker

docker官网建议使用Ubuntu操作系统作为宿主机，应该看重了Ubuntu默认支持AUFS文件系统的缘故。linux内核最小版本是3.10，必须是64位操作系统，

``` shell
#centos上安装docker
yum install -y yum-utils device-mapper-persistent-data lvm2		#安装依赖包
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo	#添加阿里yum源
yum makecache fast	#更新yum缓存
yum -y install docker-ce	#安装docker
systemctl start docker		#启动docker
docker run -itd -p 80:80 --name webserver nginx  /bin/bash   #启动一个容器,先从本地仓库查找镜像,如果没有再从官网上查找

# 第二种方式，使用docker的离线安装docker，yum可以帮助我们安装依赖
yum localinstall -y docker-ce-17.09.1.ce-1.el7.centos.x86_64.rpm

#Ubuntu上安装docker
wget -qo- https://get.Docker.com/ | sh		#使用wget获取docker安装包，更新docker也是一样

```

[Windows上安装docker参考1：菜鸟教程](https://www.runoob.com/docker/windows-docker-install.html)

**卸载docker**

``` bash
#centos上卸载docker
yum remove docker-ce		#卸载docker
rm -rf /var/lib/docker		#删除docker保留的数据

#Ubuntu上卸载docker
apt-get purge lxc-Dcoker	#卸载docker
apt-get autoremove --purge lxc-docker	#卸载docker安装包和依赖的模块
rm -rf /var/lib/docker		#删除docker所保留的数据
```

**配置docker**

通常情况下docker默认的参数就可以了，如果想对docker定制化安装，就需要配置docker。

1、创建docker组

默认情况下docker会监听本地的socket文件，这个文件是root用户创建的，其他用户没有读写权限，我们创建一个docker组，然后将docker用户加到docker组里，这样可以回避掉socket文件没有读写权限的问题

```bash
groupadd docker
usermod -aG docker ubuntu	#创建用户到组
```

2、调整内存参数

```bash
#在使用docker中，可能出现类似下面的告警，启动系统中内存和swap统计功能后可以解决
warning :your kernel does not support cgroup swap limit
warning :your kernel does not spport swap limit capabilities limitation discarded
#解决方法：
#编辑/etc/default/grub文件
#修改 GRUB_CMDLINE_LIUNX参数如下：
GRUB_CMDLINE_LIUNX="cgroup_enable=memory swapaccount=1"
#然后执行 sudo update-grub 更新grub，最后重启系统。
```

3、调整ufw（Ubuntu系统上的防火墙）参数

如果想从另一台机器访问这台主机上的容器，就需要允许外来的请求，docker daemon默认的服务tcp端口是2375.加密的端口是2376

## 2、使用镜像

[关于docker的基本命令可以参考菜鸟教程](<https://www.runoob.com/docker/docker-command-manual.html>)

``` dockerfile
docker pull ubnutu			#从互联网上下载一个镜像到本地仓库,默认使用latest这个标签
docker push  注册用户名/镜像名		#推送镜像到远程镜像仓库
docker search php				#默认从官网查找php的镜像
docker images					#列出本地镜像
docker inspect nginx:latest          #获取docker镜像的详细信息
docker inspect -f '{{.NetworkSettings.IPAddress}}' mynginx           #获取在运行的mynginx容器IP
docker rmi ubuntu:latest             #删除本地仓库的镜像，如果正在使用可加上 -f 强制删除但不建议使用
Docker rmi $(docker images -a|grep none|awk '{print $3}')  #删除没有打tag的镜像
docker rmi $(docker images | grep gcr|awk '{print $1":"$2}')	#删除某些开通的docker镜像
```

## 3、镜像迁移|导入和导出

```dockerfile
docker save nginx:latest > nginx.tar.gz		#将镜像导出来，就是指保存到本地
docker save -o nginx.03.tar nginx:latest	#或者使用这个命令
#-----------------------------------------------------------------------------
docker load < nginx.tar.gz		#将导出的镜像重新导入镜像库
docker load -i nginx.tar.gz		#或者使用这个命令
```

## 4、docker Hub介绍

官方提供的镜像仓库网址：https://hub.docker.com/

```bash
docker login -u user -p password server_url		#登陆docker Hub
docker logout localhost:8080		#退出
```

直接输入 docker login 也可以登录 默认登陆的是https://hub.docker.com/

## 5、搭建私有镜像仓库

### 5.1、docker开源的镜像分发工具--docker Registry

部署方式参考：<https://www.cnblogs.com/Eivll0m/p/7089675.html>  https://www.cnblogs.com/feinian/p/7857430.html

GitHub网址：https://hub.docker.com/_/registry/

Docker registry 是用于打包，传输，存储和分发的工具

1、安装docker-registry

docker run -d -p 5000:5000 --restart=always --name registry -v /opt/registry:/var/lib/registry registry:2

服务启动之后就可以向她推送和拉取镜像了

### 5.2、harbor部署

 需要基于docker环境，因为我们使用docker来启动harbor镜像库，所以需要先安装docker

创建docker配置文件

```bash
mkdir /etc/docker && vi /etc/docker/daemon.json
```

修改daemon.json文件中的内容

```json
# data-root: docker的数据目录，务必保证目录存在
# hub.paas: harbor镜像库域名
#192.168.191.166: harbor镜像库IP

{
        "log-driver": "journald",
        "data-root": "/home/docker_data_dir",        
        "insecure-registries": [
        "hub.paas",
        "192.168.191.166",                           
        ]
}
overlay2
{
        "storage-driver": "overlay2",
        "storage-opts": "overlay2.override_kernel_check=true",
        "log-driver": "journald",
        "data-root": "/home/docker_data_dir",
        "insecure-registries": [
        "hub.paas",
        "192.168.191.166"
        ]
}

```

```bash
# 启动docker
systemctl daemon-reload
systemctl start docker
```

**1、安装harbor**

```bash
# 解压harbor压缩包并且进入harbor文件夹
tar -zxvf harbor-offline-installer-v1.3.0-rc4.tgz && cd harbor
# 修改harbor.cfg文件如下图
vi harbor.cfg
```

修改harbor的配置文件

```bash
# 将docker-compose（二进制文件）放到宿主机上
cp docker-compose /usr/local/bin/
chmod +x /usr/local/bin/docker-compose
#启动harbor
./install.sh --with-clair
# 安装成功之后，即可通过harbor镜像库的ip地址，通过游览器来访问.初始账号密码为：admin/Harbor12345
```

第二种方法参考：

**1、安装docker-compose**

版本下载地址：https://github.com/docker/compose/releases/

二进制安装：

curl -L https://github.com/docker/compose/releases/download/1.16.1/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose

chmod +x /usr/local/bin/docker-compose

**2、根据自己的情况决定是否安装命令补全功能**

yum install bash-completion

或者

curl -L https://raw.githubusercontent.com/docker/compose/1.16.1/contrib/completion/bash/docker-compose -o /etc/bash_completion.d/docker-compose

检查是否安装成功

docker-compose --version

其他安装方式docker-compose

yum install python-pip

pip install  docker-compose

**3、安装harbor**

在线安装的方式

wget -P /usr/loca/src/     https://github.com/vmware/harbor/releases/download/v1.2.0/harbor-online-installer-v1.2.0.tgz

离线安装的方式

https://github.com/vmware/harbor/releases       #下载地址

tar -zxvf harbor-1.7.5.tar.gz

cd /usr/local/harbor/

修改配置文件

1.harbor.cfg

---

## 6、容器常用命令

### 6.1、操作命令

```bash
docker create debian:jessie		#创建容器但不启动
docker restart start stop  容器ID		#重启 启动 关闭 创建的容器
docker run -it debian:jessie /bin/bash		#运行容器,通过bash进入debian系统,退出容器后会关闭. 
docker run -p 3306:3306 --name mysql -e MYSQL_ROOT_PASSWORD=123456 -d mysql	#启动mysql容器
docker run --restart=always -itd centos:latest /bin/bash	#后台运行容器,加上--restart=always参数随宿主机一同启动，其他可选参数如下
	-a stdin: 指定标准输入输出内容类型，可选 STDIN/STDOUT/STDERR 三项；
	-d: 后台运行容器，并返回容器ID；
	-i: 以交互模式运行容器，通常与 -t 同时使用；
	-t: 为容器重新分配一个伪输入终端，通常与 -i 同时使用；
	-P: (大写)容器的80端口映射到主机的随机端口
	-p: 容器的端口映射到主机的对应端口，例如： -p 80:80 
	-v: 主机的目录映射(挂载)到容器的目录，例如：-v /home/ubuntu/nginx/www:/www
	-h "mars": 指定容器的hostname；
	-e username="ritchie": 设置环境变量；
	-c 或 --cpu-shares：设置容器使用cpu权重；
	-m 或 --memory：设置内存使用限额。例如 -m 200M、--memory 300M；
	--memory-swap：设置内存+swap的使用限额，当-m 200M --memory-swap=300M时，表示容器可以使用200M内存和100Mswap；
	--vm：启动内存工作线程数。例如：--vm 1，启动1个内存工作线程；
	--vm-bytes 280M：每个工作线程分配280M内存；
	--dns 8.8.8.8: 指定容器使用的DNS服务器，默认和宿主一致；
	--dns-search example.com: 指定容器DNS搜索域名，默认和宿主一致；
	--env-file=[]: 从指定文件读入环境变量；
	--cpuset="0-2" or --cpuset="0,1,2": 绑定容器到指定CPU运行；
	--net="bridge": 指定容器的网络连接类型，支持 bridge/host/none/container:<name|id> 四种类型；
	--link=[]: 添加链接到另一个容器；
	--expose=[]: 开放一个端口或一组端口；
	--name: 为容器指定一个名称；
docker exec -it CONTAINER ID(容器的ID) /bin/bash		#进入正在运行中的容器
docker attach		#进入正在运行的容器，例如：docker attach --sig-proxy=false mycon
docker rm 容器id		#删除容器
docker kill		#发送信号给容器默认SIGKILL例如：docker kill -s KILL mycon(-s表示向容器发送一个信号)一般用stop,只有容器停不了的情况下用
docker wai		#阻塞到一个容器，直到容器停止运行。例如：docker wait mycon
docker pause		#暂停容器中所有的进程，当容器不需要继续工作但有不关闭就需要暂停容器。 例如：docker pause mycon
docker unpause		#恢复容器中所有的进程。 例如：docker unpause mycon
docker ps		#查看容器的状态
	-a 列出所有容器 包含未运行的
	-l 列出最新创建的容器 
	-n=2 列出最近创建的2个容器 
	-q 仅列出容器ID 
	-s 显示容器大小
docker logs		#查看容器的日志(stdout/stderr)
	-f	#跟踪日志输出,例如：docker logs -f mycon（查看容器mycon的日志输出）
	--since		#显示某个开始时间的所有日志
	-t		#显示时间戳
	--tail	#仅列出最新N条容器日志，例如：docker logs --since="2017-05-01" --tail=10 mycon(查看容器mycon从2017年5月1日后的最新10条日志。）
docker events	#得到docker服务器的实时的事件
	-f		#根据条件过滤事件；例如：docker events -f "image"="mysql:5.6" --since="1466302400" （显示docker 镜像为mysql:5.6 这个时间戳对应的日期之后的相关事件。）
	--since		#从指定的时间戳后显示所有事件;例如：docker events --since="1466302400" （显示docker 在这个时间戳对应的日期之后的所有事件。）
	--until		#流水时间显示到指定的时间为止；
docker port		#显示容器的端口映射，例如：docker port mycon
docker top		#显示容器的进程信息，支持ps参数。例如docker top mycon
docker diff		#显示容器文件系统的前后变化， 检查容器里文件结构的更改。例如：docker diff mycon
docker cp /www/test mycon:/www/		#将主机的/www/test目录拷贝到容器mycon的/www目录下
docker cp mycon:/www /tmp/test		#将容器mycon中的/www目录拷贝到主机的/tmp/test目录中
docker rename 原来容器名 新容器名	#更改容器名
```

### 6.2、组件命令

docker提供了3个工具docker-Machine  、docker-Swarm 、docker-Compose，安装docker时默认不提供工具的。如果想要使用的时候需要安装。

**docker-Machine**

能够帮助我们在不同平台中快速安装和统一管理docker程序

**docker-Swarm**

能够帮助我们在管理集群中高效运行。

**docker-Compose**

属于应用层，能够帮助我们在集群中快速部署，管理多个容器组成的项目工具，可以根据负载情况随时扩展。

在linux环境下安装环境要求，必须要先安装docker，docker内核版本不低于1.7.1

[docker compose下载地址](<https://github.com/docker/compose/releases>)   安装方式如下：

```bash
curl -L https://github.com/docker/compose/releases/download/1.25.0-rc1/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose		#下载docker-compose
chmod +x /usr/local/bin/docker-compose		#下载完成后赋权，就可以了

#卸载docker-compose
rm /usr/local/bin/docker-compose
```

## 7、容器导出和导入

容器中的所有更改都是在沙盒环境下,当容器停止后所有的更改都会丢失，如果需要保存修改，可以用docker commit命令，把修改的容器保存成一个新的镜像。

```shell
docker ps -la		# 找到CONTAINER ID 然后使用命令
docker commit -m "ifconfig" c738ec435db3 centos:7.4		#保存成镜像-m参数为镜像层写一条提交信息
docker images		#可以查到刚刚保存的镜像，然后就可以把保存好的镜像迁移到其他环境了
```

**容器的导入和导出：**不管容器是否处于运行的状态，都可以使用docker export命令将容器保存到压缩文件里

```bash
docker ps		#找出需要导出的容器ID （加上-a参考，可以列出所有的容器）
docker export 39ec3c12291d > centos.tar.gz		#导出容器
docker export -o centos.tar.gz 39ec3c12291d		#也可以用这个方式导出容器
#-----------------
docker import centos.tar.gz		#导入容器，导入的容器不是在容器列表里，而是在本地镜像库里
```

## 8、数据卷

为了达到从外界获取文件以及持久化存储文件的目的，docker提出了数据卷的概念，数据卷就是挂载在容器内文件系统中的文件或者目录，当容器退出或者删除之后，数据卷不会受到影响，会依然存在docker中。容器与数据卷的关系，就像linux系统下对文件或目录进行的mount操作。

<!--创建数据卷的方式有很多种，比较常用的就是创建容器的时候一同创建数据卷，使用docker create 或docker run 命令创建容器时 加上 -v参数-->

``` bash
docker run -itd -v /data:/data fce4312d2ece /bin/bash	#通过-v参数指定的 会自动挂载到容器中。
docker run -itd --privileged=true -v /data:/data fce4312d2ece /bin/bash		#解决不能写入的问题需要加参数--privileged=true
docker create --name web -v /html -v /var/log  nginx	#通常情况下挂载多个数据卷到容器，多次使用-v 参数为镜像层写一条提交信息
docker run -itd -v /data:/data:ro fce4312d2ece /bin/bash	#加上ro参数表示只读数据卷
# 使用专有的命令创建数据卷----------------------------
docker volume create --name fan		#然后使用docker create或docker run命令时，使用-v参数<name>:<dest>
docker create  --name web -v /fan:/fan nginx:latest	#name表示数据卷名称,dest表示数据卷挂载到容器中的路径,如果这个目录已经存在,会把原来的目录隐藏起来
docker ps -a #查找到刚创建的容器id
docker start 容器ID #启动容器，然后验证
```

<!-- 报错出现权限拒绝问题参考：https://blog.csdn.net/skh2015java/article/details/82660020 -->

因为数据卷时脱离容器存在的，我们删除或停止容器时，数据卷的数据不会删除。[^已经启动的容器是无法挂载新的外部目录进行的]

```dockerfile
docker volume list	=#查找宿主机中所有存在的数据卷
docker volume rm <volume name>	#删除数据卷
docker rm -v nginx	#删除容器时加上-v参数，随容器一起删除数据卷,只有数据卷没有被其他容器使用的时候，才会将这个数据卷删除，拥有名称的数据卷不会因为使用-v参数而被删除
```

**数据卷容器**

通过挂载宿主机目录的方式实现长久保存数据，这个方法存在弊端，(破坏docker的统一性) 解决这个问题可以用数据卷容器管理数据卷，数据卷容器是专门用于存放数据卷的容器，我们在其他的容器中需要使用数据卷时，就不在把宿主机的目录当做数据卷挂载了，而是从数据卷容器中将数据卷挂载。

**创建数据卷容器**：数据卷容器的创建和普通容器的创建方法一样，使用数据卷容器时无须保证数据卷容器处于运行状态，

```bash
docker create --name myvolume -v /data:/data f32a97de94e1	#创建数据卷容器和创建数据卷方式创建一样
#同个数据卷容器可以绑定多个数据卷,为了更准确的管理数据卷，建议将不同的数据卷容器存放不同的数据卷或者将数据卷分类分别放在不同的数据卷容器中。
docker inspect myvolume | grep data		#查看宿主机与数据卷对应的目录路径
# 数据卷容器只是其他容器与数据卷连接的桥梁，在创建容器时，通过--volumes-from参数可以指定数据卷容器,加上多个--volumes-from参数，就可以同时使用多个数据卷容器
docker run -itd --name nginx --volumes-from myvolume nginx:latest /bin/bash	#创建容器时,加上--volumes-from参数可以挂载指定容器中的所有数据卷,这些数据卷的挂载点,会和数据卷容器的目录一样。
```

**数据的迁移**

要导出数据，需要创建新的容器，并将其连接到有我们数据的数据卷容器上，容器运行后，就可以进入容器执行打包命令，并将导出的数据放置到挂载的宿主机目录上。

```dockerfile
docker run -it --volumes-from myvolume -v $(pwd):/backup --name exporter --rm centos /bin/bash	#进入容器
tar cf /backup/data.tar /data	#打包备份到 data
exit	#退出
#也可以在创建容器时就把容器的启动命令设置为打包数据的命令
docker run -it --volumes-from myvolume -v $(pwd):/backup --rm 3c682e5cd3c5 tar -cvf /backup/data.tar.gz /data #备份数据卷的数据
#当容器启动时tar程序就会把数据卷文件夹中的数据打包到data.tar中，使用了--rm参考，表示容器停止后会自动删除。
```

**恢复导出的数据**

恢复数据之前，我们要创建一个新的容器，挂载宿主机的数据目录，并连接到包含目标数据卷的数据卷容器上，然后运行并进入容器，在容器中解压包。

```
docker run -it --volumes-from myvolume -v $(pwd):/backup --rm centos tar -zxvf /backup/data.tar.gz
```

[关于数据卷的备份和迁移可参考](https://blog.csdn.net/u013870094/article/details/79366542)：https://blog.csdn.net/u013870094/article/details/79366542

## 9、网络介绍

### 9.1、网络访问介绍

在docker中程序访问外网的主机，可以通过docker0(网桥)转发到宿主机的外网网卡上，所以在容器中，我们可以直接访问宿主机能访问的网络。

外部网络与容器进行连接，也必须通过docker0（网桥）的网关，有可能存在多个容器，所以就无法对所有的容器同时进行转发。对于我们想让外部网络访问容器，可以向docker提供接受访问的端口,实现与宿主机的端口绑定。

创建容器时使用参数 -P(大写) 将容器的端口随机绑定到宿主机的端口上。
``` dockerfile
docker run -itd -P nginx	#启动nginx容器
# docker会在宿主机上查找可用的端口,绑定在容器上，相同镜像创建出来的不同容器,即使容器的端口相同，在宿主机上的端口也不会相同
docker ps -l	#列出容器,查看实际绑定到宿主机上的端口
```

创建容器时使用参数-p(小写)实现宿主机和容器之间的端口映射

```shell
docker run -d -p 80:80 -p 443:443 nginx	#后台运行容器并进行端口映射
```

有时候，我们不但需要做端口映射，还希望对外部主机进行限制，通过 -p ip:hport:cport 这种形式。

```shell
docker run -d -p 192.168.10.1:80:80 nginx	#这样映射后，容器只会收到来自主机IP为192.168.10.1的请求
```

有时候，一个容器运行的应用程序，也需要和另一个容器的应用程序进行数据交换，此时，就需要通过容器连接来完成2个容器之间的数据交换。要设置容器间通信，我们可以通过在创建容器时加上 --link 参数来实现。设置容器间通信，只需要指定被连接的容器，并不需要指明被连接容器的端口，也不需要通过-p参数映射端口

我们创建一个mysql容器，并让一个web服务器连接到它

```bash
docker run -d --name mysql mysql		#创建mysql容器
docker run -d -p 80:80 --name myweb --link mysql nginx	#启动一个nginx 连接到mysql容器
```

在某些情况下，被连接的容器名称可能与连接容器内的某些配置重名，对应这样的情况，docker支持容器间使用别名进行连接的方式。

```bash
docker run -d -p 80:80 --name myweb --link mysql:db nignx  #使用--link name:alias 的方式设置被连接的容器，这样定义后，我们在nginx容器中访问mysql容器时就可以使用db作为访问时的主机名
docker exec -it myweb /bin/bash		#进入容器后可以使用 env 命令查看环境变量信息
```

在容器间可进行连接的配置建立之后，可以在容器信息中发现相关条目。可通过 docker inspect 查看。

容器间通信的主要目的并不是实现网络的访问，而是将网络间访问的方式更抽象化，由于宿主机的网络环境并不固定，所以就无法保障docker申请到的网段总是一致的，在这样的情况下，就需要修改容器中访问其他容器所使用的IP地址，但这样做可能就达不到docker快速部署的目的。docker可以通过修改hosts的方式实现一种即简单有无需修改IP地址的方案。使用docker连接其他容器时，docker会在/etc/hosts中添加一条基于容器名称或者别名的条目，这个解析指向正是被连接的容器。

当我们需要在容器中使用被连接容器地址的时候，只使用容器的名称或设置的别名即可。这样就巧妙的利用了域名解析实现了变化的IP到固定的名称的转变。

当我们创建容器并使用容器连接时，docker会在容器中做两件事，一是修改/etc/hosts文件，二是增加相关的环境变量。

docker网络主要有 以下技术实现

network namespace ：实现了网络资源的隔离，对隔离环境提供了网络设备，协议栈，路由表，防火墙，/proc/net目录，/sys/class/net目录，端口表等网络配置和实现。

veth pair：实现了打穿隔离环境的网络传输数据通道，在docker中，他的一端连接到容器中虚拟的网卡上，另一端连接到宿主机中专用的网桥上，通过这种方式实现了docker容器外部网络的互通。

linux bridge：放置在宿主机中的网桥，起到网络交换机的作用，因为容器网络通过veth pair连接到网桥上，所有他能够在容器间转发网络数据。

iptables：用于提供网络数据透传，net等功能，也可以利用他实现docker网络的防火墙等网络安全防护的需求。

通过docker network ls 命令可以查看当前docker中的网络列表，首次使用和安装docker时，docker会自动创建三个默认网络，

```bash
docker network ls  #通过命令查看会显示容器默认使用的网络。
NETWORK ID          NAME                DRIVER              SCOPE
8df1b44031a1        bridge              bridge              local
59468096be67        host                host                local
8bfdf04b64f2        none                null                local
```

默认情况下，==创建的容器都会连接到**bridge**这个网络上==，他对应的就是宿主机上的docker0网卡。一旦安装了docker，就可以看到宿主机上创建了docker0网卡。他扮演着网桥的角色。如果想改变容器中使用的网络可以在创建容器中使用 ==--network参数==。

```bash
docker run -it --name centos --network none centos:latest /bin/bash
```

**none 表示不使用网络**，容器如果绑定到none网络上，则不会为容器分配网络地址。

**host 则是直接使用宿主机的网络环境**。

### 9.2、自定义网络

有时候，我们希望容器某些容器组成小型网络，不让其他容器访问到。就需要为这个容器单独分配网络。将这些容器放入到单独的网络之前，要先创建一个网络供容器连接。==--driver==参数用来指定网络所基于的网络驱动,也可以简写为-d

```bash
docker network create --driver bridge isolated #这样就创建了名为isolated的网络环境，可使用docker network ls查看
```

要使容器和外部通信都正常运行，最关键的就是要保证网络数据转发，也就是ip forward功能正常启动。docker deamon 启动时，我们可以通过 --ip-forward参数来控制docker是否使用ip forward (默认配置是开启的)。所以通常的情况下我们不需要对其专门设置。如果已经开启了对ip forward的支持，但容器仍然无法连接外部网络，可以先检查宿主机系统中的ip forward是否被禁用。然后在查看本机的防火墙。

```bash
[root@fan15 opt]# sysctl net.ipv4.conf.all.forwarding
net.ipv4.conf.all.forwarding = 1		#启动时参数是1，如果是0时 表示ip forward处于禁用状态。
```

### 9.3、管理容器网络

docker提供了4个管理网络的命令，分别如下：

创建网络(docker network create)：

```bash
# --subnet 参数创建一个拥有指定子网范围的网络
docker network create --subnet 192.168.100.1/24 cnet	#使用docker network ls 查看就可以看到创建了cnet网络
```

获取网络列表(docker network ls)，

获取网络信息(docker network inspect)，

删除网络(docker network rm)。

以上都是介绍容器网络的操作，如果要让容器使用指定的网络，可以在创建容器时使用 --network 参数。或者随时通过(docker network connect)命令让容器连接到指定的网络

```bash
docker run -it --network cnet contos	#新创建容器的时候
docker network connect cnet mysql		#将cnet网络连接到名为mysql的容器上，这样容器里就会存在2个网卡，一个bridge自动分配的网络和一个cnet网络
docker network disconnect cnet mysql	#随时将容器的网络断开，就像拔掉网线一样。
```

### 9.4、配置docker0网桥

配置docker0网桥的时候只能在docker启动前进行。

```bash
dockerd --bip=192.168.1.1/24	#设置docker0的IP地址
dockerd --fixed-cidr=192.168.1.0/24		#设置docker0的网段
dockerd --mtu=65536		#设置docker0的最大数据包长度

brctl show		#查看容器和宿主机建立的连接
```

**自定义网桥**：通过brctl和ip命令，可以在宿主机上创建网桥和配置网桥

```bash
brctl addbr ymbr0
ip addr add 192.168.99.1 dev ymbr0
ip link set dev ymbr0 up
ip addr show ymbr0		#查看刚刚创建的网桥信息

#docker没有启动时，可以通过启动docker时加上 -b 或--bridge参考来指定网桥，如果docker已经启动，则需要先停止，然后在替换原有的docker0网桥
dockerd --bridge ymbr0
```

**配置DNS**：在linux系统中与DNS解析相关的主要有3个配置文件，在etc目录下hostname(主机名，主要是在其他主机的网络发现时告知对方自身的名称) , hosts(用于本地域名解析，他的内容就是域名及对应的解析IP) , resolv.conf(提供DNS服务器的列表，当本地解析无效时，会向互联网请求解析时所连接的服务器地址)

docker容器的文件系统是直接基于对应的基础镜像所建立的，这3个文件存在docker镜像中，我们可以在容器运行时使用mount 命令查看

```bash
[root@33f9a73a3f1a /]# mount
overlay on / type overlay (rw,relatime,seclabel,lowerdir=/var/lib/docker/overlay2/l/3NBRROPZ6AUMF6JTRODTA3H667:/var/lib/docker/overlay2/l/FILL3DZP6TINVLBPUYTKSOYEFA,upperdir=/var/lib/docker/overlay2/a38bdc234f5606823feaca99948439de90c83870eeda19d53da1bced473242e0/diff,workdir=/var/lib/docker/overlay2/a38bdc234f5606823feaca99948439de90c83870eeda19d53da1bced473242e0/work)
proc on /proc type proc (rw,nosuid,nodev,noexec,relatime)
tmpfs on /dev type tmpfs (rw,nosuid,seclabel,size=65536k,mode=755)
devpts on /dev/pts type devpts (rw,nosuid,noexec,relatime,seclabel,gid=5,mode=620,ptmxmode=666)
sysfs on /sys type sysfs (ro,nosuid,nodev,noexec,relatime,seclabel)
tmpfs on /sys/fs/cgroup type tmpfs (ro,nosuid,nodev,noexec,relatime,seclabel,mode=755)
cgroup on /sys/fs/cgroup/systemd type cgroup (ro,nosuid,nodev,noexec,relatime,seclabel,xattr,release_agent=/usr/lib/systemd/systemd-cgroups-agent,name=systemd)
cgroup on /sys/fs/cgroup/blkio type cgroup (ro,nosuid,nodev,noexec,relatime,seclabel,blkio)
cgroup on /sys/fs/cgroup/cpu,cpuacct type cgroup (ro,nosuid,nodev,noexec,relatime,seclabel,cpuacct,cpu)
cgroup on /sys/fs/cgroup/net_cls,net_prio type cgroup (ro,nosuid,nodev,noexec,relatime,seclabel,net_prio,net_cls)
cgroup on /sys/fs/cgroup/devices type cgroup (ro,nosuid,nodev,noexec,relatime,seclabel,devices)
cgroup on /sys/fs/cgroup/perf_event type cgroup (ro,nosuid,nodev,noexec,relatime,seclabel,perf_event)
cgroup on /sys/fs/cgroup/hugetlb type cgroup (ro,nosuid,nodev,noexec,relatime,seclabel,hugetlb)
cgroup on /sys/fs/cgroup/cpuset type cgroup (ro,nosuid,nodev,noexec,relatime,seclabel,cpuset)
cgroup on /sys/fs/cgroup/pids type cgroup (ro,nosuid,nodev,noexec,relatime,seclabel,pids)
cgroup on /sys/fs/cgroup/memory type cgroup (ro,nosuid,nodev,noexec,relatime,seclabel,memory)
cgroup on /sys/fs/cgroup/freezer type cgroup (ro,nosuid,nodev,noexec,relatime,seclabel,freezer)
mqueue on /dev/mqueue type mqueue (rw,nosuid,nodev,noexec,relatime,seclabel)
# /dev/mapper/centos-root on /etc/resolv.conf type xfs (rw,relatime,seclabel,attr2,inode64,noquota)
# /dev/mapper/centos-root on /etc/hostname type xfs (rw,relatime,seclabel,attr2,inode64,noquota)
# /dev/mapper/centos-root on /etc/hosts type xfs (rw,relatime,seclabel,attr2,inode64,noquota)
shm on /dev/shm type tmpfs (rw,nosuid,nodev,noexec,relatime,seclabel,size=65536k)
devpts on /dev/console type devpts (rw,nosuid,noexec,relatime,seclabel,gid=5,mode=620,ptmxmode=666)
proc on /proc/bus type proc (ro,relatime)
proc on /proc/fs type proc (ro,relatime)
proc on /proc/irq type proc (ro,relatime)
proc on /proc/sys type proc (ro,relatime)
proc on /proc/sysrq-trigger type proc (ro,relatime)
tmpfs on /proc/asound type tmpfs (ro,relatime,seclabel)
tmpfs on /proc/acpi type tmpfs (ro,relatime,seclabel)
tmpfs on /proc/kcore type tmpfs (rw,nosuid,seclabel,size=65536k,mode=755)
tmpfs on /proc/keys type tmpfs (rw,nosuid,seclabel,size=65536k,mode=755)
tmpfs on /proc/timer_list type tmpfs (rw,nosuid,seclabel,size=65536k,mode=755)
tmpfs on /proc/timer_stats type tmpfs (rw,nosuid,seclabel,size=65536k,mode=755)
tmpfs on /proc/sched_debug type tmpfs (rw,nosuid,seclabel,size=65536k,mode=755)
tmpfs on /proc/scsi type tmpfs (ro,relatime,seclabel)
tmpfs on /sys/firmware type tmpfs (ro,relatime,seclabel)
[root@33f9a73a3f1a /]# 可以看出这三个文件都是挂载的形式存在的
```

我们在创建容器时指定相关参数，可以对容器内的DNS相关内容进行配置。==通过-h或者--hostname参数==可以配置容器的主机名，这个配置对写到/etc/hostname里。==通过 --dns参数==可以指定新的DNS服务器，这个配置会写入/etc/resolv.cnf

```bash
docker run -it --name centos -h fan centos:latest /bin/bash		#更改容器的主机名
docker run -it --name centos -h fan --dns 8.8.8.8 centos:latest /bin/bash	#指定DNS服务器
```

## 10、dockerfile

为了简化制作镜像的过程，方便在多台机器上共享镜像，docker提供了一种可以通过配置文件创建镜像的方式----使用dockerfile构建镜像，这种方式是将制作的镜像操作全部写入到一个文件中，然后 docker build 命令可以读取这个文件中的所有操作，并根据这些配置创建出相应的镜像。

dockerfile中的内容主要以两种形式出现：注释行和指令行，以#开头的文本是注释行[^注意： 在dockerfile中，并非所有的以#开头的行都是注释行，有一类特殊的参数是通过#开头的行来指定的]，指令行主要分为两部分，行首是INSTRUCTION，即指令的名称，然后是arguments，即指令所接收的参数。指令是不区分大小写，但为了更清晰的分辨指令和参数，指令一般是大写。

```dockerfile
# Comment
INSTRUCTION arguments
# directive=value		#这样的称为解析指令行，主要是提供一项解析dockerfile需要使用的参数。解析指令行很少被用到。
```

下面是一个简单的构建redis镜像的dockerfile，文件名为dockerfile，如果是其他文件名需要 在构建镜像的时候 加上 -f 指定dockerfile文件

```dockerfile
FROM centos:7.4
MAINTAINER fana "17602199364@163.com"

WORKDIR /home

RUN yum install -y wget gcc && \      
        rpm --rebuilddb && \
        yum -y install gcc automake autoconf libtool make && \
        yum -y install net-tools && \
        yum -y install tar && \
        wget http://download.redis.io/redis-stable.tar.gz && \
        tar -xvzf redis-stable.tar.gz && \
        mv redis-stable/ redis && \
        rm -f redis-stable.tar.gz && \
        yum clean all && \
        cd redis && \
        make && make install
        
EXPOSE 6379
ENTRYPOINT redis-server /home/redis.conf
CMD ["redis-server"]
# --------------------------------------------------------------- #
# 第一条指令from centos:7.2.1511 中的form指令，表示我们要构建镜像所基于的镜像，通常情况下我会使用一个系统镜像来构建我们的应用，接下来是linux的命令。表示我们构建镜像是所执行的操作。 
# 因为有make命令，所以要安装   yum -y install gcc automake autoconf libtool make 
# 想要查看ifconfig，所以安装net-tools     yum -y install net-tools
# 利用这个Dockerfile构建镜像命令：
docker build -t centos/redis .	#执行命令后会自动查找当前目录下的dockerfile
docker build -f dockerfile.redis -t redis:001 .	#加上-f指定配置文件，后面 . 是必须要加上的
#启动容器： 
docker run -d --name redis -p 6379:6379 redis：001
```

```dockerfile
# 天翼云paas平台dockerfile
FROM hub.paas/base/centos-jdk8-cn:latest
ADD *.tar.gz /usr/local/gateway-service/
RUN chmod u+x /usr/local/gateway-service/bin/run.sh
EXPOSE 8888
CMD '/usr/local/gateway-service/bin/run.sh'
```

### 10.1、基础指令(FROM,MAINTAINER)

**FROM指令**：指明基础镜像名称

docker的镜像都是在bootfs层上实现的，但是我们不必每次构建镜像都是从bootfs层开始，我们可以直接在其他已经搭建好的镜像上进行修改，FROM指令就是用来指定我们所要构建的镜像是基于那个镜像建立的，==from指令必须作为第一条指令==，不过在一个dockerfile里是允许出现多个from指令的，以每个from指令为界限，都会生成不同的镜像。

FROM指令主要有以下几种格式

```dockerfile
FROM <image>			#第一种
FROM <image>:<tag>		#第二种
FROM <image>@<digest>	#第三种
# tag 和 digest 都是可选的，当不指定这两项时，docker会使用latest这个tag
```

**MAINTAINER指令** ： 用于提供镜像的作者信息，一般放在FROM命令下面。



### 10.2、控制指令(RUN,WORKDIR,ONBUILD)

控制指令是dockerfile的核心部分，我们可以通过控制指令来描述整个镜像的构建过程

**RUN指令**：构建镜像过程中，我们需要在基础镜像中做很多操作，run指令就是用来给定需要做什么操作的。

RUN指令有两种使用格式

```dockerfile
RUN command param1 param2	#如：RUN mkdir data 这种形式，在构建镜像时，实际上是以shell(/bin/sh)程序来执行操作的，所以基础镜像必须有/bin/sh
RUN ["executbale","param1","param2", ...]	#如：RUN ["/bin/bash","-c","echo hello"] 这种形式可以有效规避在某些基础镜像中没有shell程序，或者用于需要临时切换shell程序的时候,[]中的数据都会按照json字符串的格式解析，只能使用双引号，不能使用单引号或其他符号
```

注意：[^ 在使用RUN指令时，docker排断是否采用缓存构建的依据，是给出的指令是否与生成缓存使用的指令一致，也就是说，本次执行的结果与缓存中不一致，会采用缓存中的数据，而不再执行命令，这可能导致不是我们想要的结果，比如使用RUN apt-get update 时都需要使用最新的结果，可以使用docker build 命令时加上 --no-cache参数的方式解决这个问题]

**WORKDIR指令**：用于切换构建过程中的工作目录。

给出的工作目录可以是绝对目录，也可以是相对目录

```dockerfile
WORKDIR /usr/local	#绝对目路径
WORKDIR local	#如果是相对路径，在切换工作目录时，会参考当前的工作目录进行。

#也可以在workdir指令中使用环境变量
ENV BASEDIR /project
WORKDIR $BASEDIR /www
```

**ONBUILD指令**：这是一条非常特殊的指令，他可以携带别的指令。使用这条指令不会在构建当前镜像时执行，而是在构建其他镜像时使用FROM指令把z这个镜像作为基础镜像时才会执行。简单来说，就是我们在构建子镜像时运行的指令。

```dockerfile
ONBUILD INSTRUCTION arguments	#把我们需要执行的指令放在ONBUILD指令之后就能设置一个构建触发器，当其他dockerfile把这个镜像作为基础镜像并进行构建时，执行完FROM指令之后，设置的ONBUILD指令都将被触发
```

ONBUILD指令，在生成镜像时会写入到镜像的特征列表中，可以使用docker inspect命令看到镜像的构建命令，当子镜像构建完成后，这些指令也都随着消失了。它不会在继承到新构建的镜像中。

### 10.3、引入指令(ADD,COPY)

很多场合下，我们希望将文件加入到即将构建的镜像中，引入指令就可以帮我们实现这个目的。

**ADD指令**：在构建容器的过程中，可能需要将一些软件源码，配置文件，执行脚本等导入到镜像的构建过程，这时可以使用ADD指令将文件从外部传递到镜像内部。

ADD指令有以下两种形式

```dockerfile
ADD <src>... <dest>		#第一种方式 如：ADD hom* /mydir/ 如果我们给出的路径是目录，那么目录本事不会复制到镜像，被复制的是目录的内容
ADD ["<src>", "<dest>"]		#和上面的方式没有太大差别，只是避免文件路径中带有空格的情况。
```

ADD指令可以自动完成对压缩文件的解压，如果我们提供的是能够识别的压缩文件格式，则会自动解压到镜像的目标路径中。

**COPY指令**：dockerfile中还有一种引入文件的方式就是copy，与ADD指令相似

COPY指令二种使用格式

```dockerfile
COPY <src> ... <dest>	# 第一种方式copy 原路径<src>、目标路径<dest>
COPY ["<src>", ... "<dest>"]	#与add指令的规则几乎是一样的。主要区别就是不能识别网址和自动解压。不需要解压的文件可以使用这个指令
```

### 10.3、执行指令(CMD)

执行指令能够通过镜像建立容器时，容器默认执行的命令，我们通常使用这些命令启动镜像中的主要程序

**CMD指令**：docker容器是为运行单独的应用程序而设计的，当docker容器启动时，实际上是对程序的启动。而在dockerfile中，就可以通过CMD指令来创建镜像容器中的主体程序。可以出现多次CMD指令，但只要最后一次CMD命令生效。

CMD指令有三种使用格式

```dockerfile
CMD command param1 param2 ...	#第一种方法：依靠shell命令来执行
CMD ["executable","param1","param2" ...]	#和上面的方法类似，都是取决于是否使用shell程序来执行命令(推荐)
CMD ["param1","param2" ...]		#这种格式 是将参数传给ENTRYPOINT指令
# 需要注意：容器中只会绑定一个应用程序，所以在dockerfile中只能存在一个CMD指令，如果我们填写多个CMD指令，会覆盖掉之前的指令。
```

**ENTRYPOINT指令**：镜像所指定的应用程序在容器运行时，难免需要一些系统服务或者其他程序的支持。我们可以在CMD指令中启动这个服务，但是这样会让启动服务的命令与启动主程序的命令混在一起，ENTRYPOINT指令就是专门用于主程序启动前的准备工作的。

使用ENTRYPOINT指令的方式和CMD指令的方式相似。

```dockerfile
ENTRYPOINT ["executable","param1","param2" ...]	#第一种方式,直接执行程序
ENTRYPOINT command param1 param2 ...	# 使用shell程序来执行。 这两种格式在效果上和CMD是一样的
```

需要注意：当ENTRYPOINT指令被指定时，所有的CMD指令或者通过docker run 等方式的应用程序启动命令，不会在容器启动时执行。而是把这些命令当成参数。所以我们在使用 ENTRYPOINT 时需要特别注意使用的方法。我们应该在避免在使用 ENTRYPOINT指令 时把 CMD指令的形式配置成shell格式，（即：CMD command param ... ）因为这样做，在 ENTRYPOINT 里是以次级命令的方式启动 CMD的shell进程。docker 就不会把容器的生命周期绑定到进程上。可能会造成意想不到的结果。

### 10.4、配置指令(EXPOSE,ENV)

若想对镜像或者通过镜像所创建的容器进行相关的环境或者网络 等配置时，可以通过配置指令来实现。

**EXPOSE指令**：每个容器都有自己的端口系统，相互之间不连通也不共享，容器间的网络通信需要通过docker转接来完成。如果容器的应用程序需要让其他镜像访问到他所提供的端口，就需要显示出对外提供的端口号。我们要生成镜像时对外开放端口，可以使用EXPOSE指令。

EXPOSE指令的使用方法很简单，将需要共享的端口逐个传入即可。

```dockerfile
EXPOSE <prot>
```

需要注意：EXPOSE指令所指定的端口和 docker run 命令中 -p 参数所指定的端口，含义上是有区别的，EXPOSE指令给出的端口，是基于镜像的容器需要敞开的端口，而创建容器时使用的 -p参数指定的端口，是用于建立容器到宿主机外部的端口映射。就是说，要从外部访问容器内程序监听的端口，首先需要通过EXPOSE指令将这个端口标记对外敞开，在根据实际访问的来源进行配置。从其他容器中访问则需要创建该容器时使用 --link 连接到 此容器，从宿主机外部访问则需要创建该容器时使用 -p参数，建立宿主机对外端口与容器端口的转发。

**ENV指令**：在dockerfile中，我们也能指定环境变量，环境变量能够替换dockerfile中其他指令出现的参数，使用ENV指令就能设置dockerfile中的环境变量。

```dockerfile
ENV <key> <value>		#可以指定一个环境变量，在键名之后的数据都会被视为环境变量的值，如 ENV myDog The Dog
ENV <key>=<value>		#这种格式能够一次指定多个环境变量，并且可以使用\进行换行连接 如 ENV myDog="The Dog" myCat=The\ Cat \
```

**LABEL指令**：使用LABEL指令，可以为即将生成的镜像提供一些元数据作为标记，这些标记能够帮助我们展示镜像的信息。

LABEL指令的用法

```dockerfile
LABEL version="1.0"	#在LABEL指令之后，带入我们希望加入的元数据的键值对，如果有多个键值对，可以使用空格分隔他们如下，在键和值中，如果带有空格，可以使用引号，如果数据过长可以使用 \ 进行换行
LABEL "multi.labell"="value" "com.example.vendor"="You Ming" #推荐把所以的标记写到一个LABEL指令中
```

**USER指令**：USER指令用于设置执行用户，我们可以传入用户的名称或UID作为USER指令的参数

```dockerfile
USER nginx
```

**ARG指令** ：在构建过程中，我们有时还需要进行配置或者使用一些变量，ARG指令就是为我们提供了设置变量的方法。ARG指令与ENV指令有很大的不同，ENV指令用于配置环境变量，他的值会影响镜像的编译，也会体现在容器的运行中，需要改变环境变量时，要在容器启动时进行赋值。而ARG指令则只用于构建镜像的过程中，其效果不会作用于基于此镜像的容器，而覆盖参数的方式也是通过docker build 中的--build-arg来进行的

使用ARG的方式

```dockerfile
ARG <name>	#使用这样的形式定义的格式，变量的值是有外部传递过来
ARG <name>=<default>	#这样形式定义的格式，表示我们未提供变量时就使用默认值
```

用法展示

```dockerfile
FROM busybox
ARG user
USER $user
# 上面是示例，当我们真正构建镜像时，使用 --build-arg参数赋值
docker build --build-arg user=root ./busybox
```

**STOPSIGNAL指令**：当我们停止容器时，docker会向容器中的应用程序传递停止信号，我们可以通过STOPSIGNAL指令修改docker所传递的信号

定义的格式

```dockerfile
STOPSIGNAL 9	#linux内核syscall信号的数字表示
STOPSIGANL SIGKILL	#信号的名称表示
```

**SHELL指令**：CMD ENTRYPOINT等指令都是支持以shell形式执行，SHELL指令可以为他们选定shell程序，

SHELL指令的使用格式

```dockerfile
SHELL ["executable","parameters"]	#使用方法
SHELL ["/bin/bash","-c"]	#shell默认使用的是/bin/sh 若要改为/bin/bash，可以使用这个指令
```

### 10.5、特殊用法

除了基本的指令和备注信息，docker中，我们还可以通过一些特殊的使用方法，控制镜像的构建过程。

**环境变量**：通过ENV指令定义环境变量后，就可以在之后的命令中进行环境变量的替换了，环境变量的解析支持 ADD，COPY，ENV，EXPOSE，LABEL，USER，WORKDIR，VOLUME，STOPSIGNAL 这些指令，

普通的环境变量替换方法是使用 “$+变量名”的方式

```dockerfile
ENV variable value
RUN echo $variable	# value
# 也可以使用花括号将变量名包裹起来，
ENV variable value
RUN echo $variable	#value
RUN echo ${variable}_1	#value
# 如果使用的变量 刚好是我们想要使用的内容，可以使用转义符号去除环境变量的解析过程
ENV variable value
RUN echo \$variable 
```

**指令解析**：使用RUN 等指令时，可以通过 \ 来进行命令的换行。但在Windows系统中，目录的分隔符就是 \ 要解决这个问题，就要利用dockerfile中注释的一种特殊用法：解析指令行

解析指令行的一般用法是 ``` # directive = value ```	<!-- 参数名和值分布在登号的两端，参数名是区分大小写，并且参数名与值周围的空格也会被忽略掉。

### 10.6、使用dockerfile构建镜像

使用dockerfile创建apache镜像

```dockerfile
#apache server
FROM centos:latest
RUN yum update -y && \
    yum install vim -y && \
    yum install httpd -y && \
    yum install net-tools -y && \
    yum clean all

RUN sed -i 's/#ServerName www.example.com/ServerName localhost/g' /etc/httpd/conf/httpd.conf

EXPOSE 80

CMD ["/usr/sbin/httpd","-D","FOREGOUND"]
# 构建镜像，执行命令如下
docker build -f dockerfile.apache -t apache:v1 .
```

使用dockerfile 构建nginx镜像

```dockerfile
#nginx1.15.12

FROM centos:latest
ADD *.tar.gz /opt/

RUN yum update -y && \
    yum install vim -y && \
    yum install net-tools -y && \
    yum install gcc gcc-c++ -y && \
    cd /opt/zlib-1.2.11 && ./configure && make && make install && \
    cd /opt/pcre-8.43 && ./configure && make && make install && \
    cd /opt/openssl-1.1.1b && ./config && make && make install && \
    cd /opt/nginx-1.15.12 && ./configure --with-http_ssl_module --with-http_flv_module --with-http_mp4_module --with-http_realip_module --with-http_stub_status_modul
e --with-http_gzip_static_module --with-openssl=/opt/tools/openssl-1.0.2d --with-pcre=/opt/tools/pcre-8.36 --with-zlib=/opt/tools/zlib-1.2.8 --with-pcre && \
    make && make install

EXPOSE 80 443

CMD ["nginx","-g","doemon off;"]
```

使用dockerfile 构建tomcat镜像

```dockerfile
# tomcat server

FROM java:8-jre
RUN apt-get update && apt-get install -y tomcat8
EXPOSE 8080
CMD ["/usr/share/tomcat8/bin/catalina.sh","run"]
```

使用dockerfile 构建mysql镜像

使用dockerfile 构建MongoDB镜像

使用dockerfile 构建redis镜像

```dockerfile
# redis

FROM debian:jessie

RUN apt-get update \
    && bulidDeps='gcc make libc6-dev wget' \
    && apt-get install -y --no-install-recommends $buildDeps \
    && wget -O redis.tgz "http://download.redis.io/releases/redis-3.2.3.tar.gz" \
    && make -p /usr/src/redis \
    && tar -zxf redis.tgz -C /usr/src/redis --strip-components=1 \
    && rm redis.tgz \
    && cd /usr/src/redis \
    && make \
    && make install \
    && cd / \
    && rm -rf /usr/src/redis \
    && apt-get purge -y --auto-remove $buildDeps
EXPOSE 6379
CMD ["redis-server"]
```

使用dockerfile 构建java镜像

```dockerfile
FROM debian:jessie
RUN echo 'deb http://httpredir.debian.org/debian jessie-backports main' > /etc/apt/sources.list.d/jessie-backports.list \
    && apt-get update \
    && apt-get install -y openjdk-8-jre \
    && rm -rf /var/lib/apt/lists/*
CMD ["java","-version"]
```

## 11、docker资源限制

==限制容器分配最大内存==，在创建容器时加上参数 -m 或 --memory参数。

```dockerfile
docker run -d -m 512M nginx:latest		#现在物理内存
docker run -d -m 512M --memory-swap 1024M nginx:latest	#加上--memory-swap参数可以控制交换区内存的大小。
```

在docker里，没有直接使用具体的参数配置而是==通过权重来分配CPU==，使用-c或者--cpu-shares参数，设置CPU占用的权重，不能对CPU资源硬性限制。

```bash
docker run -d -c 500 nginx:latest	#设置资源占用权重，只是限制实际允许个容器对CPU资源的需求。
```

对于硬盘，可以通过--device-read-bps和--device-write-bps命令限制指定硬盘的读写速度，还有--device-read-iops和--device-write-iops限制IO

```bash

```

创建容器时加上 --ulimit 参数来配置ulimit,可以修改core dump文件大小，数据段大小，文件句柄数，进程栈深度，CPU时间，单一用户进程数，进程虚拟内存等，

```bash
docker run -d --name nginx --ulimit cpu=1000 nginx:latest
docker exec -it nginx /bin/bash		#然后进入容器查看
#执行命令 ulimit -t
docker dockerd --default-ilimit cpu=1000		#这样配置容器默认的Ulimit限制。
```

```dockerfile
docker stats --no-stream centos	#查看容器占用的资源，可以显示CPU,内存，IO，进程数量等情况。类似top命令
```

# 第二章：docker实践

## 1、在docker中使用SSH服务

ssh服务作为远程操作服务主机的主要方式之一，在docker中，我们也可以通过ssh连接和访问到容器的内部。

```dockerfile
service sshd start	#首先启动ssh
```

**使用ssh服务容器**



**构建ssh服务镜像**

```dockerfile
# ssh server
# VERSION 0.0.1
FROM ubuntu:16.0.4
MAINTAINER fana
RUN apt-get update && apt-get install -y openssh-server
RUN mkdir /var/run/sshd
RUN echo 'root:hellossh' | chpasswd
RUN sed -i 's/PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config
```

## 2、练习

本次实践的目标是搭建一个完整的web服务器，我们采用 nginx + memcached + mysql + PHP的架构，以上镜像我们从docker hup上获取。

**1、启动容器**

```bash
docker run -d --name memcached memcached:latest	#启动memcached
docker run -d --name mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123456 mysql:latest	#启动mysql，并设置密码123456
# 启动PHP 代码挂载目录时/app,需要让PHP连接到mysql和memcached
docker run -d --name php -v /app:/app --link mysql --link memcached php:latest /bin/bash
# 启动nginx
docker run -d --name nginx -v /app/nginx/nginx.conf:/etc/nginx/nginx.conf -v /app:/app --link php -p 80:80 nginx:latest /bin/bash

```



**2、程序配置**

等程序配置完成后，就可以做基本的测试了





## 3、docker运维技巧





# 第三章 kubernetes基础

以centos 7 为例，搭建kubernetes集群，建议先关闭防火墙和SElinux，最简单的安装方法就是yum安装

参考：https://blog.csdn.net/networken/article/details/89599004

参考：https://www.cnblogs.com/hongdada/p/9761336.html

**单机版的kubernetes环境**

```bash
systemctl disable firewalld	#禁止开机自启动
systemctl stop firewalld		#停止防火墙
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux	#禁用selinux
yum install -y etcd kubernetes		#yum安装

# 服务的启动程序
systemctl start etcd &&
systemctl start docker &&
systemctl start kube-apiserver &&
systemctl start kube-controller-manager &&
systemctl start kube-scheduler &&
systemctl start kubelet &&
systemctl start kube-proxy
```

## 1、使用kubeadm在CentOS上搭建Kubernetes集群

练习环境说明：[参考](https://www.kubernetes.org.cn/3808.html?tdsourcetag=s_pctim_aiomsg)      [参考](https://www.kubernetes.org.cn/5462.html)  共3台机器参考如下

| 主机名称 | IP地址         | 部署软件                              | 备注   |
| -------- | -------------- | ------------------------------------- | ------ |
| M-kube12 | 192.168.10.12  | master+etcd+docker+keepalived+haproxy | master |
| M-kube13 | 192.168.10.13  | master+etcd+docker+keepalived+haproxy | master |
| M-kube14 | 192.168.10.14  | master+etcd+docker+keepalived+haproxy | master |
| N-kube15 | 192.168.10.15  | docker+node                           | node   |
| N-kube16 | 192.168.10.16  | docker+node                           | node   |
| VIP      | 192.168.10.100 |                                       | VIP    |

### 1.1、环境准备

```bash
# 1、关闭防火墙，SELinux，安装基础包
yum install -y net-tools conntrack-tools wget vim  ntpdate libseccomp libtool-ltdl lrzsz		#在所有的机器上执行,安装基本命令

systemctl stop firewalld && systemctl disable firewalld		#执行关闭防火墙和SELinux

sestatus	#查看selinux状态
setenforce 0		#临时关闭selinux
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config

swapoff -a			#关闭swap
sed -i 's/.*swap.*/#&/' /etc/fstab

# 2、设置免密登陆
ssh-keygen -t rsa		#配置免密登陆
ssh-copy-id <ip地址>		#拷贝密钥

# 3、更改国内yum源
mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.$(date +%Y%m%d)
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.cloud.tencent.com/repo/centos7_base.repo
wget -O /etc/yum.repos.d/epel.repo http://mirrors.cloud.tencent.com/repo/epel-7.repo
#docker源
wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -O /etc/yum.repos.d/docker-ce.repo

#配置国内Kubernetes源
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
yum clean all && yum makecache -y

#----------------------
[root@localhost ~]#  cat >> /etc/yum.repos.d/kubernetes.repo << EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=0
EOF

# 4、配置内核参数，将桥接的IPv4流量传递到IPtables链
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_nonlocal_bind = 1
net.ipv4.ip_forward = 1
vm.swappiness=0
EOF
sysctl --system

# 5.配置文件描述数
echo "* soft nofile 65536" >> /etc/security/limits.conf
echo "* hard nofile 65536" >> /etc/security/limits.conf
echo "* soft nproc 65536"  >> /etc/security/limits.conf
echo "* hard nproc 65536"  >> /etc/security/limits.conf
echo "* soft  memlock  unlimited"  >> /etc/security/limits.conf
echo "* hard memlock  unlimited"  >> /etc/security/limits.conf

# 6.加载IPVS模块
yum install ipset ipvsadm -y
cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
EOF
#执行脚本
chmod 755 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules && lsmod | grep -e ip_vs -e nf_conntrack_ipv4


#参考别人的
cat << EOF > /etc/sysconfig/modules/ipvs.modules 
#!/bin/bash
ipvs_modules_dir="/usr/lib/modules/\`uname -r\`/kernel/net/netfilter/ipvs"
for i in \`ls \$ipvs_modules_dir | sed  -r 's#(.*).ko.*#\1#'\`; do
    /sbin/modinfo -F filename \$i  &> /dev/null
    if [ \$? -eq 0 ]; then
        /sbin/modprobe \$i
    fi
done
EOF

chmod +x /etc/sysconfig/modules/ipvs.modules 
bash /etc/sysconfig/modules/ipvs.modules
```

### 1.2、配置keepalived

```bash
yum install -y keepalived

#10.12机器上配置

cat <<EOF > /etc/keepalived/keepalived.conf
global_defs {
   router_id LVS_k8s
}

vrrp_script CheckK8sMaster {
    script "curl -k https://192.168.10.100:6443"
    interval 3
    timeout 9
    fall 2
    rise 2
}

vrrp_instance VI_1 {
    state MASTER
    interface ens33
    virtual_router_id 100
    priority 100
    advert_int 1
    mcast_src_ip 192.168.10.12
    nopreempt
    authentication {
        auth_type PASS
        auth_pass fana123
    }
    unicast_peer {
        192.168.10.13
        192.168.10.14
    }
    virtual_ipaddress {
        192.168.10.100/24
    }
    track_script {
        CheckK8sMaster
    }

}
EOF

#13机器keepalived配置
cat <<EOF > /etc/keepalived/keepalived.conf
global_defs {
   router_id LVS_k8s
}

vrrp_script CheckK8sMaster {
    script "curl -k https://192.168.10.100:6443"
    interval 3
    timeout 9
    fall 2
    rise 2
}

vrrp_instance VI_1 {
    state BACKUP
    interface ens33
    virtual_router_id 100
    priority 90
    advert_int 1
    mcast_src_ip 192.168.10.13
    nopreempt
    authentication {
        auth_type PASS
        auth_pass fana123
    }
    unicast_peer {
        192.168.10.12
        192.168.10.14
    }
    virtual_ipaddress {
        192.168.10.100/24
    }
    track_script {
        CheckK8sMaster
    }
}
EOF

#14机器上keepalived配置
cat <<EOF > /etc/keepalived/keepalived.conf
global_defs {
   router_id LVS_k8s
}

vrrp_script CheckK8sMaster {
    script "curl -k https://192.168.10.100:6443"
    interval 3
    timeout 9
    fall 2
    rise 2
}

vrrp_instance VI_1 {
    state BACKUP
    interface ens33
    virtual_router_id 100
    priority 80
    advert_int 1
    mcast_src_ip 192.168.10.14
    nopreempt
    authentication {
        auth_type PASS
        auth_pass fana123
    }
    unicast_peer {
        192.168.10.12
        192.168.10.13
    }
    virtual_ipaddress {
        192.168.10.100/24
    }
    track_script {
        CheckK8sMaster
    }

}
EOF

#启动keepalived
systemctl restart keepalived && systemctl enable keepalived
```

### 1.3、配置haproxy

```bash
yum install -y haproxy

#13机器上配置
cat << EOF > /etc/haproxy/haproxy.cfg
global
    log         127.0.0.1 local2
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon

defaults
    mode                    tcp
    log                     global
    retries                 3
    timeout connect         10s
    timeout client          1m
    timeout server          1m

frontend kubernetes
    bind *:6443
    mode tcp
    default_backend kubernetes-master

backend kubernetes-master
    balance roundrobin
    server M-kube12 192.168.10.12:6443 check maxconn 2000
    server M-kube13 192.168.10.13:6443 check maxconn 2000
    server M-kube14 192.168.10.14:6443 check maxconn 2000
EOF

#12,13，和 14机器上配置都一样

# 启动haproxy
systemctl enable haproxy && systemctl start haproxy
```

**也可以用容器的方式部署**

```bash
# haproxy启动脚本
mkdir -p /data/lb
cat > /data/lb/start-haproxy.sh << "EOF"
#!/bin/bash
MasterIP1=192.168.10.12
MasterIP2=192.168.10.13
MasterIP3=192.168.10.14
MasterPort=6443

docker run -d --restart=always --name HAProxy-K8S -p 6444:6444 \
        -e MasterIP1=$MasterIP1 \
        -e MasterIP2=$MasterIP2 \
        -e MasterIP3=$MasterIP3 \
        -e MasterPort=$MasterPort \
        wise2c/haproxy-k8s
EOF

#keepalived启动脚本
cat > /data/lb/start-keepalived.sh << "EOF"
#!/bin/bash
VIRTUAL_IP=192.168.10.100
INTERFACE=ens33
NETMASK_BIT=24
CHECK_PORT=6444
RID=10
VRID=160
MCAST_GROUP=224.0.0.18

docker run -itd --restart=always --name=Keepalived-K8S \
        --net=host --cap-add=NET_ADMIN \
        -e VIRTUAL_IP=$VIRTUAL_IP \
        -e INTERFACE=$INTERFACE \
        -e CHECK_PORT=$CHECK_PORT \
        -e RID=$RID \
        -e VRID=$VRID \
        -e NETMASK_BIT=$NETMASK_BIT \
        -e MCAST_GROUP=$MCAST_GROUP \
        wise2c/keepalived-k8s
EOF

#把脚本拷贝到13和14机器上，然后启动
sh /data/lb/start-haproxy.sh && sh /data/lb/start-keepalived.sh

docker ps #可以看到容器的启动状态，相关配置文件可以进入容器查看
```

### 1.4、配置etcd

**14.1、在10.12机器上配置etcd证书**

```bash
#下载cfssl包
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
#设置cfssl环境
chmod +x cfssl*
mv cfssl_linux-amd64 /usr/local/bin/cfssl
mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
mv cfssl-certinfo_linux-amd64 /usr/local/bin/cfssl-certinfo
export PATH=/usr/local/bin:$PATH

#配置CA文件（IP地址为etc节点的IP）
mkdir /root/ssl
cd /root/ssl

cat >  ca-config.json <<EOF
{
"signing": {
"default": {
  "expiry": "8760h"
},
"profiles": {
  "kubernetes-Soulmate": {
    "usages": [
        "signing",
        "key encipherment",
        "server auth",
        "client auth"
    ],
    "expiry": "8760h"
  }
}
}
}
EOF

#--------------------------------------------------------#

cat >  ca-csr.json <<EOF
{
"CN": "kubernetes-Soulmate",
"key": {
"algo": "rsa",
"size": 2048
},
"names": [
{
  "C": "CN",
  "ST": "shanghai",
  "L": "shanghai",
  "O": "k8s",
  "OU": "System"
}
]
}
EOF

#--------------------------------------------------------#

cat > etcd-csr.json <<EOF
{
  "CN": "etcd",
  "hosts": [
    "127.0.0.1",
    "192.168.10.12",
    "192.168.10.13",
    "192.168.10.14"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "shanghai",
      "L": "shanghai",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF

#--------------------------------------------------------#
cfssl gencert -initca ca-csr.json | cfssljson -bare ca

cfssl gencert -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes-Soulmate etcd-csr.json | cfssljson -bare etcd
  
#将10.13的etcd证书分发到14,15机器上

mkdir -p /etc/etcd/ssl
cp *.pem /etc/etcd/ssl/

ssh -n 192.168.10.13 "mkdir -p /etc/etcd/ssl && exit"
ssh -n 192.168.10.14 "mkdir -p /etc/etcd/ssl && exit"

scp -r /etc/etcd/ssl/*.pem 192.168.10.13:/etc/etcd/ssl/
scp -r /etc/etcd/ssl/*.pem 192.168.10.14:/etc/etcd/ssl/
```

**1.4.2、在3台主节点上操作，安装etcd**

```bash
yum install etcd -y
mkdir -p /var/lib/etcd
```

```bash
#10.12机器上操作
cat <<EOF >/etc/systemd/system/etcd.service
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/coreos

[Service]
Type=notify
WorkingDirectory=/var/lib/etcd/
ExecStart=/usr/bin/etcd \
  --name M-kube12 \
  --cert-file=/etc/etcd/ssl/etcd.pem \
  --key-file=/etc/etcd/ssl/etcd-key.pem \
  --trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --peer-cert-file=/etc/etcd/ssl/etcd.pem \
  --peer-key-file=/etc/etcd/ssl/etcd-key.pem \
  --peer-trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --initial-advertise-peer-urls https://192.168.10.12:2380 \
  --listen-peer-urls https://192.168.10.12:2380 \
  --listen-client-urls https://192.168.10.12:2379,http://127.0.0.1:2379 \
  --advertise-client-urls https://192.168.10.12:2379 \
  --initial-cluster-token etcd-cluster-0 \
  --initial-cluster M-kube12=https://192.168.10.12:2380,M-kube13=https://192.168.10.13:2380,M-kube14=https://192.168.10.14:2380 \
  --initial-cluster-state new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

```bash
#10.13上机器操作
cat <<EOF >/etc/systemd/system/etcd.service
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/coreos

[Service]
Type=notify
WorkingDirectory=/var/lib/etcd/
ExecStart=/usr/bin/etcd \
  --name M-kube13 \
  --cert-file=/etc/etcd/ssl/etcd.pem \
  --key-file=/etc/etcd/ssl/etcd-key.pem \
  --peer-cert-file=/etc/etcd/ssl/etcd.pem \
  --peer-key-file=/etc/etcd/ssl/etcd-key.pem \
  --trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --peer-trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --initial-advertise-peer-urls https://192.168.10.13:2380 \
  --listen-peer-urls https://192.168.10.13:2380 \
  --listen-client-urls https://192.168.10.13:2379,http://127.0.0.1:2379 \
  --advertise-client-urls https://192.168.10.13:2379 \
  --initial-cluster-token etcd-cluster-0 \
  --initial-cluster M-kube12=https://192.168.10.12:2380,M-kube13=https://192.168.10.13:2380,M-kube14=https://192.168.10.14:2380 \
  --initial-cluster-state new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

```bash
#10.15机器上操作
cat <<EOF >/etc/systemd/system/etcd.service
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/coreos

[Service]
Type=notify
WorkingDirectory=/var/lib/etcd/
ExecStart=/usr/bin/etcd \
  --name M-kube14 \
  --cert-file=/etc/etcd/ssl/etcd.pem \
  --key-file=/etc/etcd/ssl/etcd-key.pem \
  --peer-cert-file=/etc/etcd/ssl/etcd.pem \
  --peer-key-file=/etc/etcd/ssl/etcd-key.pem \
  --trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --peer-trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --initial-advertise-peer-urls https://192.168.10.14:2380 \
  --listen-peer-urls https://192.168.10.14:2380 \
  --listen-client-urls https://192.168.10.14:2379,http://127.0.0.1:2379 \
  --advertise-client-urls https://192.168.10.14:2379 \
  --initial-cluster-token etcd-cluster-0 \
  --initial-cluster M-kube12=https://192.168.10.12:2380,M-kube13=https://192.168.10.13:2380,M-kube14=https://192.168.10.14:2380 \
  --initial-cluster-state new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

```bash
#添加自启动
cp /etc/systemd/system/etcd.service /usr/lib/systemd/system/
systemctl daemon-reload && systemctl start etcd && systemctl enable etcd && systemctl status etcd
 
#在etc节点上检查
etcdctl --endpoints=https://192.168.10.12:2379,https://192.168.10.13:2379,https://192.168.10.14:2379 \
 --ca-file=/etc/etcd/ssl/ca.pem \
 --cert-file=/etc/etcd/ssl/etcd.pem \
 --key-file=/etc/etcd/ssl/etcd-key.pem  cluster-health

#正常的话会有如下提示
[root@M-kube13 ~]# etcdctl --endpoints=https://192.168.10.12:2379,https://192.168.10.13:2379,https://192.168.10.14:2379 \
>  --ca-file=/etc/etcd/ssl/ca.pem \
>  --cert-file=/etc/etcd/ssl/etcd.pem \
>  --key-file=/etc/etcd/ssl/etcd-key.pem  cluster-health
member 1af68d968c7e3f22 is healthy: got healthy result from https://192.168.10.12:2379
member 55204c19ed228077 is healthy: got healthy result from https://192.168.10.14:2379
member e8d9a97b17f26476 is healthy: got healthy result from https://192.168.10.13:2379
```

### 1.5、安装Docker

如今Docker分为了Docker-CE和Docker-EE两个版本，CE为社区版即免费版，EE为企业版即商业版。我们选择使用CE版。

在所有的机器上安装docker

**yum安装docker**

```bash
#1.安装yum源工具包
yum install -y yum-utils device-mapper-persistent-data lvm2

#2.下载docker-ce官方的yum源配置文件，上面操作了 这里就不操作了
# yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
# yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

#3.禁用docker-c-edge源配edge是不开发版，不稳定，下载stable版
yum-config-manager --disable docker-ce-edge
#4.更新本地YUM源缓存
yum makecache fast
#5.安装Docker-ce相应版本
yum -y install docker-ce
#6.设置开机自启动
systemctl restart docker && systemctl enable docker && systemctl status docker
```

**运行hello world验证**

```bash
[root@localhost ~]# docker run hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
9a0669468bf7: Pull complete
Digest: sha256:0e06ef5e1945a718b02a8c319e15bae44f47039005530bc617a5d071190ed3fc
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
1. The Docker client contacted the Docker daemon.
2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
3. The Docker daemon created a new container from that image which runs the
   executable that produces the output you are currently reading.
4. The Docker daemon streamed that output to the Docker client, which sent it
   to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
$ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
https://cloud.docker.com/

For more examples and ideas, visit:
https://docs.docker.com/engine/userguide/
```

### 1.6、安装kubelet与kubeadm包

**使用DaoCloud加速器(可以跳过这一步)**

```bash
curl -sSL https://get.daocloud.io/daotools/set_mirror.sh | sh -s http://0d236e3f.m.daocloud.io
# docker version >= 1.12
# {"registry-mirrors": ["http://0d236e3f.m.daocloud.io"]}
# Success.
# You need to restart docker to take effect: sudo systemctl restart docker
systemctl restart docker
```

**在所有机器安装kubectl kubelet kubeadm kubernetes-cni**

```bash
yum list kubectl kubelet kubeadm kubernetes-cni		#查看可安装的包
已加载插件：fastestmirror
Loading mirror speeds from cached hostfile
* base: mirrors.tuna.tsinghua.edu.cn
* extras: mirrors.sohu.com
* updates: mirrors.sohu.com
#显示可安装的软件包
kubeadm.x86_64                                    1.14.3-0                                              kubernetes
kubectl.x86_64                                    1.14.3-0                                             kubernetes
kubelet.x86_64                                    1.14.3-0                                              kubernetes
kubernetes-cni.x86_64                             0.7.5-0                                              kubernetes
[root@localhost ~]#

#然后安装kubectl kubelet kubeadm kubernetes-cni
yum install -y kubectl kubelet kubeadm kubernetes-cni

# Kubelet负责与其他节点集群通信，并进行本节点Pod和容器生命周期的管理。
# Kubeadm是Kubernetes的自动化部署工具，降低了部署难度，提高效率。
# Kubectl是Kubernetes集群管理工具
```

**修改kubelet配置文件**（可不操作）

```bash
vi /etc/systemd/system/kubelet.service.d/10-kubeadm.conf	#或者在如下目录可不操作
/usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf
# 修改一行
Environment="KUBELET_CGROUP_ARGS=--cgroup-driver=cgroupfs"
# 添加一行
Environment="KUBELET_EXTRA_ARGS=--v=2 --fail-swap-on=false --pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/k8sth/pause-amd64:3.0"
#重新加载配置
systemctl daemon-reload		


#1.命令补全
yum install -y bash-completion
source /usr/share/bash-completion/bash_completion
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc
#启动所有主机上的kubelet服务
systemctl enable kubelet && systemctl start kubelet		
```

### 1.7、初始化集群

**kubeadm init主要执行了以下操作：**

​	[init]：指定版本进行初始化操作
​	[preflight] ：初始化前的检查和下载所需要的Docker镜像文件
​	[kubelet-start]：生成kubelet的配置文件”/var/lib/kubelet/config.yaml”，没有这个文件kubelet无法启动，所以初始化之前的kubelet实际上启动失败。
​	[certificates]：生成Kubernetes使用的证书，存放在/etc/kubernetes/pki目录中。
​	[kubeconfig] ：生成 KubeConfig 文件，存放在/etc/kubernetes目录中，组件之间通信需要使用对应文件。
​	[control-plane]：使用/etc/kubernetes/manifest目录下的YAML文件，安装 Master 组件。
​	[etcd]：使用/etc/kubernetes/manifest/etcd.yaml安装Etcd服务。
​	[wait-control-plane]：等待control-plan部署的Master组件启动。
​	[apiclient]：检查Master组件服务状态。
​	[uploadconfig]：更新配置
​	[kubelet]：使用configMap配置kubelet。
​	[patchnode]：更新CNI信息到Node上，通过注释的方式记录。
​	[mark-control-plane]：为当前节点打标签，打了角色Master，和不可调度标签，这样默认就不会使用Master节点来运行Pod。
​	[bootstrap-token]：生成token记录下来，后边使用kubeadm join往集群中添加节点时会用到
​	[addons]：安装附加组件CoreDNS和kube-proxy

**1.7.1、在10.12 机器上添加集群初始化配置文件**

```bash
kubeadm config print init-defaults > kubeadm-config.yaml	#这个命令可以生成初始化配置文件也可以自己写

# 1.创建初始化集群配置文件，
cat <<EOF > /etc/kubernetes/kubeadm-master.config
apiVersion: kubeadm.k8s.io/v1beta1
kind: ClusterConfiguration
kubernetesVersion: v1.14.3
controlPlaneEndpoint: "192.168.10.100:6443"
imageRepository: registry.aliyuncs.com/google_containers  
apiServer:
  certSANs:
  - 192.168.10.12
  - 192.168.10.13
  - 192.168.10.14
  - 192.168.10.100
etcd:
  external:
    endpoints:
    - https://192.168.10.12:2379
    - https://192.168.10.13:2379
    - https://192.168.10.14:2379
    caFile: /etc/etcd/ssl/ca.pem
    certFile: /etc/etcd/ssl/etcd.pem
    keyFile: /etc/etcd/ssl/etcd-key.pem
networking:
  podSubnet: 10.244.0.0/16
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: ipvs
EOF

#2.然后执行
kubeadm config images pull --config kubeadm-master.config	#可以先执行这个提前下载镜像
kubeadm init --config kubeadm-master.config --experimental-upload-certs | tee kubeadm-init.log
# 追加tee命令可以将初始化日志输出到kubeadm-init.log中，添加--experimental-upload-certs参数可以在后续执行加入节点时自动分发证书文件。

#3.初始化失败后处理方法
kubeadm reset		#初始化失败或者成功，都可以直接执行kubeadm reset命令清理集群或节点
#或
rm -rf /etc/kubernetes/*.conf
rm -rf /etc/kubernetes/manifests/*.yaml
docker ps -a |awk '{print $1}' |xargs docker rm -f
systemctl  stop kubelet

#初始化正常的结果如下
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join 192.168.10.100:6443 --token y6v90q.i6bl1bwcgg8clvh5 \
    --discovery-token-ca-cert-hash sha256:179c5689ef32be2123c9f02015ef25176d177c54322500665f1170f26368ae3d \
    --experimental-control-plane --certificate-key 3044cb04c999706795b28c1d3dcd2305dcf181787d7c6537284341a985395c20

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use 
"kubeadm init phase upload-certs --experimental-upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.10.100:6443 --token y6v90q.i6bl1bwcgg8clvh5 \
    --discovery-token-ca-cert-hash sha256:179c5689ef32be2123c9f02015ef25176d177c54322500665f1170f26368ae3d 
    
#5.然后拷贝文件
mkdir -p /root/.kube
cp -i /etc/kubernetes/admin.conf /root/.kube/config
chown $(id -u):$(id -g) /root/.kube/config		#如果是其他用户需要使用kubectl命令，需要拷贝到$HOME目录，然后赋权

```

**1.7.2、查看当前状态**

```bash
[root@M-kube12 kubernetes]# kubectl get node
NAME       STATUS     ROLES    AGE     VERSION
m-kube12   NotReady   master   3m40s   v1.14.3		# STATUS显示的状态还是不可用

[root@M-kube12 kubernetes]# kubectl -n kube-system get pod
NAME                               READY   STATUS    RESTARTS   AGE
coredns-8686dcc4fd-fmlsh           0/1     Pending   0          3m40s
coredns-8686dcc4fd-m22j7           0/1     Pending   0          3m40s
etcd-m-kube12                      1/1     Running   0          2m59s
kube-apiserver-m-kube12            1/1     Running   0          2m53s
kube-controller-manager-m-kube12   1/1     Running   0          2m33s
kube-proxy-4kg8d                   1/1     Running   0          3m40s
kube-scheduler-m-kube12            1/1     Running   0          2m45s

[root@M-kube12 kubernetes]# kubectl get cs
NAME                 STATUS    MESSAGE             ERROR
controller-manager   Healthy   ok                  
scheduler            Healthy   ok                  
etcd-0               Healthy   {"health":"true"} 
```

**1.7.3、部署flannel网络，在12机器上执行**

```bash
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
#版本信息：quay.io/coreos/flannel:v0.10.0-amd64

cat kube-flannel.yml | grep image
cat kube-flannel.yml | grep 10.244
sed -i 's#quay.io/coreos/flannel:v0.11.0-amd64#willdockerhub/flannel:v0.11.0-amd64#g' kube-flannel.yml
kubectl apply -f kube-flannel.yml
#或者
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/a70459be0084506e4ec919aa1c114638878db11b/Documentation/kube-flannel.yml


#等待一会 查看 node和pod 状态全部为Running
[root@M-fana3 kubernetes]# kubectl get node              
NAME      STATUS   ROLES    AGE   VERSION
m-fana3   Ready    master   42m   v1.14.3		#状态正常了
[root@M-fana3 kubernetes]# kubectl -n kube-system get pod
NAME                              READY   STATUS    RESTARTS   AGE
coredns-8686dcc4fd-2z6m2          1/1     Running   0          42m
coredns-8686dcc4fd-4k7mm          1/1     Running   0          42m
etcd-m-fana3                      1/1     Running   0          41m
kube-apiserver-m-fana3            1/1     Running   0          41m
kube-controller-manager-m-fana3   1/1     Running   0          41m
kube-flannel-ds-amd64-6zrzt       1/1     Running   0          109s
kube-proxy-lc8d5                  1/1     Running   0          42m
kube-scheduler-m-fana3            1/1     Running   0          41m

#如果遇到问题想如下情况，有可能镜像拉取失败了，
kubectl -n kube-system get pod                                          
NAME                               READY   STATUS                  RESTARTS   AGE
coredns-8686dcc4fd-c9mw7           0/1     Pending                 0          43m
coredns-8686dcc4fd-l8fpm           0/1     Pending                 0          43m
kube-apiserver-m-kube12            1/1     Running                 0          42m
kube-controller-manager-m-kube12   1/1     Running                 0          17m
kube-flannel-ds-amd64-gcmmp        0/1     Init:ImagePullBackOff   0          11m
kube-proxy-czzk7                   1/1     Running                 0          43m
kube-scheduler-m-kube12            1/1     Running                 0          42m

#可以通过 kubectl describe pod kube-flannel-ds-amd64-gcmmp --namespace=kube-system 查看pod状态，看到最后报错如下，可以手动下载或者二进制安装
Node-Selectors:  beta.kubernetes.io/arch=amd64
Tolerations:     :NoSchedule
                 node.kubernetes.io/disk-pressure:NoSchedule
                 node.kubernetes.io/memory-pressure:NoSchedule
                 node.kubernetes.io/network-unavailable:NoSchedule
                 node.kubernetes.io/not-ready:NoExecute
                 node.kubernetes.io/pid-pressure:NoSchedule
                 node.kubernetes.io/unreachable:NoExecute
                 node.kubernetes.io/unschedulable:NoSchedule
Events:
  Type     Reason          Age                    From               Message
  ----     ------          ----                   ----               -------
  Normal   Scheduled       11m                    default-scheduler  Successfully assigned kube-system/kube-flannel-ds-amd64-gcmmp to m-kube12
  Normal   Pulling         11m                    kubelet, m-kube12  Pulling image "willdockerhub/flannel:v0.11.0-amd64"
  Warning  FailedMount     7m27s                  kubelet, m-kube12  MountVolume.SetUp failed for volume "flannel-token-6g9n7" : couldn't propagate object cache: timed out waiting for the condition
  Warning  FailedMount     7m27s                  kubelet, m-kube12  MountVolume.SetUp failed for volume "flannel-cfg" : couldn't propagate object cache: timed out waiting for the condition
  Warning  Failed          4m21s                  kubelet, m-kube12  Failed to pull image "willdockerhub/flannel:v0.11.0-amd64": rpc error: code = Unknown desc = context canceled
  Warning  Failed          3m53s                  kubelet, m-kube12  Failed to pull image "willdockerhub/flannel:v0.11.0-amd64": rpc error: code = Unknown desc = Error response from daemon: Get https://registry-1.docker.io/v2/: net/http: request canceled (Client.Timeout exceeded while awaiting headers)
  Warning  Failed          3m16s                  kubelet, m-kube12  Failed to pull image "willdockerhub/flannel:v0.11.0-amd64": rpc error: code = Unknown desc = Error response from daemon: Get https://registry-1.docker.io/v2/: net/http: TLS handshake timeout
  Warning  Failed          3m16s (x3 over 4m21s)  kubelet, m-kube12  Error: ErrImagePull
  Normal   SandboxChanged  3m14s                  kubelet, m-kube12  Pod sandbox changed, it will be killed and re-created.
  Normal   BackOff         2m47s (x6 over 4m21s)  kubelet, m-kube12  Back-off pulling image "willdockerhub/flannel:v0.11.0-amd64"
  Warning  Failed          2m47s (x6 over 4m21s)  kubelet, m-kube12  Error: ImagePullBackOff
  Normal   Pulling         2m33s (x4 over 7m26s)  kubelet, m-kube12  Pulling image "willdockerhub/flannel:v0.11.0-amd64"


```

**在13和14机器上操作加入集群**

```bash
#执行加入集群命令
kubeadm join 192.168.10.100:6443 --token y6v90q.i6bl1bwcgg8clvh5 \
    --discovery-token-ca-cert-hash sha256:179c5689ef32be2123c9f02015ef25176d177c54322500665f1170f26368ae3d \
    --experimental-control-plane --certificate-key 3044cb04c999706795b28c1d3dcd2305dcf181787d7c6537284341a985395c20
# 拷贝kube到用户目录
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# 验证机器状态
kubectl -n kube-system get pod -o wide	#查看pod运行情况

kubectl get nodes -o wide #查看节点情况

kubectl -n kube-system get svc	#查看service
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   16m
```

11













这个一定要记住,以后无法重现，添加节点需要

```bash
# 配置kubectl工具
mkdir -p /root/.kube
cp /etc/kubernetes/admin.conf /root/.kube/config	#如果是其他用户也要拷贝到对应的用户目录下然后赋权
kubectl get nodes
kubectl get cs

#部署flannel网络
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/a70459be0084506e4ec919aa1c114638878db11b/Documentation/kube-flannel.yml
```

**6、添加node节点**

```bash
#在node节点的机器上执行操作
kubeadm join 192.168.10.13:6443 --token wdmykh.u84g6ijzu4n99qez --discovery-token-ca-cert-hash sha256:868b5d27c078ddd3ce98bf67bbad4d8568d3cb134763b732e7ad4b47eda196b2
#提示如下
[kubeadm] WARNING: kubeadm is in beta, please do not use it for production clusters.
[preflight] Running pre-flight checks
[preflight] WARNING: docker version is greater than the most recently validated version. Docker version: 17.09.0-ce. Max validated version: 1.12
[preflight] WARNING: kubelet service is not enabled, please run 'systemctl enable kubelet.service'
[preflight] Starting the kubelet service
[discovery] Trying to connect to API Server "192.168.10.13:6443"
[discovery] Created cluster-info discovery client, requesting info from "https://192.168.10.13:6443"
[discovery] Cluster info signature and contents are valid, will use API Server "https://192.168.10.13:6443"
[discovery] Successfully established connection with API Server "192.168.10.13:6443"
[bootstrap] Detected server version: v1.14.3
[bootstrap] The server supports the Certificates API (certificates.k8s.io/v1beta1)
[csr] Created API client to obtain unique certificate for this node, generating keys and certificate signing request
[csr] Received signed certificate from the API server, generating KubeConfig...
[kubeconfig] Wrote KubeConfig file to disk: "/etc/kubernetes/kubelet.conf"

Node join complete:
* Certificate signing request sent to master and response
 received.
* Kubelet informed of new secure connection details.

Run 'kubectl get nodes' on the master to see this machine join.
```

**7、查看集群验证**

```bash
kubectl get cs		#在master节点上操作
NAME                 STATUS    MESSAGE              ERROR
scheduler            Healthy   ok
controller-manager   Healthy   ok
etcd-0               Healthy   {"health": "true"}
kubectl get nodes	#查看node节点
NAME      STATUS     AGE       VERSION
master    Ready      24m       v1.14.3
node1     NotReady   45s       v1.14.3
node2     NotReady   7s        v1.14.3
kubectl get pods --all-namespaces		#查看pod
NAMESPACE     NAME                             READY     STATUS              RESTARTS   AGE
kube-system   etcd-master                      1/1       Running             0          24m
kube-system   kube-apiserver-master            1/1       Running             0          24m
kube-system   kube-controller-manager-master   1/1       Running             0          24m
kube-system   kube-dns-2425271678-h48rw        0/3       ImagePullBackOff    0          25m
kube-system   kube-flannel-ds-28n3w            1/2       CrashLoopBackOff    13         24m
kube-system   kube-flannel-ds-ndspr            0/2       ContainerCreating   0          41s
kube-system   kube-flannel-ds-zvx9j            0/2       ContainerCreating   0          1m
kube-system   kube-proxy-qxxzr                 0/1       ImagePullBackOff    0          41s
kube-system   kube-proxy-shkmx                 0/1       ImagePullBackOff    0          25m
kube-system   kube-proxy-vtk52                 0/1       ContainerCreating   0          1m
kube-system   kube-scheduler-master            1/1       Running             0          24m

#创建pod验证集群是否正常
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80 --type=NodePort
kubectl get pod,svc
```

如果出现：The connection to the server localhost:8080 was refused - did you specify the right host or port?

解决办法： 为了使用kubectl访问apiserver，在~/.bash_profile中追加下面的环境变量： export KUBECONFIG=/etc/kubernetes/admin.conf source ~/.bash_profile 重新初始化kubectl

**8、部署dashboard**

```bash
#1.创建yaml文件
wget https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml

sed -i 's/k8s.gcr.io/loveone/g' kubernetes-dashboard.yaml
sed -i "160a \ \ \ \ \ \ nodePort: 30001" kubernetes-dashboard.yaml
sed -i "161a \ \ type:\ NodePort" kubernetes-dashboard.yaml

#2.创建dashboard
kubectl create -f kubernetes-dashboard.yaml

#3.检查相关服务状态
kubectl get deployment kubernetes-dashboard -n kube-system
kubectl get pods -n kube-system -o wide
kubectl get services -n kube-system
netstat -ntlp|grep 30001

#4.查看访问dashboard的认证令牌
kubectl describe secrets -n kube-system $(kubectl -n kube-system get secret | awk '/dashboard-admin/{print $1}')
```





## 2、以二进制文件方式安装kubernetes集群

一般生产集群版搭建最少3台master，多台node，以下练习环境 总共4台机器：[参考](<https://www.jianshu.com/p/832bcd89bc07>)

练习环境说明：

master:192.168.10.15 192.168.10.16

node:192.168.10.13 192.168.14

[参考1](https://www.cnblogs.com/harlanzhang/p/10131264.html) [参考2](https://blog.51cto.com/13941177/2316025) 

在master节点上需要部署，etcd 	kube-apiserver	kube-controller-manager	kuber-scheduler

在 Node 节点上需要部署， etcd	docker		kubelet		kube-proxy

### 2.1、下载安装包

[GitHub网址](<https://github.com/kubernetes/kubernetes/releases>)  <https://github.com/kubernetes/kubernetes/releases>  

下载Server Binaries 中的 [kubernetes-server-linux-amd64.tar.gz](https://dl.k8s.io/v1.14.3/kubernetes-server-linux-amd64.tar.gz) 安装包，上传到master服务器上 并解压到/usr/bin目录

下载Node Binaries中的 [kubernetes-node-linux-amd64.tar.gz](https://dl.k8s.io/v1.14.3/kubernetes-node-linux-amd64.tar.gz) 安装包，上传到node服务器上 并解压到/usr/bin目录

下载 Cl Binares

下载 etcd安装包 GitHub地址：<https://github.com/etcd-io/etcd/releases> 上传到master服务器上，解压到/usr/bin目录



### 2.2、搭建服务

**1、etcd服务**  ，将文件解压到/usr/bin目录



**2、kube-apiserver服务** 



**3、kube-controller-manager服务**



**4、kube-scheduler服务**



至此master机器上就搭建完成了，下面搭建的都是node上的服务，node需要先安装docker并启动

**5、kubelet服务** 



**6、kube-proxy服务**

















---

## 2、kubectl常用命令

kubectl 命令的语法：kubectl	[command]	[type]	[name]	[flags]	含义如下

command：子命令，用于操作kubernetes集群资源对象的命令，如：create delete describe get apply

type：资源对象的类型，区分大小写，能以单数，复数形式或者简写的形式表示

name：资源对象的名称，区分大小写，如果不指定名称，则系统返回属于type的全部列表，如 kubecel get pods 返回的是所有pod列表

flags：子命令的可选参数，

**可操作的资源对象类型**

| 资源对象的名称 | 缩写 |
| -------------- | ---- |
| clusters       |      |
|                |      |
|                |      |
|                |      |
|                |      |
|                |      |
|                |      |
|                |      |
|                |      |
|                |      |
|                |      |
|                |      |
|                |      |
|                |      |
|                |      |
|                |      |
|                |      |
|                |      |
|                |      |
|                |      |
|                |      |
|                |      |
|                |      |
|                |      |
|                |      |
|                |      |
|                |      |
|                |      |
|                |      |



```bash
kubectel get services	#查看创建的容器服务
kubectl get nodes	#查看集群中有多少node
kubectl describe node (node_name)	#查看某个node的详细信息
kubectl describe node 192.168.0.212	# 显示 Node 的详细信息
kubectl get endpoints		#查看endpoint列表(会显示pod名和访问的IP地址)

kubectl get namespaces		#查看命名空间

kubectl get pods	#查看pod实例
kubectl get pod
kubectl get pod -n kube 	# 查看所有 pod 列表,  -n 后跟 namespace, 查看指定的命名空间
kubectl delete pod --all	# 删除所有 Pod
kubectl delete pod,svc -l name=<label-name>		# 删除所有包含某个 label 的pod 和 service

# 执行 pod 的 date 命令
kubectl exec <pod-name> -- date
kubectl exec <pod-name> -- bash
kubectl exec <pod-name> -- ping 10.24.51.9
# 通过bash获得 pod 中某个容器的TTY，相当于登录容器
kubectl exec -it <pod-name> -c <container-name> -- bash
eg:
kubectl exec -it redis-master-cln81 -- bash

# 显示 Pod 的详细信息, 特别是查看 pod 无法创建的时候的日志
kubectl describe pod <pod-name>
eg:
kubectl describe pod redis-master-tqds9


# 根据 yaml 创建资源, apply 可以重复执行，create 不行
kubectl create -f pod.yaml
kubectl apply -f pod.yaml
kubectl create -f myweb-rc.yaml	#通过RC文件，创建tomcat容器

# 基于 pod.yaml 定义的名称删除 pod 
kubectl delete -f pod.yaml 


# 查看 RC 和 service 列表， -o wide 查看详细信息
kubectl get rc,svc
kubectl get pod,svc -o wide  
kubectl get pod <pod-name> -o yaml
kubectl get svc <pod-name> -o yaml	#查看微服务分配的 cluster ip


# 查看容器的日志
kubectl logs <pod-name>
kubectl logs -f <pod-name> # 实时查看日志
```

**tomcat应用yml文件**

```yaml
apiVersion:v1
kind:ReplicationController
metadata:
  name:myweb
spec:
  replicas:2
  selector:
    app:myweb
  template:
    metadata:
      labels:
        app:myweb
    spec:
      containers:
      - name:myweb
        image:kubeguide/tomcat-app:v1
        ports:
        - containerPort:8080
```



## 3、kubernetes基本概念

kubernetes中 node pod service等都可以看作一种资源对象，几乎所有的资源对象都可以通过kubectl工具执行增删改查等操作。并将其保存在etcd中持久化存储。

**kubernetes集群的二种管理角色：master和node**

**==Master 	是指集群的控制节点==**  每个集群都需要一个master节点来负责整个集群的管理和控制，基本上kubernetes的所有控制命令都会发给他，他来负责具体的执行过程，一般执行命令都是在master节点上执行的，master节点要占用一个独立的服务器(高可用部署建议用3台机器)。如果master宕机或者不可用，集群内的容器都会失效。

**Master节点上运行着以下关键进程：**

kubernetes API server :(kube-apiserver) 提供了http rest接口的关键服务进程，是kubernetes里所有资源的增删改查等操作的唯一接口。

kubernetes Controller Manager: (kube-controller-manager) 所有资源对象的自动化控制中心，可以理解为资源对象的 大总管。

Kubernetes Scheduler :(kube-scheduler) 负责资源调度（pod调度）的进程，相当于公交公司的调度室。

master节点上还需要启动一个etcd服务，所有的资源对象的数据都报错在etcd中。

**==Node  集群中的机器都成为node节点==** 是kubernetes集群中的工作负载节点。每个node都会被master分配一些docker容器，当某个node宕机后，其他的docker容器会迁移到其他node节点上。

**Node节点上运行着以下关键进程**

kubelet:负责pod的创建，启动 暂停等任务。

kube-proxy：实现与kubernetes service的通讯，和负载均衡机制的重要组件。

docker engine: docker引擎，负责本机的容器创建和管理工作。

**==Pod==** 每个pod都有一个特殊的pause容器和一个或多个相关的用户业务容器。pod有二种类型：普通的pod和静态pod（比较特殊，他不存放在etcd存储里，而是放在某个node上的一个具体文件中，并且只在此node上运行。）普通的pod一旦被创建就会放在etcd中存储，随后会被Master调度到某个的node上进行绑定，然后该pod被kubelet进程实例化成一组相关的docker容器，并启动起来。

**==Label==** 是一个用户自定义的key=value的键值对，可以使用到各种资源对象上，（node,pod,service,RC等）

**==Replication Controller==** 简称RC，定义了一个期望的场景，即声明某种pod的副本数量在任意时刻都符合某个预期值，s所有RC的定义包括以下几个部分

pod的数量（replicas），用于筛选目标pod的label selector , 当pod的副本数量小于预期数量时，用于创建新pod的pod模板。

下面是一个完整的RC定义例子,

```dockerfile
apiVersion:v1
kind:ReplicationController
metadata:
  name:frontend
spec:
  replicas:1
  selector:
    tier:frontend
  template:
    metadata:
      labels:
        app:app-demo
        tier:frontend
    spec:
      containers:
      - name:tomcat-demo
        image:tomcat
        imagePullPolicy:IfNotPresent
        env:
        - name:GET_HOSTS_FROM
          value:dns
        ports:
        - containerPort:80
```

当我们吧RC提交到kubernetes集群后，master上的Controller Manager会定期巡检当前存活的pod，并保障pod实例数等于RC的期望值。在运行时我们可以通过修RC，来实现对pod实例数的修改 kubectl scale rc redis-slave --replicas=3 ，

新版的kubernetes出现了replica set,可以称为下一次的RC，和Replication Controller区别是支持基于集合的Lable selector，而Replication Controller只支持等式的Lable Selector，但他主要被Deployment这个资源对象所使用，从而形成一套pod创建，删除 更新的编排机制。

**==Deployment==** 相当于RC的一次升级，为了更好的解决pod的编排问题。在内部是使用了replica set来实现的。他的定义和Replica Set的定义很类似。示例如下

```dockerfile
apiVersion: extensions/vlbetal
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 1
  selector:
  matchLabels:
    tier: frontend
  matchExpressions:
    - {key: tier, operator: In, values: [frontend]}
template:
  metadata:
    labels:
      app: app-demo
      tier: frontend
  spec:
    containers:
    - name: tomcat-demo
      image: tomcat
      imagePullPolicy: IfNotPresent
      ports:
      - containerPort: 8080
      
# 运行下面命令创建Deployment
kubectl create -f tomcat-deployment.yaml
kubectl get deployments		#查看deployment的信息
#显示内容解释：DESIRED:pod数量的期望值，CURRENT:当前replica的值，UP-TO-DATA：最新版本的pod数量，AVAILBLE:集群中存活的pod数量
kubectl get rs	#查看replica set信息
kubectl describe deployments	#可以看到deployment控制pod的水平扩展过程。
```

**==Horizontal Pod Autoscaler==** pod只能扩容简称HPA，通过追踪分析RC控制的所有pod的负载变化情况，来确定是否需要针对性的调整目标的pod数量。有二种方式作为pod负载的度量指标，CPUUtilizationPercentage和应用程序自定义的度量指标，比如服务在每秒的请求数(TPS或QPS)。

```bash
kubectl autoscale deployment php-apache --cpu-percent=90 --min=1 --max=10	#除了yaml文件定义外，通过命令创建HPA资源对象
```

**==StatefulSet==** 在kubernetes里，pod的管理对象RC、Deployment、DaemonSet和job都是面向无状态的服务，但现实中有很多服务是有状态的。staefulset可以看作是Deployment/rc的一个特殊变种，有以下特性：1、每个pod都有稳定和唯一的网络标识，可以发现集群内的其他成员/2、pod的启停是可以控制的，3、采用了稳定的持久化存储卷，删除pod不会删除与statefulset相关的存储卷。

**==service==** 就是指我们的微服务，

kubernetes里有三种IP。

Node IP : node节点的IP地址，集群中每个节点的物理网卡的IP地址。

Pod IP: pod的ip地址 ，他是docker0网桥分配的IP，是一个虚拟的二层网络，真实的流量都是通过 node IP的物理网卡流出的。

Cluster IP :service的IP地址，也是一个虚拟的IP，属于kubernetes集群内部的地址，无法再集群直接使用。仅仅作为kubernetes service 这个对象，无法ping，不具备TCP/IP通信的基础，只属于kubernetes集群这样的一个封闭空间，如果需要和Cluster IP 网通信，一般是采用NodePort，因为Cluster IP是kubernetes自己设计的路由规则。

**==Volume==** 是pod中能够被多个容器访问的共享目录，用途和目的与docker的容器卷类似，单不能等价。kubernetes提供了很多Volume类型

1、emptyDir	是pod分配到node时创建的，当pod从node上移除时，emptyDir中的数据也会被永久被删除

2、hostPath	是pod在挂载宿主机上的文件或目录，通常用于，容器的应用日志需要保存时，可以使用宿主机存储，需要访问宿主机上docker引擎内部数据结构的容器应用时，可以定义hostPath为宿主机/var/lib/docker目录，使容器内部可以直接访问docker的文件系统。

3、gcePersistentDisk	这样的类型表示使用谷歌公有云提供的永久磁盘。

4、awsElasticBlockStore	表示使用了亚马逊提供的 EBS Volume存储数据

5、NFS	使用NFS网络文件系统提供的共享目录存储数据

6、其他如：iscsi		flocker		rbd		gitRepo

**==Persistent Volume==** 可以理解成 kubernetes集群中的某个网络存储对应的一块存储。与Volume是有区别的，PV是网络存储，不属于NOde，但可以在每个node上访问，他不是定义在pod上的，而是独立于pod之外，

**==namespace==**(命名空间) 是非常重要的感念，多数情况下用于实现多租户的资源隔离，默认kubernetes启动后，会创建一个名为default的命名空间。

**==Annotation==**(注解) 与label用法类似，不同的是label是具有严格的命名规则，而Annotation则是用户任意定义的附加信息。











# 第四章 kubernetes高阶实践































