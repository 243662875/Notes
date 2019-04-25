[TOC]

# 第一章 docker的基础

## 1、安装docker

``` shell
yum install -y yum-utils device-mapper-persistent-data lvm2		#安装依赖包
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo	#添加阿里yum源
yum makecache fast	#更新yum缓存
yum -y install docker-ce	#安装docker
systemctl start docker		#启动docker
docker run -itd -p 80:80 --name webserver nginx  /bin/bash   #启动一个容器,先从本地仓库查找镜像,如果没有再从官网上查找
```

**卸载docker**

``` bash
yum remove docker-ce		#卸载docker
rm -rf /var/lib/docker		#删除docker目录
```

## 2、使用镜像

``` dockerfile
docker pull ubnutu			#从互联网上下载一个镜像到本地仓库,默认使用latest这个标签
docker push  注册用户名/镜像名		#推送镜像到远程镜像仓库
docker search php				#默认从官网查找php的镜像
docker images					#列出本地镜像
docker inspect nginx:latest          #获取docker镜像的详细信息
docker inspect -f '{{.NetworkSettings.IPAddress}}' mynginx           #获取在运行的mynginx容器IP
docker rmi ubuntu:latest             #删除本地仓库的镜像，如果正在使用可加上 -f 强制删除但不建议使用
Docker rmi $(docker images -a|grep none|awk '{print $3}')  删除没有打tag的镜像
```

## 3、镜像迁移

```dockerfile
docker save nginx:latest > nginx.tar.gz		#将镜像导出来，就是指保存到本地
docker save -o nginx.03.tar nginx:latest	#或者使用这个命令
#-----------------------------------------------------------------------------
docker load < nginx.tar.gz		#将导出的镜像重新导入镜像库
docker load -i nginx.tar.gz		#或者使用这个命令
```

## 4、docker Hub介绍

官方提供的镜像仓库网址：https://hub.docker.com/

``` docker login -u user -p password server_url		#登陆docker Hub ```

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

 需要基于docker环境

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

## 6、使用容器

```shell
docker create debian:jessie		#创建容器但不启动
docker run -it debian:jessie /bin/bash		#运行容器,通过bash进入debian系统,退出容器后会关闭. 
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
docker restart start stop  镜像名或容器ID		#重启 启动 关闭 容器
docker rm 容器id		#删除容器
docker kill		#发送信号给容器默认SIGKILL例如：docker kill -s KILL mycon (-s表示向容器发送一个信号)
docker wai		#阻塞到一个容器，直到容器停止运行。例如：docker wait mycon
docker pause		#暂停容器中所有的进程。 例如：docker pause mycon
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
```

## 7、容器保存和迁移

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

## 9、网络访问

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

有时候，一个容器运行的应用程序，也需要和另一个容器的应用程序进行数据交换，此时，就需要通过容器连接来完成2个容器之间的数据交换。要设置容器间通信，我们可以通过在创建容器时加上 --link 参数来实现。







