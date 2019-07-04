[TOC]





# 第三章 kubernetes环境搭建

以centos 7 为例，搭建kubernetes集群，建议先关闭防火墙和SElinux，最简单的安装方法就是yum安装

[参考1](https://blog.csdn.net/networken/article/details/89599004)        [参考2](https://www.cnblogs.com/hongdada/p/9761336.html)

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

练习环境说明：[参考1](https://www.kubernetes.org.cn/3808.html?tdsourcetag=s_pctim_aiomsg)      [参考2](https://www.kubernetes.org.cn/5462.html)  

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
    script "curl -k https://192.168.10.100:6444"
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
    script "curl -k https://192.168.10.100:6444"
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
    script "curl -k https://192.168.10.100:6444"
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
    bind *:6444
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
mkdir /root/ssl && cd /root/ssl

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

mkdir -p /etc/etcd/ssl && cp *.pem /etc/etcd/ssl/

ssh -n 192.168.10.13 "mkdir -p /etc/etcd/ssl && exit"
ssh -n 192.168.10.14 "mkdir -p /etc/etcd/ssl && exit"

scp -r /etc/etcd/ssl/*.pem 192.168.10.13:/etc/etcd/ssl/
scp -r /etc/etcd/ssl/*.pem 192.168.10.14:/etc/etcd/ssl/
```

**1.4.2、在3台主节点上操作，安装etcd并配置**

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
#10.14机器上操作
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
member 1af68d968c7e3f22 is healthy: got healthy result from https://192.168.10.12:2379
member 55204c19ed228077 is healthy: got healthy result from https://192.168.10.14:2379
member e8d9a97b17f26476 is healthy: got healthy result from https://192.168.10.13:2379
cluster is healthy
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
#6.配置daemon, 因为kubelet的启动环境变量要与docker的cgroup-driver驱动相同,以下是官方推荐处理方式
#由于国内拉取镜像较慢，配置文件最后追加了阿里云镜像加速配置。
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ],
  "registry-mirrors": ["https://uyah70su.mirror.aliyuncs.com"]
}
EOF
#7.设置开机自启动
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
```

``` bash
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

[参考：kubernetes](<https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init/>)

[参考：kubeadm](<https://godoc.org/k8s.io/kubernetes/cmd/kubeadm/app/apis/kubeadm/v1beta1>)

```bash
kubeadm config print init-defaults > kubeadm-config.yaml	#这个命令可以生成初始化配置文件然后修改,也可以直接用下面的

# 1.创建初始化集群配置文件
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

**1.7.3、部署flannel网络，在所有节点上执行**

```bash
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
#或者github
https://github.com/coreos/flannel/blob/master/Documentation/kube-flannel.yml
#版本信息：quay.io/coreos/flannel:v0.11.0-amd64

cat kube-flannel.yml | grep image
cat kube-flannel.yml | grep 10.244

sed -i 's#quay.io/coreos/flannel:v0.11.0-amd64#willdockerhub/flannel:v0.11.0-amd64#g' kube-flannel.yml	#如果网络比较好，可不修改

kubectl apply -f kube-flannel.yml

#或者直接创建
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

**1.7.4、加入集群后验证**

```bash
#1.master上执行，加入集群命令
kubeadm join 192.168.10.100:6443 --token y6v90q.i6bl1bwcgg8clvh5 \
    --discovery-token-ca-cert-hash sha256:179c5689ef32be2123c9f02015ef25176d177c54322500665f1170f26368ae3d \
    --experimental-control-plane --certificate-key 3044cb04c999706795b28c1d3dcd2305dcf181787d7c6537284341a985395c20
#2.拷贝kube到用户目录
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

#3.node上执行 加入集群
#如果忘记node节点加入集群的命令可以使用kubeadm token create --print-join-command 查看

kubeadm join 192.168.10.100:6443 --token y6v90q.i6bl1bwcgg8clvh5 \
    --discovery-token-ca-cert-hash sha256:179c5689ef32be2123c9f02015ef25176d177c54322500665f1170f26368ae3d

#4.验证集群状态
kubectl -n kube-system get pod -o wide	#查看pod运行情况

kubectl get nodes -o wide #查看节点情况

kubectl -n kube-system get svc	#查看service
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   16m

ipvsadm -ln		#查看代理规则
```

**1.7.5、集群测试**

准备部署一个简单的web服务来测试集群。

```bash
cat > /opt/deployment-goweb.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: goweb
spec:
  selector:
    matchLabels:
      app: goweb
  replicas: 4
  template:
    metadata:
      labels:
        app: goweb
    spec: 
      containers: 
      - image: lingtony/goweb
        name: goweb
        ports: 
        - containerPort: 8000
EOF

#-------------------------------------

cat > /opt/svc-goweb.yaml << EOF
apiVersion: v1
kind: Service
metadata:
  name: gowebsvc
spec:
  selector:
    app: goweb
  ports:
  - name: default
    protocol: TCP
    port: 80
    targetPort: 8000
EOF

# -----------------------------------部署服务
kubectl apply -f deployment-goweb.yaml
kubectl  apply -f svc-goweb.yaml
#--------------查看pod
get pod -o wide
NAME                    READY   STATUS    RESTARTS   AGE   IP           NODE       NOMINATED NODE   READINESS GATES
goweb-6c569f884-4ln4s   1/1     Running   0          75s   10.244.1.2   n-kube15   <none>           <none>
goweb-6c569f884-jcnrs   1/1     Running   0          75s   10.244.1.3   n-kube15   <none>           <none>
goweb-6c569f884-njnzk   1/1     Running   0          75s   10.244.1.4   n-kube15   <none>           <none>
goweb-6c569f884-zxnrx   1/1     Running   0          75s   10.244.1.5   n-kube15   <none>           <none>

#--------查看服务
kubectl  get svc
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
gowebsvc     ClusterIP   10.105.87.199   <none>        80/TCP    84s
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP   30m

#-----访问测试,可以看到对SVC的请求会在pod之间负载
curl http://10.105.87.199/info  #  Hostname: goweb-6c569f884-jcnrs
curl http://10.105.87.199/info  #  Hostname: goweb-6c569f884-4ln4s
curl http://10.105.87.199/info  #  Hostname: goweb-6c569f884-zxnrx
curl http://10.105.87.199/info  #  Hostname: goweb-6c569f884-njnzk
curl http://10.105.87.199/info  #  Hostname: goweb-6c569f884-jcnrs
curl http://10.105.87.199/info  #  Hostname: goweb-6c569f884-4ln4s
curl http://10.105.87.199/info  #  Hostname: goweb-6c569f884-zxnrx
curl http://10.105.87.199/info  #  Hostname: goweb-6c569f884-njnzk
curl http://10.105.87.199/info  #  Hostname: goweb-6c569f884-jcnrs           
```

### 1.8、配置dashboard

默认是没web界面的，可以在master机器上安装一个dashboard插件，实现通过web来管理。

dashboard项目的GitHub地址：https://github.com/kubernetes/dashboard/releases

准备的镜像：

k8s.gcr.io/kubernetes-dashboard-amd64:v1.10.1

咱们可以先从阿里镜像库拉取镜像

```bash
#1.下载镜像
vim /etc/kubernetes/dashboard.sh

#!/bin/bash
DASHDOARD_VERSION=v1.10.1
HEAPSTER_VERSION=v1.5.4
GRAFANA_VERSION=v5.0.4
INFLUXDB_VERSION=v1.5.2
username=registry.cn-hangzhou.aliyuncs.com/google_containers
images=(
        kubernetes-dashboard-amd64:${DASHDOARD_VERSION}
        heapster-grafana-amd64:${GRAFANA_VERSION}
        heapster-amd64:${HEAPSTER_VERSION}
        heapster-influxdb-amd64:${INFLUXDB_VERSION}
        )
for image in ${images[@]}
do
docker pull ${username}/${image}
docker tag ${username}/${image} k8s.gcr.io/${image}
docker rmi ${username}/${image}
done

#2.准备yaml文件,下载GitHub上的文件
wget https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml

#3.修改配置文件在最下面
vim /etc/kubernetes/kubernetes-dashboard.yaml
# ------------------- Dashboard Service ------------------- #
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system
spec:
  type: NodePort		## 新增把 Dashboard 端口暴露出来，方便外部访问
  ports:
    - port: 443
      targetPort: 8443
	  nodePort: 9527	## 暴露出的端口
  selector:
    k8s-app: kubernetes-dashboard
#---------yaml中有涉及的image版本，根据实际情况修改-------#
spec:
      containers:
      - name: kubernetes-dashboard
        image: k8s.gcr.io/kubernetes-dashboard-amd64:v1.10.1
        imagePullPolicy: IfNotPresent	#可以添加拉取策略

#4.创建服务账号和集群角色绑定配置文件
cat << EOF > /etc/kubernetes/kubernetes-dashboard-admin-rbac.yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-admin
  namespace: kube-system
  
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard-admin
  labels:
    k8s-app: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: kubernetes-dashboard-admin
  namespace: kube-system
EOF

#5.执行安装
kubectl apply -f kubernetes-dashboard.yaml
kubectl apply -f kubernetes-dashboard-admin-rbac.yaml

#6.执行完成后查看pod是否正常运行
kubectl get pod -n kube-system |grep kubernetes-dashboard
#显示如下
kubernetes-dashboard-6cfdc589c7-c6qmq   1/1     Running   0          99m

#7.查看服务暴露的端口号
kubectl get service  -n kube-system |grep kubernetes-dashboard
#显示如下
kubernetes-dashboard-external   NodePort    10.96.149.139   <none>        443:9527/TCP           99m

#8.查看 Token
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep kubernetes-dashboard-admin-token | awk '{print $1}')
#显示如下
Name:         kubernetes-dashboard-admin-token-2rrq2
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: kubernetes-dashboard-admin
              kubernetes.io/service-account.uid: aeff190c-93eb-11e9-904c-000c29959a05

Type:  kubernetes.io/service-account-token

Data
====
namespace:  11 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJrdWJlcm5ldGVzLWRhc2hib2FyZC1hZG1pbi10b2tlbi0ycnJxMiIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJrdWJlcm5ldGVzLWRhc2hib2FyZC1hZG1pbiIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6ImFlZmYxOTBjLTkzZWItMTFlOS05MDRjLTAwMGMyOTk1OWEwNSIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDprdWJlLXN5c3RlbTprdWJlcm5ldGVzLWRhc2hib2FyZC1hZG1pbiJ9.VcyOeYta1PrleZ4PDn_mvxtf8jiAo9DOboL5inMk9QJY1raDCI7EOHaVDF1OPLgYR2JqTVDLjshLwFkm3I4zO49piApgxd9fLrIA1RW30trNG9XxfG8P5O00RuYQxdRGfIeYcSdFgeroKdHY10wgBsAFbd8DWkc_IyYPHe-gnn_Y2U5Hd1tPZGOk_ZvZXhjlQd25vYouBI1RBEVUlcug5HaDGqHH_2yYmba4AFI2rVjsnxNbeSca5Ri9384vCsJQSkvh1uKMQTXuUXZb3z6x2nKKx9vA7LxoHYKJkyLMNbvKqL5QYpS3t9aVuzYTWVuUxEunnmEcT9R5oqceGwCwtg
ca.crt:     1025 bytes

#9.在浏览器输入<master_ip:端口> 就可以访问Dashboard了，然后kubernetes仪表板，选择令牌，输入查到的Token
```

## 2、二进制文件方式安装kubernetes集群

所有操作全部用root使用者进行，高可用一般建议大于等于3台的奇数,我们使用3台master来做高可用

练习环境说明：[参考1](https://www.cnblogs.com/harlanzhang/p/10131264.html)		 [参考2](https://blog.51cto.com/13941177/2316025) 		[参考3](https://www.cnblogs.com/JetpropelledSnake/p/10612763.html) 		[参考4](https://www.kubernetes.org.cn/5163.html)		[参考5](https://blog.csdn.net/Yan_Chou/article/details/78954602)		[参考6](http://www.pianshen.com/article/1167211572/#362_kubeapiserver_693) 		[参考7](https://blog.51cto.com/13210651/2362052)		[参考8](https://www.jianshu.com/p/c492f5efdadf?utm_campaign=maleskine&utm_content=note&utm_medium=seo_notes&utm_source=recommendation)		[参考GitHub](https://github.com/opsnull/follow-me-install-kubernetes-cluster)		

master： kube-apiserver，kube-controller-manager，kube-scheduler，flanneld 

node： kubelet，kube-proxy，flannel

Service_CIDR：10.254.0.0/16	服务网段，部署前路由不可达，部署后集群内部使用IP:Port可达

Cluster_CIDR：172.30.0.0/16	pod网段，部署前路由不可达，部署后路由可达(flanneld 保证)

| 主机名称 | IP地址         | 部署软件                       | 备注   |
| -------- | -------------- | ------------------------------ | ------ |
| k8s-m12  | 192.168.10.12  | keepalived+haproxy+etcd+master | master |
| k8s-m13  | 192.168.10.13  | keepalived+haproxy+etcd+master | master |
| k8s-m14  | 192.168.10.14  | keepalived+haproxy+etcd+master | master |
| k8s-n15  | 192.168.10.15  | node+docker                    | work   |
| k8s-n16  | 192.168.10.16  | node+docker                    | work   |
| VIP      | 192.168.10.100 |                                | VIP    |

### 2.1、下载安装包

[kubernetes的GitHub网址](<https://github.com/kubernetes/kubernetes/releases>) ： <https://github.com/kubernetes/kubernetes/releases>  

下载Server Binaries中的 [kubernetes-server-linux-amd64.tar.gz](https://dl.k8s.io/v1.14.3/kubernetes-server-linux-amd64.tar.gz) 安装包

下载Node Binaries中的 [kubernetes-node-linux-amd64.tar.gz](https://dl.k8s.io/v1.14.3/kubernetes-node-linux-amd64.tar.gz) 安装包

下载Client Binares中的 kubernetes-client-linux-amd64.tar.gz 安装包

[各种CA证书类型参考](https://github.com/kubernetes-incubator/apiserver-builder-alpha/blob/master/docs/concepts/auth.md)

**k8s各版本组件下载地址**

```bash
https://github.com/kubernetes/kubernetes/tree/v1.14.3

#kubernetes
wget https://storage.googleapis.com/kubernetes-release/release/v1.14.3/kubernetes-node-linux-amd64.tar.gz
wget https://storage.googleapis.com/kubernetes-release/release/v1.14.3/kubernetes-client-linux-amd64.tar.gz
wget https://storage.googleapis.com/kubernetes-release/release/v1.14.3/kubernetes-server-linux-amd64.tar.gz
wget https://storage.googleapis.com/kubernetes-release/release/v1.14.3/kubernetes.tar.gz
#etcd
wget https://github.com/etcd-io/etcd/releases/download/v3.3.13/etcd-v3.3.13-linux-amd64.tar.gz
#flannel
wget https://github.com/coreos/flannel/releases/download/v0.11.0/flannel-v0.11.0-linux-amd64.tar.gz
#cni-plugins
wget https://github.com/containernetworking/plugins/releases/download/v0.8.1/cni-plugins-linux-amd64-v0.8.1.tgz
#docker
wget https://download.docker.com/linux/static/stable/x86_64/docker-18.09.6.tgz
#cfssl
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
#heapster
wget https://github.com/kubernetes-retired/heapster/archive/v1.5.4.tar.gz
```

### 2.2、环境准备

```bash
#1.12机器上生成密钥，无密码ssh登陆
ssh-keygen -t rsa
ssh-copy-id 192.168.10.13  #依次拷贝到其他节点上

#2.关闭防火墙，以下点所有机器执行
systemctl stop firewalld
systemctl disable firewalld

#3.关闭swap分区
swapoff -a
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

#4.关闭SELinux
sestatus	#查看selinux状态
setenforce 0		#临时关闭selinux
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config

#5.升级内核参考：https://www.cnblogs.com/fan-gx/p/11006762.html

#6.修改文件句柄数
cat <<EOF >>/etc/security/limits.conf
* soft nofile 65536
* hard nofile 65536
* soft nproc 65536
* hard nproc 65536
* soft  memlock  unlimited
* hard memlock  unlimited
EOF

#7.安装ipvs
yum install ipvsadm ipset sysstat conntrack libseccomp -y
#开机加载内核模块，并设置开机自动加载
cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
EOF

#然后执行脚本
chmod 755 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules
lsmod | grep -e ip_vs -e nf_conntrack_ipv4

#8.修改系统参数
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_nonlocal_bind = 1
net.ipv4.ip_forward = 1
vm.swappiness=0
EOF
sysctl --system
#-----------下面参考别人的---------#
# cat <<EOF > /etc/sysctl.d/k8s.conf
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 10
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
net.ipv4.neigh.default.gc_stale_time = 120
net.ipv4.conf.all.rp_filter = 0
net.ipv4.conf.default.rp_filter = 0
net.ipv4.conf.default.arp_announce = 2
net.ipv4.conf.lo.arp_announce = 2
net.ipv4.conf.all.arp_announce = 2
net.ipv4.ip_forward = 1
net.ipv4.tcp_max_tw_buckets = 5000
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 1024
net.ipv4.tcp_synack_retries = 2
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.netfilter.nf_conntrack_max = 2310720
fs.inotify.max_user_watches=89100
fs.may_detach_mounts = 1
fs.file-max = 52706963
fs.nr_open = 52706963
net.bridge.bridge-nf-call-arptables = 1
vm.swappiness = 0
vm.overcommit_memory=1
vm.panic_on_oom=0
EOF

#9.在生产环境建议预留内存,避免由于内存耗尽导致ssh连不上主机（32G的机器留2G，251的留3G, 500G的留5G）。下面是预留5G
echo 'vm.min_free_kbytes=5000000' >> /etc/sysctl.conf
sysctl -p
```

### 2.3、部署docker

二进制部署方式可参考：https://www.kubernetes.org.cn/3831.html    这里为了方便直接yum安装所有节点

```bash
#1.安装yum源工具包
yum install -y yum-utils device-mapper-persistent-data lvm2

#2.下载docker-ce官方的yum源配置文件
# yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
# yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

#3.禁用docker-c-edge源配edge是不开发版，不稳定，下载stable版
yum-config-manager --disable docker-ce-edge
#4.更新本地YUM源缓存
yum makecache fast
#5.安装Docker-ce相应版本
yum -y install docker-ce
#6.配置daemon, 因为kubelet的启动环境变量要与docker的cgroup-driver驱动相同,以下是官方推荐处理方式（现在新版二进制kubelet就是cgroup了）
#由于国内拉取镜像较慢，配置文件最后追加了阿里云镜像加速配置。
mkdir -p /etc/docker && 
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ],
  "registry-mirrors": ["https://uyah70su.mirror.aliyuncs.com"]
}
EOF
#7.设置开机自启动
systemctl restart docker && systemctl enable docker && systemctl status docker

#8.可以先在自己电脑下来安装包，本环境安装的是18.09版本
yum install --downloadonly docker-ce-18.09 --downloaddir=/opt	#yum下载docker-ce
yum localinstall docker-ce -y		#然后安装
```

### 2.4、部署etcd

关于更多介绍可以参考：[参考1](https://blog.51cto.com/13210651/2358716)		[参考2](https://blog.51cto.com/sgk2011/2108542)

etcd是用来保存集群所有状态的 Key/Value 存储系统，常用于服务发现、共享配置以及并发控制（如 leader 选举、分布式锁等）。kubernetes 使用 etcd 存储所有运行数据。

所有 Kubernetes 组件会通过 API Server 来跟 Etcd 进行沟通从而保存或读取资源状态。有条件的可以单独几台机器跑,不过需要配置apiserver指向etcd集群。

#### 2.4.1、创建etcd证书

如果不希望将cfssl工具安装到部署主机上，可以在其他的主机上进行该步骤，生成以后将证书拷贝到部署etcd的主机上即可。不是要证书也可以部署，etcd.service文件和etcd.conf文件不要有https的URL

```bash
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 -O /usr/local/bin/cfssl
wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64 -O /usr/local/bin/cfssljson
wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64 -O /usr/local/bin/cfssl-certinfo

chmod +x /usr/local/bin/cfssl*

#配置CA文件
mkdir /root/ssl && cd /root/ssl

cat >  ca-config.json <<EOF
{
"signing": {
"default": {
  "expiry": "8760h"
},
"profiles": {
  "kubernetes": {
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

#2----------------------------------------------
cat >  ca-csr.json <<EOF
{
"CN": "kubernetes",
"key": {
"algo": "rsa",
"size": 2048
},
"names": [
{
  "C": "CN",
  "ST": "ShangHai",
  "L": "ShangHai",
  "O": "k8s",
  "OU": "System"
}
]
}
EOF

#3--------------------------------------------------
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
      "ST": "ShangHai",
      "L": "ShangHai",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF
# hosts字段的IP地址是指授权使用证书的etcd地址
#------------------------------------
cfssl gencert -initca ca-csr.json | cfssljson -bare ca

cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes etcd-csr.json | cfssljson -bare etcd

#生产后证书包含文件如下，共9个
ca-config.json
ca.csr
ca-csr.json
ca-key.pem
ca.pem
etcd.csr
etcd-csr.json
etcd-key.pem
etcd.pem

#将生成好的etcd.pem和etcd-key.pem以及ca.pem三个文件拷贝到etcd机器上
mkdir -p /etc/kubernetes/ssl && cp *.pem /etc/kubernetes/ssl/
ssh -n 192.168.10.13 "mkdir -p /etc/kubernetes/ssl && exit"
ssh -n 192.168.10.14 "mkdir -p /etc/kubernetes/ssl && exit"

scp -r /etc/kubernetes/ssl/*.pem 192.168.10.13:/etc/kubernetes/ssl/
scp -r /etc/kubernetes/ssl/*.pem 192.168.10.14:/etc/kubernetes/ssl/
```

#### 2.4.2、部署etcd

将下载的etcd二进制文件上传到etcd节点机器上。

```bash
#在etcd的机器上安装etcd程序
mkdir -p /var/lib/etcd

tar -zxvf etcd-v3.3.13-linux-amd64.tar.gz
cp etcd-v3.3.13-linux-amd64/etcd* /usr/local/bin
scp etcd-v3.3.13-linux-amd64/etcd* 192.168.10.13:/usr/local/bin
scp etcd-v3.3.13-linux-amd64/etcd* 192.168.10.14:/usr/local/bin

#1.在12机器上创建etcd.service文件
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
ExecStart=/usr/local/bin/etcd \
  --name k8s-m12 \
  --cert-file=/etc/kubernetes/ssl/etcd.pem \
  --key-file=/etc/kubernetes/ssl/etcd-key.pem \
  --trusted-ca-file=/etc/kubernetes/ssl/ca.pem \
  --peer-cert-file=/etc/kubernetes/ssl/etcd.pem \
  --peer-key-file=/etc/kubernetes/ssl/etcd-key.pem \
  --peer-trusted-ca-file=/etc/kubernetes/ssl/ca.pem \
  --initial-advertise-peer-urls https://192.168.10.12:2380 \
  --listen-peer-urls https://192.168.10.12:2380 \
  --listen-client-urls https://192.168.10.12:2379,http://127.0.0.1:2379 \
  --advertise-client-urls https://192.168.10.12:2379 \
  --initial-cluster-token etcd-cluster-0 \
  --initial-cluster k8s-m12=https://192.168.10.12:2380,k8s-m13=https://192.168.10.13:2380,k8s-m14=https://192.168.10.14:2380 \
  --initial-cluster-state new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF

#2.启动etcd服务
systemctl daemon-reload && systemctl enable etcd.service && systemctl start etcd.service && systemctl status etcd
```

```bash
#1.在13机器上创建etcd.service文件
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
ExecStart=/usr/local/bin/etcd \
  --name k8s-m13 \
  --cert-file=/etc/kubernetes/ssl/etcd.pem \
  --key-file=/etc/kubernetes/ssl/etcd-key.pem \
  --peer-cert-file=/etc/kubernetes/ssl/etcd.pem \
  --peer-key-file=/etc/kubernetes/ssl/etcd-key.pem \
  --trusted-ca-file=/etc/kubernetes/ssl/ca.pem \
  --peer-trusted-ca-file=/etc/kubernetes/ssl/ca.pem \
  --initial-advertise-peer-urls https://192.168.10.13:2380 \
  --listen-peer-urls https://192.168.10.13:2380 \
  --listen-client-urls https://192.168.10.13:2379,http://127.0.0.1:2379 \
  --advertise-client-urls https://192.168.10.13:2379 \
  --initial-cluster-token etcd-cluster-0 \
  --initial-cluster k8s-m12=https://192.168.10.12:2380,k8s-m13=https://192.168.10.13:2380,k8s-m14=https://192.168.10.14:2380 \
  --initial-cluster-state new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF

#2.启动etcd服务
systemctl daemon-reload && systemctl enable etcd.service && systemctl start etcd.service && systemctl status etcd
```

```bash
#1.在14机器上创建etcd.service文件
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
ExecStart=/usr/local/bin/etcd \
  --name k8s-m14 \
  --cert-file=/etc/kubernetes/ssl/etcd.pem \
  --key-file=/etc/kubernetes/ssl/etcd-key.pem \
  --peer-cert-file=/etc/kubernetes/ssl/etcd.pem \
  --peer-key-file=/etc/kubernetes/ssl/etcd-key.pem \
  --trusted-ca-file=/etc/kubernetes/ssl/ca.pem \
  --peer-trusted-ca-file=/etc/kubernetes/ssl/ca.pem \
  --initial-advertise-peer-urls https://192.168.10.14:2380 \
  --listen-peer-urls https://192.168.10.14:2380 \
  --listen-client-urls https://192.168.10.14:2379,http://127.0.0.1:2379 \
  --advertise-client-urls https://192.168.10.14:2379 \
  --initial-cluster-token etcd-cluster-0 \
  --initial-cluster k8s-m12=https://192.168.10.12:2380,k8s-m13=https://192.168.10.13:2380,k8s-m14=https://192.168.10.14:2380 \
  --initial-cluster-state new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF

#2.启动etcd服务
systemctl daemon-reload && systemctl enable etcd.service && systemctl start etcd.service && systemctl status etcd
```

#### 2.4.3、验证集群

```bash
#1.查看集群状态
etcdctl --ca-file=/etc/kubernetes/ssl/ca.pem --cert-file=/etc/kubernetes/ssl/etcd.pem --key-file=/etc/kubernetes/ssl/etcd-key.pem cluster-health
#返回如下，代表集群正常
member 1af68d968c7e3f22 is healthy: got healthy result from https://192.168.10.12:2379
member 55204c19ed228077 is healthy: got healthy result from https://192.168.10.14:2379
member e8d9a97b17f26476 is healthy: got healthy result from https://192.168.10.13:2379
cluster is healthy

#2.查看集群成员
etcdctl --endpoints=https://192.168.10.12:2379,https://192.168.10.13:2379,https://192.168.10.14:2379  --ca-file=/etc/kubernetes/ssl/ca.pem  --cert-file=/etc/kubernetes/ssl/etcd.pem  --key-file=/etc/kubernetes/ssl/etcd-key.pem member list
#返回如下结果
1af68d968c7e3f22: name=k8s-m12 peerURLs=https://192.168.10.12:2380 clientURLs=https://192.168.10.12:2379 isLeader=false
55204c19ed228077: name=k8s-m14 peerURLs=https://192.168.10.14:2380 clientURLs=https://192.168.10.14:2379 isLeader=false
e8d9a97b17f26476: name=k8s-m13 peerURLs=https://192.168.10.13:2380 clientURLs=https://192.168.10.13:2379 isLeader=true
```

### 2.5、部署flannel

所有的节点都需要安装flannel，，主要目的是跨主机的docker能够互相通信，也是保障kubernetes集群的网络基础和保障

#### 2.5.1、创建flannel证书

```bash
#1.生产TLS证书，是让kubectl当做client证书使用,(证书只需要生成一次)
cd /root/ssl
cat > flanneld-csr.json <<EOF
{
  "CN": "flanneld",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "ShangHai",
      "L": "ShangHai",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF

#2.生成证书和私钥
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes flanneld-csr.json | cfssljson -bare flanneld
#包含以下文件
flanneld.csr
flanneld-csr.json
flanneld-key.pem
flanneld.pem

#3.然后将证书拷贝到所有节点下。
cp flanneld*.pem /etc/kubernetes/ssl
scp flanneld*.pem 192.168.10.13:/etc/kubernetes/ssl
scp flanneld*.pem 192.168.10.14:/etc/kubernetes/ssl
scp flanneld*.pem 192.168.10.15:/etc/kubernetes/ssl
scp flanneld*.pem 192.168.10.16:/etc/kubernetes/ssl
```
#### 2.5.2、部署flannel

```bash
#1.开始安装flannel
tar -zvxf flannel-v0.11.0-linux-amd64.tar.gz
cp flanneld mk-docker-opts.sh /usr/local/bin
scp flanneld mk-docker-opts.sh 192.168.10.13:/usr/local/bin
scp flanneld mk-docker-opts.sh 192.168.10.14:/usr/local/bin
scp flanneld mk-docker-opts.sh 192.168.10.15:/usr/local/bin
scp flanneld mk-docker-opts.sh 192.168.10.16:/usr/local/bin

#2.向etcd写入集群Pod网段信息，在etcd集群中任意一台执行一次即可
etcdctl \
--endpoints=https://192.168.10.12:2379,https://192.168.10.13:2379,https://192.168.10.14:2379 \
--ca-file=/etc/kubernetes/ssl/ca.pem \
--cert-file=/etc/kubernetes/ssl/flanneld.pem \
--key-file=/etc/kubernetes/ssl/flanneld-key.pem \
mk /kubernetes/network/config '{"Network":"172.30.0.0/16", "SubnetLen": 24, "Backend": {"Type": "vxlan"}}'
#----得到返回信息如下，设置的网络是172.30.0.0/16，子网掩码是24位
{"Network":"172.30.0.0/16", "SubnetLen": 24, "Backend": {"Type": "vxlan"}}

#2.1.列出键值存储的目录
etcdctl \
--ca-file=/etc/kubernetes/ssl/ca.pem \
--cert-file=/etc/kubernetes/ssl/flanneld.pem \
--key-file=/etc/kubernetes/ssl/flanneld-key.pem ls -r
#2.2.查看键值存储
etcdctl \
--ca-file=/etc/kubernetes/ssl/ca.pem \
--cert-file=/etc/kubernetes/ssl/flanneld.pem \
--key-file=/etc/kubernetes/ssl/flanneld-key.pem get /kubernetes/network/config
#2.3查看已分配pod的子网列表
etcdctl \
--ca-file=/etc/kubernetes/ssl/ca.pem \
--cert-file=/etc/kubernetes/ssl/flanneld.pem \
--key-file=/etc/kubernetes/ssl/flanneld-key.pem ls  /kubernetes/network/subnets

#3、创建flannel.service文件
cat > /etc/systemd/system/flannel.service << EOF
[Unit]
Description=Flanneld overlay address etcd agent
After=network.target
After=network-online.target
Wants=network-online.target
After=etcd.service
Before=docker.service

[Service]
Type=notify
ExecStart=/usr/local/bin/flanneld \
  -etcd-cafile=/etc/kubernetes/ssl/ca.pem \
  -etcd-certfile=/etc/kubernetes/ssl/flanneld.pem \
  -etcd-keyfile=/etc/kubernetes/ssl/flanneld-key.pem \
  -etcd-endpoints=https://192.168.10.12:2379,https://192.168.10.13:2379,https://192.168.10.14:2379 \
  -etcd-prefix=/kubernetes/network
ExecStartPost=/usr/local/bin/mk-docker-opts.sh -k DOCKER_NETWORK_OPTIONS -d /run/flannel/docker
Restart=on-failure

[Install]
WantedBy=multi-user.target
RequiredBy=docker.service
EOF
#mk-docker-opts.sh 脚本将分配给flanneld的Pod子网网段信息写入到/run/flannel/docker文件中，后续docker启动时使用这个文件中参数值设置docker0网桥。
#flanneld 使用系统缺省路由所在的接口和其它节点通信，对于有多个网络接口的机器（如，内网和公网），可以用 -iface=enpxx 选项值指定通信接口。

#4、启动flannel
systemctl daemon-reload && systemctl enable flannel && systemctl start flannel && systemctl status flannel

#5.验证flannel
cat /run/flannel/docker				#/run/flannel/docker是flannel分配给docker的子网信息，
#显示如下
DOCKER_OPT_BIP="--bip=172.30.7.1/24"
DOCKER_OPT_IPMASQ="--ip-masq=true"
DOCKER_OPT_MTU="--mtu=1450"
DOCKER_NETWORK_OPTIONS=" --bip=172.30.7.1/24 --ip-masq=true --mtu=1450"

cat /run/flannel/subnet.env 		#/run/flannel/subnet.env包含了flannel整个大网段以及在此节点上的子网段
#显示如下
FLANNEL_NETWORK=172.30.0.0/16
FLANNEL_SUBNET=172.30.7.1/24
FLANNEL_MTU=1450
FLANNEL_IPMASQ=false

ip add | grep flannel		#查看网卡信息
4: flannel.1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default 
    inet 172.30.7.0/32 scope global flannel.1
```

#### 2.5.3、配置docker支持flannel

```bash
#1.配置docker支持flannel网络,需要在[Service]标签下新加
vim /etc/systemd/system/multi-user.target.wants/docker.service
EnvironmentFile=/run/flannel/docker		#这行新加内容，下面行新加$后面的内容
ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock $DOCKER_NETWORK_OPTIONS

#2.重启docker,然后可以查看到已分配pod的子网列表
systemctl daemon-reload && systemctl restart docker && systemctl status docker

ip add | grep docker
#docker0网口IP地址，已改变
4: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    inet 172.30.7.1/24 brd 172.30.7.255 scope global docker0
```

#### 2.5.4、设置CNI插件支持flannel

```bash
tar -zxvf cni-plugins-linux-amd64-v0.8.1.tgz -C /opt/cni
mkdir -p /etc/cni/net.d
cat > /etc/cni/net.d/10-default.conf <<EOF
{
    "name": "flannel",
    "type": "flannel",
    "delegate": {
        "bridge": "docker0",
        "isDefaultGateway": true,
        "mtu": 1400
    }
}
EOF

#把相关指令和文件拷贝到所有主机
scp /opt/cni/* 192.168.10.13:/usr/local/bin && scp /etc/cni/net.d/* 192.168.10.13:/etc/cni/net.d/
```

### 2.6、部署keepalived+haproxy

keepalived 提供 kube-apiserver 对外服务的 VIP；haproxy 监听 VIP，后端连接所有 kube-apiserver 实例，提供健康检查和负载均衡功能；

本文档复用 master 节点的三台机器，haproxy 监听的端口(8443) 需要与 kube-apiserver 的端口 6443 不同，避免冲突。

keepalived 在运行过程中周期检查本机的 haproxy 进程状态，如果检测到 haproxy 进程异常，则触发重新选主的过程，VIP 将飘移到新选出来的主节点，从而实现 VIP 的高可用。所有组件（如 kubeclt、apiserver、controller-manager、scheduler 等）都通过 VIP 和 haproxy 监听的 8443 端口访问 kube-apiserver 服务。

#### 2.6.1、安装haproxy

```bash
yum install -y haproxy

#12机器上配置
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

listen  admin_stats
    bind 0.0.0.0:9090
    mode http
    log 127.0.0.1 local0 err
    stats refresh 30s
    stats uri /status
    stats realm welcome login\ Haproxy
    stats auth admin:123456
    stats hide-version
    stats admin if TRUE

frontend kubernetes
    bind *:8443
    mode tcp
    default_backend kubernetes-master

backend kubernetes-master
    balance roundrobin
    server k8s-m12 192.168.10.12:6443 check maxconn 2000
    server k8s-m13 192.168.10.13:6443 check maxconn 2000
    server k8s-m14 192.168.10.14:6443 check maxconn 2000
EOF

#13 和 14机器上配置都一样

# 启动haproxy
systemctl enable haproxy && systemctl start haproxy && systemctl status haproxy
```

#### 2.6.2、安装keepalived

```bash
yum install -y keepalived

#10.12机器上配置

cat <<EOF > /etc/keepalived/keepalived.conf
global_defs {
   router_id LVS_k8s
}

vrrp_script CheckK8sMaster {
    script "curl -k https://192.168.10.100:8443"
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
    script "curl -k https://192.168.10.100:8443"
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
    script "curl -k https://192.168.10.100:8443"
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
systemctl restart keepalived && systemctl enable keepalived && systemctl status keepalived

#查看vip
ip add | grep 10.100
```

### 2.7、部署master

kube-scheduler，kube-controller-manager 和 kube-apiserver 三者的功能紧密相关；同时kube-scheduler 和 kube-controller-manager 只能有一个进程处于工作状态，如果运行多个，则需要通过选举产生一个 leader；

#### 2.7.1、部署kubectl命令工具

kubectl 是 kubernetes 集群的命令行管理工具，默认从 `~/.kube/config` 文件读取 kube-apiserver 地址、证书、用户名等信息，如果没有配置，执行 kubectl 命令时可能会出错。`~/.kube/config`只需要部署一次，然后拷贝到其他的master。

```bash
#1.解压命令
tar -zxvf kubernetes-server-linux-amd64.tar.gz
cd kubernetes/server/bin/
cp kube-apiserver kubeadm kube-controller-manager kubectl kube-scheduler /usr/local/bin
scp kube-apiserver kubeadm kube-controller-manager kubectl kube-scheduler 192.168.10.13:/usr/local/bin
scp kube-apiserver kubeadm kube-controller-manager kubectl kube-scheduler 192.168.10.14:/usr/local/bin

#2.创建CA证书
cd /root/ssl
cat > admin-csr.json <<EOF
{
  "CN": "admin",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "ShangHai",
      "L": "ShangHai",
      "O": "system:masters",
      "OU": "System"
    }
  ]
}
EOF

#3.生成证书和私钥
cfssl gencert -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes admin-csr.json | cfssljson -bare admin

#4.创建~/.kube/config文件
kubectl config set-cluster kubernetes \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://192.168.10.100:8443 \
  --kubeconfig=kubectl.kubeconfig

#4.1.设置客户端认证参数
kubectl config set-credentials admin \
  --client-certificate=admin.pem \
  --client-key=admin-key.pem \
  --embed-certs=true \
  --kubeconfig=kubectl.kubeconfig

#4.2.设置上下文参数
kubectl config set-context kubernetes \
  --cluster=kubernetes \
  --user=admin \
  --kubeconfig=kubectl.kubeconfig
  
#4.3.设置默认上下文
kubectl config use-context kubernetes --kubeconfig=kubectl.kubeconfig

#4.4.拷贝kubectl.kubeconfig文件
cp kubectl.kubeconfig ~/.kube/config
scp kubectl.kubeconfig 192.168.10.13:/root/.kube/config
scp kubectl.kubeconfig 192.168.10.14:/root/.kube/config

cp admin*.pem /etc/kubernetes/ssl/
scp admin*.pem 192.168.10.13:/etc/kubernetes/ssl/
scp admin*.pem 192.168.10.14:/etc/kubernetes/ssl/
```

#### 2.7.2、部署api-server

```bash
#1.创建CA证书,hosts字段指定授权使用该证书的IP或域名列表，这里列出了VIP/apiserver节点IP/kubernetes服务IP和域名
cd /root/ssl
cat > kubernetes-csr.json <<EOF
{
  "CN": "kubernetes",
  "hosts": [
    "127.0.0.1",
    "192.168.10.12",
    "192.168.10.13",
    "192.168.10.14",
    "192.168.10.100",
    "10.254.0.1",
    "kubernetes",
    "kubernetes.default",
    "kubernetes.default.svc",
    "kubernetes.default.svc.cluster",
    "kubernetes.default.svc.cluster.local"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "ShangHai",
      "L": "ShangHai",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF

#2.生成证书和私钥
cfssl gencert -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes kubernetes-csr.json | cfssljson -bare kubernetes

#3.将证书拷贝到其他master节点
cp kubernetes*.pem /etc/kubernetes/ssl/
scp kubernetes*.pem 192.168.10.13:/etc/kubernetes/ssl/
scp kubernetes*.pem 192.168.10.14:/etc/kubernetes/ssl/

#4.创建加密配置文件
cat > encryption-config.yaml <<EOF
kind: EncryptionConfig
apiVersion: v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: $(head -c 32 /dev/urandom | base64)
      - identity: {}
EOF

#4.1创建kube-apiserver使用的客户端令牌文件
cat <<EOF > bootstrap-token.csv
$(head -c 32 /dev/urandom | base64),kubelet-bootstrap,10001,"system:kubelet-bootstrap"
EOF

#5.将加密文件拷贝到其他master节点
cp encryption-config.yaml bootstrap-token.csv /etc/kubernetes/ssl
scp encryption-config.yaml bootstrap-token.csv 192.168.10.13:/etc/kubernetes/ssl
scp encryption-config.yaml bootstrap-token.csv 192.168.10.14:/etc/kubernetes/ssl

#6.创建kube-apiserver.service文件
cat > /etc/systemd/system/kube-apiserver.service << EOF
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
ExecStart=/usr/local/bin/kube-apiserver \
  --enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \
  --anonymous-auth=false \
  --experimental-encryption-provider-config=/etc/kubernetes/ssl/encryption-config.yaml \
  --advertise-address=0.0.0.0 \
  --bind-address=0.0.0.0 \
  --insecure-bind-address=127.0.0.1 \
  --secure-port=6443 \
  --insecure-port=0 \
  --authorization-mode=Node,RBAC \
  --runtime-config=api/all \
  --enable-bootstrap-token-auth \
  --service-cluster-ip-range=10.254.0.0/16 \
  --service-node-port-range=30000-32700 \
  --tls-cert-file=/etc/kubernetes/ssl/kubernetes.pem \
  --tls-private-key-file=/etc/kubernetes/ssl/kubernetes-key.pem \
  --client-ca-file=/etc/kubernetes/ssl/ca.pem \
  --kubelet-client-certificate=/etc/kubernetes/ssl/kubernetes.pem \
  --kubelet-client-key=/etc/kubernetes/ssl/kubernetes-key.pem \
  --service-account-key-file=/etc/kubernetes/ssl/ca-key.pem \
  --etcd-cafile=/etc/kubernetes/ssl/ca.pem \
  --etcd-certfile=/etc/kubernetes/ssl/kubernetes.pem \
  --etcd-keyfile=/etc/kubernetes/ssl/kubernetes-key.pem \
  --etcd-servers=https://192.168.10.12:2379,https://192.168.10.13:2379,https://192.168.10.14:2379 \
  --enable-swagger-ui=true \
  --allow-privileged=true \
  --apiserver-count=3 \
  --audit-log-maxage=30 \
  --audit-log-maxbackup=3 \
  --audit-log-maxsize=100 \
  --audit-log-path=/var/log/kubernetes/kube-apiserver-audit.log \
  --event-ttl=1h \
  --alsologtostderr=true \
  --logtostderr=false \
  --log-dir=/var/log/kubernetes \
  --v=2
Restart=on-failure
RestartSec=5
Type=notify
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF

mkdir -p /var/log/kubernetes		#创建日志目录然后拷贝到其他master
scp /etc/systemd/system/kube-apiserver.service 192.168.10.13:/etc/systemd/system/
scp /etc/systemd/system/kube-apiserver.service 192.168.10.14:/etc/systemd/system/

#7.启动服务

systemctl daemon-reload && systemctl enable kube-apiserver && systemctl start kube-apiserver && systemctl status kube-apiserver

#8.授予kubernetes证书访问kubelet api权限。在执行kubectl exec、run、logs 等命令时，apiserver会转发到kubelet。这里定义 RBAC规则，授权apiserver调用kubelet API。
kubectl create clusterrolebinding kube-apiserver:kubelet-apis --clusterrole=system:kubelet-api-admin --user kubernetes

#8.1预定义的ClusterRole system:kubelet-api-admin授予访问kubelet所有 API 的权限：
kubectl describe clusterrole system:kubelet-api-admin

#9.检查api-server和集群状态
netstat -ptln | grep kube-apiserver
tcp        0      0 192.168.10.12:6443      0.0.0.0:*               LISTEN      13000/kube-apiserve 

kubectl cluster-info
#显示如下
Kubernetes master is running at https://192.168.10.100:8443
To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.

kubectl get all --all-namespaces
#显示如下
NAMESPACE   NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
default     service/kubernetes   ClusterIP   10.254.0.1   <none>        443/TCP   12m

kubectl get componentstatuses
#显示如下,因scheduler和controller-manager还没有部署
NAME                 STATUS      MESSAGE                                ERROR
scheduler            Unhealthy   Get http://127.0.0.1:10251/healthz: dial tcp 127.0.0.1:10251: connect: connection refused   
controller-manager   Unhealthy   Get http://127.0.0.1:10252/healthz: dial tcp 127.0.0.1:10252: connect: connection refused   
etcd-2               Healthy     {"health":"true"}
etcd-1               Healthy     {"health":"true"} 
etcd-0               Healthy     {"health":"true"}   
```

#### 2.7.3、部署kube-controller-manager

该集群包含 3 个节点，启动后将通过竞争选举机制产生一个 leader 节点，其它节点为阻塞状态。当 leader 节点不可用后，剩余节点将再次进行选举产生新的 leader 节点，从而保证服务的可用性。

```bash
#1.创建CA证书
cd /root/ssl
cat > kube-controller-manager-csr.json << EOF
{
    "CN": "system:kube-controller-manager",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "hosts": [
      "127.0.0.1",
      "192.168.10.12",
      "192.168.10.13",
      "192.168.10.14"
    ],
    "names": [
      {
        "C": "CN",
        "ST": "ShangHai",
        "L": "ShangHai",
        "O": "system:kube-controller-manager",
        "OU": "System"
      }
    ]
}
EOF

#2.生成证书
cfssl gencert -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager

#3.将证书拷贝到其他master节点
cp kube-controller-manager*.pem /etc/kubernetes/ssl/
scp kube-controller-manager*.pem 192.168.10.13:/etc/kubernetes/ssl/
scp kube-controller-manager*.pem 192.168.10.14:/etc/kubernetes/ssl/

#4.创建kubeconfig文件
kubectl config set-cluster kubernetes \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://192.168.10.100:8443 \
  --kubeconfig=kube-controller-manager.kubeconfig

kubectl config set-credentials system:kube-controller-manager \
  --client-certificate=kube-controller-manager.pem \
  --client-key=kube-controller-manager-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-controller-manager.kubeconfig

kubectl config set-context system:kube-controller-manager \
  --cluster=kubernetes \
  --user=system:kube-controller-manager \
  --kubeconfig=kube-controller-manager.kubeconfig

kubectl config use-context system:kube-controller-manager --kubeconfig=kube-controller-manager.kubeconfig

#5.拷贝kube-controller-manager.kubeconfig到其他master节点
cp kube-controller-manager.kubeconfig /etc/kubernetes/ssl/
scp kube-controller-manager.kubeconfig 192.168.10.13:/etc/kubernetes/ssl/
scp kube-controller-manager.kubeconfig 192.168.10.14:/etc/kubernetes/ssl/

#6.创建kube-controller-manager.service文件
cat > /etc/systemd/system/kube-controller-manager.service  << EOF
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \
  --address=127.0.0.1 \
  --master=https://192.168.10.100:8443 \
  --kubeconfig=/etc/kubernetes/ssl/kube-controller-manager.kubeconfig \
  --allocate-node-cidrs=true \
  --authentication-kubeconfig=/etc/kubernetes/ssl/kube-controller-manager.kubeconfig \
  --service-cluster-ip-range=10.254.0.0/16 \
  --cluster-cidr=172.30.0.0/16 \
  --cluster-name=kubernetes \
  --cluster-signing-cert-file=/etc/kubernetes/ssl/ca.pem \
  --cluster-signing-key-file=/etc/kubernetes/ssl/ca-key.pem \
  --experimental-cluster-signing-duration=8760h \
  --leader-elect=true \
  --feature-gates=RotateKubeletServerCertificate=true \
  --controllers=*,bootstrapsigner,tokencleaner \
  --horizontal-pod-autoscaler-use-rest-clients=true \
  --horizontal-pod-autoscaler-sync-period=10s \
  --tls-cert-file=/etc/kubernetes/ssl/kube-controller-manager.pem \
  --tls-private-key-file=/etc/kubernetes/ssl/kube-controller-manager-key.pem \
  --service-account-private-key-file=/etc/kubernetes/ssl/ca-key.pem \
  --root-ca-file=/etc/kubernetes/ssl/ca.pem \
  --use-service-account-credentials=true \
  --alsologtostderr=true \
  --logtostderr=false \
  --log-dir=/var/log/kubernetes \
  --v=2
Restart=on
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

#7.拷贝到其他master节点，然后启动服务
scp /etc/systemd/system/kube-controller-manager.service 192.168.10.13:/etc/systemd/system/
scp /etc/systemd/system/kube-controller-manager.service 192.168.10.14:/etc/systemd/system/

systemctl daemon-reload && systemctl enable kube-controller-manager && systemctl start kube-controller-manager && systemctl status kube-controller-manager

#8.检查服务
netstat -lnpt|grep kube-controll
tcp        0      0 127.0.0.1:10252         0.0.0.0:*               LISTEN      14492/kube-controll 
tcp6       0      0 :::10257                :::*                    LISTEN      14492/kube-controll 

kubectl get cs
#显示如下
NAME                 STATUS      MESSAGE                                               ERROR
scheduler            Unhealthy   Get http://127.0.0.1:10251/healthz: dial tcp 127.0.0.1:10251: connect: connection refused 
controller-manager   Healthy     ok                               
etcd-1               Healthy     {"health":"true"}
etcd-2               Healthy     {"health":"true"}
etcd-0               Healthy     {"health":"true"}

#检查leader所在机器
kubectl get endpoints kube-controller-manager --namespace=kube-system  -o yaml
#显示如下,k8s-m12选为leader
apiVersion: v1
kind: Endpoints
metadata:
  annotations:
    control-plane.alpha.kubernetes.io/leader: '{"holderIdentity":"k8s-m12_6f9b09e6-995b-11e9-b2bf-000c29959a05","leaseDurationSeconds":15,"acquireTime":"2019-06-28T04:16:00Z","renewTime":"2019-06-28T04:21:32Z","leaderTransitions":0}'
  creationTimestamp: "2019-06-28T04:16:00Z"
  name: kube-controller-manager
  namespace: kube-system
  resourceVersion: "1481"
  selfLink: /api/v1/namespaces/kube-system/endpoints/kube-controller-manager
  uid: 6f9d838f-995b-11e9-9cb7-000c29959a05
```

[关于 controller 权限和 use-service-account-credentials 参数](https://github.com/kubernetes/kubernetes/issues/48208)
[kublet 认证和授权](https://kubernetes.io/docs/admin/)

#### 2.7.4、部署kube-scheduler

该集群包含 3 个节点，启动后将通过竞争选举机制产生一个 leader 节点，其它节点为阻塞状态。当 leader 节点不可用后，剩余节点将再次进行选举产生新的 leader 节点，从而保证服务的可用性

```bash
#1.创建CA证书
cd /root/ssl
cat > kube-scheduler-csr.json << EOF
{
    "CN": "system:kube-scheduler",
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
        "ST": "ShangHai",
        "L": "ShangHai",
        "O": "system:kube-scheduler",
        "OU": "System"
      }
    ]
}
EOF

#2.生成证书
cfssl gencert -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes kube-scheduler-csr.json | cfssljson -bare kube-scheduler

#3.创建kube-scheduler.kubeconfig文件
kubectl config set-cluster kubernetes \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://192.168.10.100:8443 \
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config set-credentials system:kube-scheduler \
  --client-certificate=kube-scheduler.pem \
  --client-key=kube-scheduler-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config set-context system:kube-scheduler \
  --cluster=kubernetes \
  --user=system:kube-scheduler \
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config use-context system:kube-scheduler --kubeconfig=kube-scheduler.kubeconfig

#4.拷贝kubeconfig到其他master节点
cp kube-scheduler.kubeconfig kube-scheduler*.pem /etc/kubernetes/ssl/
scp kube-scheduler.kubeconfig kube-scheduler*.pem 192.168.10.13:/etc/kubernetes/ssl/
scp kube-scheduler.kubeconfig kube-scheduler*.pem 192.168.10.14:/etc/kubernetes/ssl/

#5.创建kube-scheduler.service文件
cat > /etc/systemd/system/kube-scheduler.service << EOF
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-scheduler \
  --address=127.0.0.1 \
  --master=https://192.168.10.100:8443 \
  --kubeconfig=/etc/kubernetes/ssl/kube-scheduler.kubeconfig \
  --leader-elect=true \
  --alsologtostderr=true \
  --logtostderr=false \
  --log-dir=/var/log/kubernetes \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

#6.将kube-scheduler.service拷贝到其他master节点，然后启动服务
scp /etc/systemd/system/kube-scheduler.service 192.168.10.13:/etc/systemd/system
scp /etc/systemd/system/kube-scheduler.service 192.168.10.14:/etc/systemd/system

systemctl daemon-reload && systemctl enable kube-scheduler && systemctl start kube-scheduler && systemctl status kube-scheduler

#7.检查服务
netstat -lnpt|grep kube-sche
tcp        0      0 127.0.0.1:10251         0.0.0.0:*               LISTEN      15137/kube-schedule 
tcp6       0      0 :::10259                :::*                    LISTEN      15137/kube-schedule

kubectl get cs
#显示如下
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok                  
controller-manager   Healthy   ok                  
etcd-2               Healthy   {"health":"true"}   
etcd-1               Healthy   {"health":"true"}   
etcd-0               Healthy   {"health":"true"} 

kubectl get endpoints kube-scheduler --namespace=kube-system  -o yaml
#显示如下，k8s-m12选为leader
apiVersion: v1
kind: Endpoints
metadata:
  annotations:
    control-plane.alpha.kubernetes.io/leader: '{"holderIdentity":"k8s-m12_1c3f7882-995f-11e9-a5c1-000c29959a05","leaseDurationSeconds":15,"acquireTime":"2019-06-28T04:42:19Z","renewTime":"2019-06-28T04:45:18Z","leaderTransitions":0}'
  creationTimestamp: "2019-06-28T04:42:19Z"
  name: kube-scheduler
  namespace: kube-system
  resourceVersion: "2714"
  selfLink: /api/v1/namespaces/kube-system/endpoints/kube-scheduler
  uid: 1cda2b3a-995f-11e9-ac7d-000c2928fce6
```

#### 2.7.5、在所有master节点上查看功能是否正常

```bash
kubectl get componentstatuses
NAME                 STATUS    MESSAGE             ERROR
controller-manager   Healthy   ok                  
scheduler            Healthy   ok                  
etcd-2               Healthy   {"health":"true"}   
etcd-1               Healthy   {"health":"true"}   
etcd-0               Healthy   {"health":"true"}  
```

### 2.8、部署node

node节点运行kubelet  kube-proxy  docker  flannel。

#### 2.8.1、部署kubelet

kubelet运行在每个 worker 节点上，接收 kube-apiserver 发送的请求，管理 Pod 容器，执行交互式命令，如 exec、run、logs 等。kubelet 启动时自动向 kube-apiserver注册节点信息，内置的 cadvisor 统计和监控节点的资源使用情况。

```bash
#1.解压包，拷贝命令
tar -zxvf kubernetes-node-linux-amd64.tar.gz
cd /opt/kubernetes/node/bin
cp kubectl kubelet kube-proxy /usr/local/bin
scp kubectl kubelet kube-proxy 192.168.10.16:/usr/local/bin

#2.创建kubelet-bootstrap.kubeconfig文件（也是在12机器上执行）要创建3次分别是(k8s-m12，k8s-m13，k8s-m14)

#2.1.创建 token
cd /root/ssl
export BOOTSTRAP_TOKEN=$(kubeadm token create \
  --description kubelet-bootstrap-token \
  --groups system:bootstrappers:k8s-m12 \
  --kubeconfig ~/.kube/config)

#2.2.设置集群参数
kubectl config set-cluster kubernetes \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://192.168.10.100:8443 \
  --kubeconfig=kubelet-bootstrap-k8s-m12.kubeconfig

#2.3.设置客户端认证参数
kubectl config set-credentials kubelet-bootstrap \
  --token=${BOOTSTRAP_TOKEN} \
  --kubeconfig=kubelet-bootstrap-k8s-m12.kubeconfig

#2.4.设置上下文参数
kubectl config set-context default \
  --cluster=kubernetes \
  --user=kubelet-bootstrap \
  --kubeconfig=kubelet-bootstrap-k8s-m12.kubeconfig

#2.5.设置默认上下文
kubectl config use-context default --kubeconfig=kubelet-bootstrap-k8s-m12.kubeconfig

#3.查看kubeadm为各节点创建的token
kubeadm token list --kubeconfig ~/.kube/config
#显示如下
11rq5j.3f628cf6ura1hf2x   20h       2019-06-29T13:01:52+08:00   authentication,signing   kubelet-bootstrap-token   system:bootstrappers:k8s-m14
8zamvk.rfat8wyzh8311f89   20h       2019-06-29T12:59:26+08:00   authentication,signing   kubelet-bootstrap-token   system:bootstrappers:k8s-m12
lhxalz.busnf6izk82e0xqx   20h       2019-06-29T13:01:03+08:00   authentication,signing   kubelet-bootstrap-token   system:bootstrappers:k8s-m13

#3.1.r如果需要删除创建的token
kubeadm token --kubeconfig ~/.kube/config delete lhxalz.busnf6izk82e0xqx		
# 创建的token有效期为 1 天，超期后将不能再被使用，且会被kube-controller-manager的tokencleaner清理(如果启用该 controller 的话)。
# kube-apiserver接收kubelet的bootstrap token后，将请求的user设置为system:bootstrap；group设置为 system:bootstrappers；

#3.2.查看各token关联的secret
kubectl get secrets  -n kube-system		

#4.拷贝bootstrap kubeconfig文件到各个node机器上
scp kubelet-bootstrap-kube12.kubeconfig 192.168.10.15:/etc/kubernetes/ssl/kubelet-bootstrap.kubeconfig
scp kubelet-bootstrap-kube12.kubeconfig 192.168.10.16:/etc/kubernetes/ssl/kubelet-bootstrap.kubeconfig

#5.创建kubelet配置文件
cd /root/ssl
cat > kubelet.config.json <<EOF
{
  "kind": "KubeletConfiguration",
  "apiVersion": "kubelet.config.k8s.io/v1beta1",
  "authentication": {
    "x509": {
      "clientCAFile": "/etc/kubernetes/ssl/ca.pem"
    },
    "webhook": {
      "enabled": true,
      "cacheTTL": "2m0s"
    },
    "anonymous": {
      "enabled": false
    }
  },
  "authorization": {
    "mode": "Webhook",
    "webhook": {
      "cacheAuthorizedTTL": "5m0s",
      "cacheUnauthorizedTTL": "30s"
    }
  },
  "address": "0.0.0.0",
  "port": 10250,
  "readOnlyPort": 0,
  "cgroupDriver": "cgroupfs",
  "hairpinMode": "promiscuous-bridge",
  "serializeImagePulls": false,
  "featureGates": {
    "RotateKubeletClientCertificate": true,
    "RotateKubeletServerCertificate": true
  },
  "clusterDomain": "cluster.local",
  "clusterDNS": ["10.254.0.2"]
}
EOF

#6.拷贝到其他主机,注意，可以修改address为本机IP地址
cp kubelet.config.json /etc/kubernetes/ssl
scp kubelet.config.json 192.168.10.15:/etc/kubernetes/ssl
scp kubelet.config.json 192.168.10.16:/etc/kubernetes/ssl

#7.创建kubelet.service文件
mkdir -p /var/log/kubernetes && mkdir -p /var/lib/kubelet	#先创建目录

cat <<EOF > /etc/systemd/system/kubelet.service 
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=docker.service
Requires=docker.service

[Service]
WorkingDirectory=/var/lib/kubelet
ExecStart=/usr/local/bin/kubelet \
  --bootstrap-kubeconfig=/etc/kubernetes/ssl/kubelet-bootstrap.kubeconfig \
  --cert-dir=/etc/kubernetes/ssl \
  --network-plugin=cni \
  --cni-conf-dir=/etc/cni/net.d \
  --cni-bin-dir=/usr/local/bin/ \
  --fail-swap-on=false \
  --kubeconfig=/etc/kubernetes/ssl/kubelet.kubeconfig \
  --config=/etc/kubernetes/ssl/kubelet.config.json \
  --hostname-override=192.168.10.15 \
  --pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/google_containers/pause-amd64:3.1 \
  --allow-privileged=true \
  --alsologtostderr=true \
  --logtostderr=false \
  --cgroup-driver=systemd \
  --log-dir=/var/log/kubernetes \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

#拷贝到其他主机，注意修改hostname-override为本机IP地址

#8.Bootstrap Token Auth 和授予权限 ,需要先将bootstrap-token文件中的kubelet-bootstrap用户赋予system:node-bootstrapper角色，然后kubelet才有权限创建认证请求
kubectl create clusterrolebinding kubelet-bootstrap --clusterrole=system:node-bootstrapper --group=system:bootstrappers

#9.启动kubele服务
systemctl daemon-reload && systemctl enable kubelet && systemctl restart kubelet && systemctl status kubelet

#10.检查服务
netstat -lantp|grep kubelet
tcp        0      0 192.168.10.15:46936     192.168.10.100:8443     ESTABLISHED 15299/kubelet

#8.通过kubelet 的TLS 证书请求，kubelet 首次启动时向kube-apiserver 发送证书签名请求，必须通过后kubernetes 系统才会将该 Node 加入到集群。查看未授权的CSR 请求
kubectl get csr
NAME                                                   AGE   REQUESTOR                 CONDITION
node-csr-ZyWLfyY4nBb1GPBCCNGf2pCjbFKGHt04q50R1_3oprU   16m   system:bootstrap:rhwf4g   Pending
node-csr-hiZbOHizDYsE_n36kfuSxWTmUzobCEnCpIXfN54Lh6Y   18m   system:bootstrap:rhwf4g   Pending
```

**approve kubelet csr请求**

```bash
#1.手动approve csr请求(推荐自动的方式)
kubectl certificate approve node-csr-ZyWLfyY4nBb1GPBCCNGf2pCjbFKGHt04q50R1_3oprU #手动创建
#显示如下
certificatesigningrequest.certificates.k8s.io/node-csr-ZyWLfyY4nBb1GPBCCNGf2pCjbFKGHt04q50R1_3oprU approved

#1.1.查看Approve结果
kubectl describe csr node-csr-ZyWLfyY4nBb1GPBCCNGf2pCjbFKGHt04q50R1_3oprU	
#显示如下
Name:               node-csr-ZyWLfyY4nBb1GPBCCNGf2pCjbFKGHt04q50R1_3oprU
Labels:             <none>
Annotations:        <none>
CreationTimestamp:  Wed, 26 Jun 2019 15:12:40 +0800
Requesting User:    system:bootstrap:rhwf4g
Status:             Approved,Issued
Subject:
         Common Name:    system:node:192.168.10.16
         Serial Number:  
         Organization:   system:nodes
Events:  <none>

#1.2.特别多可以用这样的方式
kubectl get csr|grep 'Pending' | awk 'NR>0{print $1}'| xargs kubectl certificate approve
kubectl get csr|awk 'NR==3{print $1}'| xargs kubectl describe csr	#查看Approve结果

#2.自动approve csr请求(推荐),创建ClusterRoleBinding，分别用于自动 approve client、renew client、renew server 证书
cd /root/ssl
cat > csr-crb.yaml <<EOF
# Approve all CSRs for the group "system:bootstrappers"
 kind: ClusterRoleBinding
 apiVersion: rbac.authorization.k8s.io/v1
 metadata:
   name: auto-approve-csrs-for-group
 subjects:
 - kind: Group
   name: system:bootstrappers
   apiGroup: rbac.authorization.k8s.io
 roleRef:
   kind: ClusterRole
   name: system:certificates.k8s.io:certificatesigningrequests:nodeclient
   apiGroup: rbac.authorization.k8s.io
---
 # To let a node of the group "system:bootstrappers" renew its own credentials
 kind: ClusterRoleBinding
 apiVersion: rbac.authorization.k8s.io/v1
 metadata:
   name: node-client-cert-renewal
 subjects:
 - kind: Group
   name: system:bootstrappers
   apiGroup: rbac.authorization.k8s.io
 roleRef:
   kind: ClusterRole
   name: system:certificates.k8s.io:certificatesigningrequests:selfnodeclient
   apiGroup: rbac.authorization.k8s.io
---
# A ClusterRole which instructs the CSR approver to approve a node requesting a
# serving cert matching its client cert.
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: approve-node-server-renewal-csr
rules:
- apiGroups: ["certificates.k8s.io"]
  resources: ["certificatesigningrequests/selfnodeserver"]
  verbs: ["create"]
---
 # To let a node of the group "system:nodes" renew its own server credentials
 kind: ClusterRoleBinding
 apiVersion: rbac.authorization.k8s.io/v1
 metadata:
   name: node-server-cert-renewal
 subjects:
 - kind: Group
   name: system:nodes
   apiGroup: rbac.authorization.k8s.io
 roleRef:
   kind: ClusterRole
   name: approve-node-server-renewal-csr
   apiGroup: rbac.authorization.k8s.io
EOF

#3.拷贝到其他节点
cp csr-crb.yaml /etc/kubernetes/ssl
scp csr-crb.yaml 192.168.10.13:/etc/kubernetes/ssl
scp csr-crb.yaml 192.168.10.14:/etc/kubernetes/ssl

#4.生效配置
kubectl apply -f /etc/kubernetes/ssl/csr-crb.yaml

#5.验证
kubectl get csr		#等待一段时间，查看CSR都被自动approve
#显示如下
NAME                                                   AGE   REQUESTOR                 CONDITION
node-csr-cF4D5xoTEQCkK5QCsCAmsHGItlZ2cJ43RjkGXpM4BNw   38m   system:bootstrap:8zamvk   Approved,Issued
node-csr-lUIuS1_ggYM8Q95rgsUrBawzrsAXQ4QfYcP3BbPnWl8   36m   system:bootstrap:lhxalz   Approved,Issued


kubectl get --all-namespaces -o wide nodes		#所有节点均 ready
#显示如下
NAME            STATUS   ROLES    AGE     VERSION   INTERNAL-IP     EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION                CONTAINER-RUNTIME
192.168.10.15   Ready    <none>   5m33s   v1.14.3   192.168.10.15   <none>        CentOS Linux 7 (Core)   4.4.103-1.el7.elrepo.x86_64   docker://18.9.6
192.168.10.16   Ready    <none>   54s     v1.14.3   192.168.10.16   <none>        CentOS Linux 7 (Core)   4.4.103-1.el7.elrepo.x86_64   docker://18.9.6

kubectl get nodes
NAME            STATUS   ROLES    AGE     VERSION
192.168.10.15   Ready    <none>   6m55s   v1.14.3
192.168.10.16   Ready    <none>   2m16s   v1.14.3

netstat -lnpt|grep kubelet
tcp        0      0 127.0.0.1:10248         0.0.0.0:*               LISTEN      20302/kubelet       
tcp        0      0 192.168.10.15:10250     0.0.0.0:*               LISTEN      20302/kubelet       
tcp        0      0 127.0.0.1:37706         0.0.0.0:*               LISTEN      20302/kubelet       
tcp        0      0 192.168.10.15:60332     192.168.10.100:8443     ESTABLISHED 20302/kubelet 
#10248: healthz http 服务,10250; https API 服务；注意：未开启只读端口 10255；由于关闭了匿名认证，同时开启了 webhook 授权，所有访问 10250 端口 https API 的请求都需要被认证和授权。
```

**kublet api 认证和授权**

kublet的配置文件kubelet.config.json配置了如下认证参数：

- authentication.anonymous.enabled：设置为 false，不允许匿名访问 10250 端口；
- authentication.x509.clientCAFile：指定签名客户端证书的 CA 证书，开启 HTTPs 证书认证；
- authentication.webhook.enabled=true：开启 HTTPs bearer token 认证；

同时配置了如下授权参数：

- authroization.mode=Webhook：开启 RBAC 授权；

```bash
# kubelet 收到请求后，使用 clientCAFile 对证书签名进行认证，或者查询 bearer token 是否有效。如果两者都没通过，则拒绝请求，提示 Unauthorized
curl -s --cacert /etc/kubernetes/ssl/ca.pem https://127.0.0.1:10250/metrics
curl -s --cacert /etc/kubernetes/ssl/ca.pem -H "Authorization: Bearer 123456" https://192.168.10.15:10250/metrics

#通过认证后，kubelet 使用 SubjectAccessReview API 向 kube-apiserver 发送请求，查询证书或 token 对应的 user、group 是否有操作资源的权限(RBAC)；

#1.证书认证和授权

#权限不足的证书；
curl -s --cacert /etc/kubernetes/ssl/ca.pem --cert /etc/kubernetes/ssl/kube-controller-manager.pem --key /etc/kubernetes/ssl/kube-controller-manager-key.pem https://192.168.10.15:10250/metrics

#使用部署 kubectl 命令行工具时创建的、具有最高权限的 admin 证书；
curl -s --cacert /etc/kubernetes/ssl/ca.pem --cert /etc/kubernetes/ssl/admin.pem --key /etc/kubernetes/ssl/admin-key.pem https://192.168.10.15:10250/metrics|head

#2.bear token认证和授权：

# 创建一个ServiceAccount，将它和ClusterRole system:kubelet-api-admin绑定，从而具有调用kubelet API的权限：
kubectl create sa kubelet-api-test

kubectl create clusterrolebinding kubelet-api-test --clusterrole=system:kubelet-api-admin --serviceaccount=default:kubelet-api-test 
SECRET=$(kubectl get secrets | grep kubelet-api-test | awk '{print $1}')
TOKEN=$(kubectl describe secret ${SECRET} | grep -E '^token' | awk '{print $2}')
echo ${TOKEN}

curl -s --cacert /etc/kubernetes/ssl/ca.pem -H "Authorization: Bearer ${TOKEN}" https://192.168.10.15:10250/metrics|head

# cadvisor 和 metrics
# cadvisor 统计所在节点各容器的资源(CPU、内存、磁盘、网卡)使用情况，分别在自己的 http web 页面(4194 端口)和 10250 以 promehteus metrics 的形式输出。

# 浏览器访问 http://192.168.10.15:4194/containers/ 可以查看到 cadvisor 的监控页面：
# 浏览器访问 https://192.168.10.15:10250/metrics 和 https://192.168.10.15:10250/metrics/cadvisor 分别返回 kublet 和 cadvisor 的 metrics。
```

注意：kublet.config.json 设置 authentication.anonymous.enabled 为 false，不允许匿名证书访问 10250 的 https 服务；参考[A.浏览器访问kube-apiserver安全端口.md](https://github.com/opsnull/follow-me-install-kubernetes-cluster/blob/master/A.浏览器访问kube-apiserver安全端口.md)，创建和导入相关证书，然后访问上面的 10250 端口；

```bash
#1.需要安装jdk然后使用keytool工具
.\keytool -import -v -trustcacerts -alias appmanagement -file "E:\ca.pem" -storepass password -keystore cacerts
#2.然后在linux上执行
openssl pkcs12 -export -out admin.pfx -inkey admin-key.pem -in admin.pem -certfile ca.pem
#3.然后把证书导进去，就可以正常访问了
```

#### 2.8.2、部署kube-proxy

kube-proxy 运行在所有 worker 节点上，，它监听 apiserver 中 service 和 Endpoint 的变化情况，创建路由规则来进行服务负载均衡。

```bash
#1.创建CA证书
cd /root/ssl
cat > kube-proxy-csr.json <<EOF
{
  "CN": "system:kube-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "ShangHai",
      "L": "ShangHai",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF

#2.生成证书和私钥
cfssl gencert -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes  kube-proxy-csr.json | cfssljson -bare kube-proxy

#3.创建kubeconfig文件

#3.1.设置集群参数
kubectl config set-cluster kubernetes \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://192.168.10.100:8443 \
  --kubeconfig=kube-proxy.kubeconfig
#3.2.设置客户端认证参数
kubectl config set-credentials kube-proxy \
  --client-certificate=kube-proxy.pem \
  --client-key=kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig
#3.3.设置上下文参数
kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig
#3.4.设置默认上下文
kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig

#4.拷贝到其他节点
cp kube-proxy*.pem kube-proxy.kubeconfig /etc/kubernetes/ssl/
scp kube-proxy*.pem kube-proxy.kubeconfig 192.168.10.15:/etc/kubernetes/ssl/
scp kube-proxy*.pem kube-proxy.kubeconfig 192.168.10.16:/etc/kubernetes/ssl/

#5.创建kube-proxy配置文件
cd /root/ssl
cat >kube-proxy.config.yaml <<EOF
apiVersion: kubeproxy.config.k8s.io/v1alpha1
bindAddress: 192.168.10.15
clientConnection:
  kubeconfig: /etc/kubernetes/ssl/kube-proxy.kubeconfig
clusterCIDR: 172.30.0.0/16
healthzBindAddress: 192.168.10.15:10256
hostnameOverride: 192.168.10.15
kind: KubeProxyConfiguration
metricsBindAddress: 192.168.10.15:10249
mode: "ipvs"
EOF

#6.拷贝到其他节点
cp kube-proxy.config.yaml /etc/kubernetes/ssl/
scp kube-proxy.config.yaml 192.168.10.15:/etc/kubernetes/ssl/
scp kube-proxy.config.yaml 192.168.10.16:/etc/kubernetes/ssl/

#7.创建kube-proxy.service文件，然后拷贝到其他节点
cat << EOF > /etc/systemd/system/kube-proxy.service
[Unit]
Description=Kubernetes Kube-Proxy Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
WorkingDirectory=/var/lib/kube-proxy
ExecStart=/usr/local/bin/kube-proxy \
  --config=/etc/kubernetes/ssl/kube-proxy.config.yaml \
  --alsologtostderr=true \
  --logtostderr=false \
  --log-dir=/var/log/kubernetes/kube-proxy \
  --v=2
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF

#8.启动kube-proxy服务
mkdir -p /var/lib/kube-proxy && mkdir -p /var/log/kubernetes/kube-proxy
systemctl daemon-reload && systemctl enable kube-proxy && systemctl restart kube-proxy && systemctl status kube-proxy

netstat -lnpt|grep kube-proxy	#查看端口

ipvsadm -ln		#查看ipvs路由规则
#显示如下
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.254.0.1:443 rr
  -> 192.168.10.12:6443           Masq    1      0          0         
  -> 192.168.10.13:6443           Masq    1      0          0         
  -> 192.168.10.14:6443           Masq    1      0          0  

```

#### 2.8.3、验证集群功能

```bash
kubectl get nodes		#查看节点状态

# 1、创建nginx 测试文件
cat << EOF > nginx-web.yml 
apiVersion: v1
kind: Service
metadata:
  name: nginx-web
  labels:
    tier: frontend
spec:
  type: NodePort
  selector:
    tier: frontend
  ports:
  - name: http
    port: 80
    targetPort: 80
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-con
  labels:
    tier: frontend
spec:
  replicas: 3
  template:
    metadata:
      labels:
        tier: frontend
    spec:
      containers:
      - name: nginx-pod
        image: nginx
        ports:
        - containerPort: 80
EOF

#2.执行文件
kubectl create -f nginx-web.yml		
#显示已创建
service/nginx-web created
deployment.extensions/nginx-con created

#3.查看pod状态
kubectl get pod -o wide
#显示如下
NAME                         READY   STATUS    RESTARTS   AGE    IP            NODE            NOMINATED NODE   READINESS GATES
nginx-con-7dc84bdfb6-h6bt6   1/1     Running   0          105s   172.30.85.2   192.168.10.16   <none>           <none>
nginx-con-7dc84bdfb6-nt5qs   1/1     Running   0          105s   172.30.34.3   192.168.10.15   <none>           <none>
nginx-con-7dc84bdfb6-sfg87   1/1     Running   0          105s   172.30.34.2   192.168.10.15   <none>           <none>

#4.测试IP是否ping通
ping -c4 172.30.34.2
PING 172.30.34.2 (172.30.34.2) 56(84) bytes of data.
64 bytes from 172.30.34.2: icmp_seq=1 ttl=63 time=0.543 ms
64 bytes from 172.30.34.2: icmp_seq=2 ttl=63 time=0.684 ms
64 bytes from 172.30.34.2: icmp_seq=3 ttl=63 time=0.886 ms
64 bytes from 172.30.34.2: icmp_seq=4 ttl=63 time=0.817 ms

#5.查看server集群IP
kubectl get svc		#显示如下
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP   10.254.0.1       <none>        443/TCP        37h
nginx-web    NodePort    10.254.153.104   <none>        80:31808/TCP   4m19s
# 10.254.153.104是nginx的集群IP，代理前面3个pod，80是集群IP的端口31808是nodeport端口

#6.curl访问node_ip：nodeport
curl -I 192.168.10.15:31808		#状态200表示访问成功
HTTP/1.1 200 OK
Server: nginx/1.17.0
Date: Sat, 29 Jun 2019 05:03:15 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 21 May 2019 14:23:57 GMT
Connection: keep-alive
ETag: "5ce409fd-264"
Accept-Ranges: bytes

#7.在flannel网络主机上访问集群IP
ip add | grep 10.254
    inet 10.254.0.1/32 brd 10.254.0.1 scope global kube-ipvs0
    inet 10.254.153.104/32 brd 10.254.153.104 scope global kube-ipvs0

curl -I http://10.254.153.104:80	#返回如下
HTTP/1.1 200 OK
Server: nginx/1.17.0
Date: Sat, 29 Jun 2019 05:05:56 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 21 May 2019 14:23:57 GMT
Connection: keep-alive
ETag: "5ce409fd-264"
Accept-Ranges: bytes
```

### 2.9、部署集群插件

插件是集群的附件组件，丰富和完善了集群的功能

#### 2.9.1、部署coredns插件

```bash
#1.将kubernetes-server-linux-amd64.tar.gz解压后，再解压其中的 kubernetes-src.tar.gz 文件
tar -zxvf kubernetes-src.tar.gz -C src	 #coredns对应的目录是：cluster/addons/dns

#2.修改配置文件
cd src/cluster/addons/dns/coredns
cp coredns.yaml.base /etc/kubernetes/coredns.yaml

sed -i "s/__PILLAR__DNS__DOMAIN__/cluster.local/g" /etc/kubernetes/coredns.yaml
sed -i "s/__PILLAR__DNS__SERVER__/10.254.0.2/g" /etc/kubernetes/coredns.yaml

#3.创建coredns
kubectl create -f /etc/kubernetes/coredns.yaml

#4.检查codedns功能
kubectl -n kube-system get all -o wide
#显示如下
NAME                          READY   STATUS    RESTARTS   AGE
pod/coredns-8854569d4-5vshp   1/1     Running   0          58m
#
NAME               TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
service/kube-dns   ClusterIP   10.254.0.2   <none>        53/UDP,53/TCP,9153/TCP   81m
#
NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/coredns   1/1     1            1           58m
#
NAME                                DESIRED   CURRENT   READY   AGE
replicaset.apps/coredns-8854569d4   1         1         1       58m
#4.1
kubectl -n kube-system describe pod coredns
#4.2
kubectl -n kube-system logs coredns-8854569d4-5vshp

#5.使用容器验证
kubectl run dns-test --rm -it --image=alpine /bin/sh
#进入容器 ping 百度正常
ping www.baidu.com
PING www.baidu.com (182.61.200.6): 56 data bytes
64 bytes from 182.61.200.6: seq=0 ttl=127 time=41.546 ms
64 bytes from 182.61.200.6: seq=1 ttl=127 time=35.043 ms
64 bytes from 182.61.200.6: seq=2 ttl=127 time=38.977 ms
64 bytes from 182.61.200.6: seq=3 ttl=127 time=40.633 ms

#查看所有集群pod
kubectl get --all-namespaces pods

#6.如果遇到镜像下载不下来，可以修改文件
sed -i "s/k8s.gcr.io/coredns/g" /etc/kubernetes/coredns.yaml
```

#### 2.9.2、部署dashboard插件

 参考
https://github.com/kubernetes/dashboard/wiki/Access-control 
https://github.com/kubernetes/dashboard/issues/2558 
https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/

```bash
#1.将kubernetes-server-linux-amd64.tar.gz 解压后，再解压其中的 kubernetes-src.tar.gz 文件。dashboard 对应的目录是：cluster/addons/dashboard ，拷贝dashboard的文件

mkdir -p /etc/kubernetes/dashboard

cp -a /opt/kubernetes/src/cluster/addons/dashboard/{dashboard-configmap.yaml,dashboard-controller.yaml,dashboard-rbac.yaml,dashboard-secret.yaml,dashboard-service.yaml} /etc/kubernetes/dashboard

#2.修改配置文件
sed -i "s@image:.*@image: registry.cn-hangzhou.aliyuncs.com/google_containers/kubernetes-dashboard-amd64:v1.10.1@g" /etc/kubernetes/dashboard/dashboard-controller.yaml
sed -i "/spec/a\  type: NodePort" /etc/kubernetes/dashboard/dashboard-service.yaml
sed -i "/targetPort/a\    nodePort: 32700" /etc/kubernetes/dashboard/dashboard-service.yaml

#3.执行所有定义文件
kubectl create -f /etc/kubernetes/dashboard

#4.查看分配的NodePort
kubectl -n kube-system get all -o wide
#
NAME                                        READY   STATUS    RESTARTS   AGE
pod/coredns-8854569d4-5vshp                 1/1     Running   0          119m
pod/kubernetes-dashboard-7d5f7c58f5-mr8zn   1/1     Running   0          5m1s
#
NAME                           TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                  AGE
service/kube-dns               ClusterIP   10.254.0.2     <none>        53/UDP,53/TCP,9153/TCP   142m
service/kubernetes-dashboard   NodePort    10.254.63.16   <none>        443:32700/TCP            51s
#
NAME                                   READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/coredns                1/1     1            1           119m
deployment.apps/kubernetes-dashboard   1/1     1            1           5m4s
#
NAME                                              DESIRED   CURRENT   READY   AGE
replicaset.apps/coredns-8854569d4                 1         1         1       119m
replicaset.apps/kubernetes-dashboard-7d5f7c58f5   1         1         1       5m4s

kubectl -n kube-system describe pod kubernetes-dashboard

#NodePort映射到dasrd pod 443端口；
#dashboard的 --authentication-mode 支持 token、basic，默认为 token。如果使用 basic，则 kube-apiserver 必须配置 '--authorization-mode=ABAC' 和 '--basic-auth-file' 参数。

#5.查看 dashboard 支持的命令行参数
kubectl exec --namespace kube-system -it kubernetes-dashboard-7d5f7c58f5-mr8zn -- /dashboard --help

#6.访问dashboard
# 为了集群安全，从1.7开始，dashboard只允许通过https访问，如果使用kube proxy则必须监听localhost或 127.0.0.1，对于NodePort没有这个限制，但是仅建议在开发环境中使用。对于不满足这些条件的登录访问，在登录成功后浏览器不跳转，始终停在登录界面。
参考1：https://github.com/kubernetes/dashboard/wiki/Accessing-Dashboard---1.7.X-and-above
参考2：https://github.com/kubernetes/dashboard/issues/2540
# 三种访问 dashboard 的方式
# 通过NodePort访问dashboard：
# 通过kubectl proxy访问dashboard：
# 通过kube-apiserver访问dashboard；

#7.通过NodePort访问dashboard
# kubernetes-dashboard服务暴露了NodePort，可以使用http://NodeIP:NodePort地址访问dashboard；

#8.通过 kubectl proxy 访问 dashboard
#启动代理：
kubectl proxy --address='localhost' --port=8086 --accept-hosts='^*$'
# --address 必须为 localhost 或 127.0.0.1；
# 需要指定 --accept-hosts 选项，否则浏览器访问 dashboard 页面时提示 “Unauthorized”；
# 浏览器访问 URL：http://127.0.0.1:8086/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy

#9.通过 kube-apiserver 访问 dashboard
# 获取集群服务地址列表：
kubectl cluster-info
# 必须通过 kube-apiserver 的安全端口(https)访问 dashbaord，访问时浏览器需要使用自定义证书，否则会被 kube-apiserver 拒绝访问。
# 创建和导入自定义证书的步骤，参考：A.浏览器访问kube-apiserver安全端口
# 浏览器访问 URL：https://192.168.10.100:8443/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy

#10.创建登录 Dashboard 的 token 和 kubeconfig 配置文件
# 上面提到，Dashboard 默认只支持 token 认证，所以如果使用 KubeConfig 文件，需要在该文件中指定 token，不支持使用 client 证书认证。

# 创建登录 token，访问 dashboard时使用
kubectl create sa dashboard-admin -n kube-system
kubectl create clusterrolebinding dashboard-admin --clusterrole=cluster-admin --serviceaccount=kube-system:dashboard-admin
ADMIN_SECRET=$(kubectl get secrets -n kube-system | grep dashboard-admin | awk '{print $1}')
DASHBOARD_LOGIN_TOKEN=$(kubectl describe secret -n kube-system ${ADMIN_SECRET} | grep -E '^token' | awk '{print $2}')
echo ${DASHBOARD_LOGIN_TOKEN}

#使用输出的 token 登录 Dashboard。

#创建使用 token 的 KubeConfig 文件
cd /root/ssl
#设置集群参数
kubectl config set-cluster kubernetes \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://192.168.10.100:8443 \
  --kubeconfig=dashboard.kubeconfig

#设置客户端认证参数，使用上面创建的 Token
kubectl config set-credentials dashboard_user \
  --token=${DASHBOARD_LOGIN_TOKEN} \
  --kubeconfig=dashboard.kubeconfig

#设置上下文参数
kubectl config set-context default \
  --cluster=kubernetes \
  --user=dashboard_user \
  --kubeconfig=dashboard.kubeconfig

#设置默认上下文
kubectl config use-context default --kubeconfig=dashboard.kubeconfig

#生成的 dashboard.kubeconfig 登录 Dashboard。
#由于缺少 Heapster 插件，当前 dashboard 不能展示 Pod、Nodes 的 CPU、内存等统计数据和图表；
```

#### 2.9.3、部署heapster插件

Heapster是一个收集者，将每个Node上的cAdvisor的数据进行汇总，然后导到第三方工具(如InfluxDB)。Heapster 是通过调用 kubelet 的 http API 来获取 cAdvisor 的 metrics 数据的。由于 kublet 只在 10250 端口接收 https 请求，故需要修改 heapster 的 deployment 配置。同时，需要赋予 kube-system:heapster ServiceAccount 调用 kubelet API 的权限。

参考：配置 heapster：https://github.com/kubernetes/heapster/blob/master/docs/source-configuration.md

heapster下载地址：https://github.com/kubernetes-retired/heapster/releases

```bash
#1.解压heapster
mkdir /opt/heapster
tar -xzvf heapster-1.5.4.tar.gz -C /opt/heapster

#2.修改配置
mkdir -p /etc/kubernetes/heapster
cp -a /opt/heapster/deploy/kube-config/influxdb/{grafana.yaml,heapster.yaml,influxdb.yaml} /etc/kubernetes/heapster

sed -i "s@image:.*@image: registry.cn-hangzhou.aliyuncs.com/google_containers/heapster-grafana-amd64:v4.4.3@g" /etc/kubernetes/heapster/grafana.yaml

sed -i "67a\  type: NodePort" /etc/kubernetes/heapster/grafana.yaml

sed -i "/targetPort/a\    nodePort: 32699" /etc/kubernetes/heapster/grafana.yaml

sed -i "s@image:.*@image: registry.cn-hangzhou.aliyuncs.com/google_containers/heapster-amd64:v1.5.3@g" /etc/kubernetes/heapster/heapster.yaml

# 由于 kubelet 只在 10250 监听 https 请求，故添加相关参数；
sed -i "s@source=.*@source=kubernetes:https://kubernetes.default?kubeletHttps=true\&kubeletPort=10250@g" /etc/kubernetes/heapster/heapster.yaml

sed -i "s@image:.*@image: registry.cn-hangzhou.aliyuncs.com/google_containers/heapster-influxdb-amd64:v1.3.3@g" /etc/kubernetes/heapster/influxdb.yaml

# 将 serviceAccount kube-system:heapster 与 ClusterRole system:kubelet-api-admin 绑定，授予它调用 kubelet API 的权限；
cp -a /opt/heapster/deploy/kube-config/rbac/heapster-rbac.yaml /etc/kubernetes/heapster

cat > /etc/kubernetes/heapster/heapster-rbac.yaml <<EOF
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: heapster-kubelet-api
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kubelet-api-admin
subjects:
- kind: ServiceAccount
  name: heapster
  namespace: kube-system
EOF

#3.执行所有定义文件
kubectl create -f  /etc/kubernetes/heapster
kubectl apply -f  /etc/kubernetes/heapster/heapster-rbac.yaml

#4.检查执行结果
kubectl -n kube-system get all -o wide | grep -E 'heapster|monitoring'

kubectl -n kube-system describe pod heapster

kubectl -n kube-system describe pod monitoring

# 检查 kubernets dashboard 界面，可以正确显示各 Nodes、Pods 的 CPU、内存、负载等统计数据和图表：

kubectl -n kube-system get all -o wide

kubectl -n kube-system logs heapster-7bdc95b5cc-8h7zt

#5.访问 grafana,通过 NodePort 访问：
kubectl get svc -n kube-system|grep -E 'monitoring|heapster'
#显示如下,grafana 监听 NodePort 32699；
heapster               ClusterIP   10.254.159.62    <none>        80/TCP                   12m     k8s-app=heapster
monitoring-grafana     NodePort    10.254.167.38    <none>        80:32699/TCP             4m29s   k8s-app=grafana
monitoring-influxdb    ClusterIP   10.254.155.141   <none>        8086/TCP                 12m     k8s-app=influxdb

kubectl get pod -n kube-system -o wide |grep -E 'monitoring|heapster' 
#显示如下，然后浏览器访问 URL：http://192.168.10.16:32699/?orgId=1
heapster-7bdc95b5cc-8h7zt               1/1     Running   0          13m     172.30.34.4    192.168.10.15
monitoring-grafana-6cf5948cd4-rstxk     1/1     Running   0          5m      172.30.85.11   192.168.10.16
monitoring-influxdb-7d6c5fb944-qfd65    1/1     Running   0          13m     172.30.85.10   192.168.10.16

#6.通过 kube-apiserver 访问： 获取 monitoring-grafana 服务 URL：
kubectl cluster-info	
#查到浏览器访问URL：https://192.168.10.100:8443/api/v1/namespaces/kube-system/services/monitoring-grafana/proxy

#通过 kubectl proxy 访问：创建代理
kubectl proxy --address='192.168.10.16' --port=8086 --accept-hosts='^*$'
# 浏览器访问 URL：http://192.168.10.16:8086/api/v1/namespaces/kube-system/services/monitoring-grafana/proxy/?orgId=1
```

#### 2.9.4、部署EFK插件

```bash
#1.修改定义文件；EFK 对应的目录在压缩包kubernetes-src.tar.gz：里的cluster/addons/fluentd-elasticsearch/
mkdir -p /etc/kubernetes/efk
cp -a cluster/addons/fluentd-elasticsearch/*.yaml /etc/kubernetes/efk

sed -i "s@image:.*elasticsearch.*@image: registry.cn-hangzhou.aliyuncs.com/zhangbohan/elasticsearch:v6.6.1@g" /etc/kubernetes/efk/es-statefulset.yaml

sed -i "s@image:.*fluentd.*@image: registry.cn-hangzhou.aliyuncs.com/k8s-yun/fluentd-elasticsearch:v2.4.0@g" /etc/kubernetes/efk/fluentd-es-ds.yaml

#2.给 Node 设置标签
#DaemonSet fluentd-es只会调度到设置了标签 beta.kubernetes.io/fluentd-ds-ready=true的Node，需要在期望运行fluentd的 Node 上设置该标签；
kubectl get nodes

kubectl label nodes 192.168.10.15 beta.kubernetes.io/fluentd-ds-ready=true

kubectl describe nodes 192.168.10.15

kubectl get nodes --show-labels

#3.执行定义文件

kubectl create -f /etc/kubernetes/efk

#4.检查执行结果
kubectl -n kube-system get all  -o wide|grep -E 'elasticsearch|fluentd|kibana'

kubectl -n kube-system  get service |grep -E 'elasticsearch|kibana'

kubectl -n kube-system describe pods elasticsearch

kubectl -n kube-system describe pods fluentd

kubectl -n kube-system describe pods kibana

# kibana Pod 第一次启动时会用**较长时间(0-20分钟)**来优化和 Cache 状态页面，可以 tailf 该 Pod 的日志观察进度：
kubectl -n kube-system -f logs kibana-logging-7445dc9757-pvpcv

# 注意：只有当的 Kibana pod 启动完成后，才能查看 kibana dashboard，否则会提示 refuse。

#5.访问 kibana 通过 kube-apiserver 访问：
kubectl cluster-info|grep -E 'Elasticsearch|Kibana'
#查到浏览器访问 URL： https://192.168.10.100:8443/api/v1/namespaces/kube-system/services/kibana-logging/proxy

#通过 kubectl proxy 访问：
#创建代理
kubectl proxy --address='192.168.10.12' --port=8086 --accept-hosts='^*$'
#浏览器访问 URL：http://192.168.10.12:8086/api/v1/namespaces/kube-system/services/kibana-logging/proxy

#在 Settings -> Indices 页面创建一个 index（相当于 mysql 中的一个 database），选中 Index contains time-based events，使用默认的 logstash-* pattern，点击 Create ;
# 创建 Index 后，稍等几分钟就可以在 Discover 菜单下看到 ElasticSearch logging 中汇聚的日志；
```

#### 2.9.5、部署metrics-server插件

 参考：https://kubernetes.feisky.xyz/zh/addons/metrics.html
metrics-server RBAC：https://github.com/kubernetes-incubator/metrics-server/issues/40
metrics-server 参数：https://github.com/kubernetes-incubator/metrics-server/issues/25
https://kubernetes.io/docs/tasks/debug-application-cluster/core-metrics-pipeline/

```bash
#1.创建CA证书，创建 metrics-server 证书签名请求:
#注意： CN 名称为 aggregator，需要与 kube-apiserver 的 --requestheader-allowed-names 参数配置一致；
cd /root/ssl
cat > metrics-server-csr.json <<EOF
{
  "CN": "aggregator",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "ShangHai",
      "L": "ShangHai",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF

#3.生成 metrics-server 证书和私钥：

cfssl gencert -ca=ca.pem \
  -ca-key=ca-key.pem  \
  -config=ca-config.json  \
  -profile=kubernetes metrics-server-csr.json | cfssljson -bare metrics-server

#4.拷贝metrics-server到master节点：
cp metrics-server*.pem /etc/kubernetes/ssl
scp metrics-server*.pem 192.168.10.13:/etc/kubernetes/ssl
scp metrics-server*.pem 192.168.10.14:/etc/kubernetes/ssl

#5.修改kubernetes控制平面组件的配置以支持metrics-server
#kube-apiserver添加如下配置参数：
--requestheader-client-ca-file=/etc/kubernetes/ssl/ca.pem \
--requestheader-allowed-names="" \
--requestheader-extra-headers-prefix="X-Remote-Extra-" \
--requestheader-group-headers=X-Remote-Group \
--requestheader-username-headers=X-Remote-User \
--proxy-client-cert-file=/etc/kubernetes/ssl/metrics-server.pem \
--proxy-client-key-file=/etc/kubernetes/ssl/metrics-server-key.pem \
--enable-aggregator-routing=true \
--runtime-config=api/all=true
#--requestheader-XXX、--proxy-client-XXX 是 kube-apiserver 的 aggregator layer 相关的配置参数，metrics-server & HPA 需要使用；
# --requestheader-client-ca-file：用于签名 --proxy-client-cert-file 和 --proxy-client-key-file 指定的证书；在启用了 metric aggregator 时使用；
# 如果 --requestheader-allowed-names 不为空，则--proxy-client-cert-file 证书的 CN 必须位于 allowed-names 中，默认为 aggregator;
# 如果 kube-apiserver 机器没有运行 kube-proxy，则还需要添加 --enable-aggregator-routing=true 参数；

# 关于 --requestheader-XXX 相关参数，参考：
https://github.com/kubernetes-incubator/apiserver-builder/blob/master/docs/concepts/auth.md

https://docs.bitnami.com/kubernetes/how-to/configure-autoscaling-custom-metrics/
# 注意：requestheader-client-ca-file 指定的 CA 证书，必须具有 client auth and server auth；

# kube-controllr-manager添加如下配置参数：用于配置 HPA 控制器使用 REST 客户端获取 metrics 数据。
--horizontal-pod-autoscaler-use-rest-clients=true

#6.修改插件metrics-server-deployment配置文件,metrics-server插件位于kubernetes-src.tar.gz里的cluster/addons/metrics-server/ 目录下。
mkdir -p /etc/kubernetes/metrics-server
cp cluster/addons/metrics-server/metrics-server-deployment.yaml /etc/kubernetes/metrics-server

#修改配置文件
sed -i "s@image.*metrics-server-amd64.*@image: mirrorgooglecontainers/metrics-server-amd64:v0.3.1@g" /etc/kubernetes/metrics-server/metrics-server-deployment.yaml

sed -i "/ports:/i\        - --source=kubernetes.summary_api:https://kubernetes.default?kubeletHttps=true&kubeletPort=10250" /etc/kubernetes/metrics-server/metrics-server-deployment.yaml

sed -i "s@image.*addon-resizer.*@image: siriuszg/addon-resizer:1.8.4@g" /etc/kubernetes/metrics-server/metrics-server-deployment.yaml

# metrics-server 的参数格式与 heapster 类似。由于 kubelet 只在 10250 监听 https 请求，故添加相关参数；

# 新建一个ClusterRoleBindings 定义文件；授予kube-system:metrics-server ServiceAccount访问kubelet API的相关权限：
cat > /etc/kubernetes/metrics-server/auth-kubelet.yaml <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: metrics-server:system:kubelet-api-admin
  labels:
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kubelet-api-admin
subjects:
- kind: ServiceAccount
  name: metrics-server
  namespace: kube-system
EOF

#7.创建 metrics-server
kubectl create -f /etc/kubernetes/metrics-server/

#8.查看运行情况
kubectl get pods -n kube-system |grep metrics-server

kubectl get svc -n kube-system|grep metrics-server

kubectl -n kube-system describe pods metrics-server

#9.查看 metrcs-server 输出的 metrics

#metrics-server 输出的 APIs：https://github.com/kubernetes/community/blob/master/contributors/design-proposals/instrumentation/resource-metrics-api.md

# 通过 kube-apiserver 或 kubectl proxy 访问：
https://192.168.10.100:8443/apis/metrics.k8s.io/v1beta1/nodes
https://192.168.10.100:8443/apis/metrics.k8s.io/v1beta1/pods
https://192.168.10.100:8443/apis/metrics.k8s.io/v1beta1/namespace//pods/

# 直接使用 kubectl 命令访问：
kubectl get --raw apis/metrics.k8s.io/v1beta1/nodes kubectl get --raw apis/metrics.k8s.io/v1beta1/pods kubectl get --raw apis/metrics.k8s.io/v1beta1/nodes/ kubectl get --raw apis/metrics.k8s.io/v1beta1/namespace//pods/

kubectl get --raw "/apis/metrics.k8s.io/v1beta1" | jq .
{
  "kind": "APIResourceList",
  "apiVersion": "v1",
  "groupVersion": "metrics.k8s.io/v1beta1",
  "resources": [
    {
      "name": "nodes",
      "singularName": "",
      "namespaced": false,
      "kind": "NodeMetrics",
      "verbs": [
        "get",
        "list"
      ]
    },
    {
      "name": "pods",
      "singularName": "",
      "namespaced": true,
      "kind": "PodMetrics",
      "verbs": [
        "get",
        "list"
      ]
    }
  ]
}

kubectl get --raw "/apis/metrics.k8s.io/v1beta1/nodes" | jq .
{
  "kind": "NodeMetricsList",
  "apiVersion": "metrics.k8s.io/v1beta1",
  "metadata": {
    "selfLink": "/apis/metrics.k8s.io/v1beta1/nodes"
  },
  "items": [
    {
      "metadata": {
        "name": "k8s-03m",
        "selfLink": "/apis/metrics.k8s.io/v1beta1/nodes/k8s-03m",
        "creationTimestamp": "2018-06-16T10:24:03Z"
      },
      "timestamp": "2018-06-16T10:23:00Z",
      "window": "1m0s",
      "usage": {
        "cpu": "133m",
        "memory": "1115728Ki"
      }
    },
    {
      "metadata": {
        "name": "k8s-01m",
        "selfLink": "/apis/metrics.k8s.io/v1beta1/nodes/k8s-01m",
        "creationTimestamp": "2018-06-16T10:24:03Z"
      },
      "timestamp": "2018-06-16T10:23:00Z",
      "window": "1m0s",
      "usage": {
        "cpu": "221m",
        "memory": "6799908Ki"
      }
    },
    {
      "metadata": {
        "name": "k8s-02m",
        "selfLink": "/apis/metrics.k8s.io/v1beta1/nodes/k8s-02m",
        "creationTimestamp": "2018-06-16T10:24:03Z"
      },
      "timestamp": "2018-06-16T10:23:00Z",
      "window": "1m0s",
      "usage": {
        "cpu": "76m",
        "memory": "1130180Ki"
      }
    }
  ]
}
# /apis/metrics.k8s.io/v1beta1/nodes 和 /apis/metrics.k8s.io/v1beta1/pods 返回的 usage 包含 CPU 和 Memory；
```

### 2.10、各节点重启操作

```bash
#master节点服务
#启动
for master_ip in ${MASTER_IPS[@]};do
    echo ">>> ${master_ip}"

done
systemctl daemon-reload
systemctl restart etcd
systemctl restart keepalived
systemctl restart kube-apiserver
systemctl restart kube-controller-manager
systemctl restart kube-scheduler
systemctl restart haproxy
systemctl restart docker
#查看状态
systemctl daemon-reload
systemctl status etcd
systemctl status keepalived
systemctl status kube-apiserver
systemctl status kube-controller-manager
systemctl status kube-scheduler
#检查日志
journalctl -f -t etcd
journalctl -f -u etcd
journalctl -f -u keepalived
journalctl -f -u kube-apiserver
journalctl -f -u kube-controller-manager
journalctl -f -u kube-scheduler
#停止
systemctl daemon-reload
systemctl stop etcd
systemctl stop keepalived
systemctl stop kube-apiserver
systemctl stop kube-controller-manager
systemctl stop kube-scheduler

#node节点服务
#启动
systemctl daemon-reload
systemctl restart docker
systemctl restart kubelet
systemctl restart kube-proxy
systemctl restart flanneld

#查看状态
systemctl daemon-reload
systemctl status docker
systemctl status kubelet
systemctl status kube-proxy
systemctl status flanneld

#检查日志
journalctl -f -u docker
journalctl -f -u kubelet
journalctl -f -u kube-proxy
journalctl -f -u flanneld

#停止
systemctl daemon-reload
systemctl stop docker
systemctl stop kubelet
systemctl stop kube-proxy
systemctl stop flanneld

etcdctl \
  --ca-file=/etc/kubernetes/ssl/ca.pem \
  --cert-file=/etc/kubernetes/ssl/etcd.pem \
  --key-file=/etc/kubernetes/ssl/etcd-key.pem \
--endpoints=https://192.168.10.51:2379,https://192.168.10.52:2379,https://192.168.10.53:2379 \
cluster-health

etcdctl \
  --ca-file=/etc/kubernetes/ssl/ca.pem \
  --cert-file=/etc/kubernetes/ssl/etcd.pem \
  --key-file=/etc/kubernetes/ssl/etcd-key.pem \
--endpoints=https://192.168.10.51:2379,https://192.168.10.52:2379,https://192.168.10.53:2379 \
member list

#查看集群Pod网段(/16)
etcdctl \
  --ca-file=/etc/kubernetes/ssl/ca.pem \
  --cert-file=/etc/kubernetes/ssl/etcd.pem \
  --key-file=/etc/kubernetes/ssl/etcd-key.pem \
get /kubernetes/network/config
#查看已分配的Pod子网段列表(/24)
etcdctl \
  --ca-file=/etc/kubernetes/ssl/ca.pem \
  --cert-file=/etc/kubernetes/ssl/etcd.pem \
  --key-file=/etc/kubernetes/ssl/etcd-key.pem \
ls /kubernetes/network/subnets
* 查看某一Pod网段对应的flanneld进程监听的IP和网络参数
etcdctl \
  --ca-file=/etc/kubernetes/ssl/ca.pem \
  --cert-file=/etc/kubernetes/ssl/etcd.pem \
  --key-file=/etc/kubernetes/ssl/etcd-key.pem \
get /kubernetes/network/subnets/10.2.53.0-24

#清理集群
## 1、清理 Node 节点
* 停相关进程：
sudo systemctl stop kubelet kube-proxy flanneld docker
* 清理文件：
* umount kubelet 挂载的目录
mount | grep '/var/lib/kubelet'| awk '{print $3}'|xargs sudo umount
* 删除 kubelet 工作目录
sudo rm -rf /var/lib/kubelet
* 删除 flanneld 写入的网络配置文件
sudo rm -rf /var/run/flannel/
* 删除 docker 工作目录
sudo rm -rf /var/lib/docker
* 删除 docker 的一些运行文件
sudo rm -rf /var/run/docker/
* 清理 kube-proxy 和 docker 创建的 iptables：
sudo iptables -F && sudo iptables -X && sudo iptables -F -t nat && sudo iptables -X -t nat
* 删除 flanneld 和 docker 创建的网桥：
ip link del flannel.1
ip link del docker0
* 删除 systemd unit 文件
sudo rm -rf /etc/systemd/system/{kubelet,docker,flanneld}.service
* 删除证书文件
sudo rm -rf /etc/flanneld/cert /usr/local/k8s/cert
* 删除程序文件
sudo rm -rf /usr/local/k8s/bin/*

## 2、清理 Master 节点
* 停相关进程：
sudo systemctl stop kube-apiserver kube-controller-manager kube-scheduler
* 清理文件：
* 删除 kube-apiserver 工作目录
sudo rm -rf /var/run/kubernetes
* 删除 systemd unit 文件
sudo rm -rf /etc/systemd/system/{kube-apiserver,kube-controller-manager,kube-scheduler}.service
* 删除证书文件
sudo rm -rf /etc/flanneld/cert /usr/local/k8s/cert
* 删除程序文件
sudo rm -rf /usr/local/k8s/bin/{kube-apiserver,kube-controller-manager,kube-scheduler}

## 3、清理 etcd 集群
* 停相关进程：
sudo systemctl stop etcd
* 清理文件：
* 删除 etcd 的工作目录和数据目录
sudo rm -rf /var/lib/etcd
* 删除 systemd unit 文件
sudo rm -rf /etc/systemd/system/etcd.service
* 删除 x509 证书文件
sudo rm -rf /etc/etcd/cert/*
* 删除程序文件
sudo rm -rf /usr/local/k8s/bin/etcd
```

## 3、kubernetes核心服务配置说明



## 4、kubernetes的版本升级











