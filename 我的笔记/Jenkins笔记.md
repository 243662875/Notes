[TOC]



# Jenkins基础

jenkins官网地址为：[http://jenkins-ci.org/](http://mirrors.jenkins-ci.org/war/latest/jenkins.war)  本文参考：https://jenkins.io/zh/doc/ 

jenkins本身是用java语言开发的，所以安装jenkins的机器至少要有jdk(Java 8以上版本)。

jenkins是一个广泛用于持续构建的可视化web工具。可以很好的支持各种语言（比如：java, c#, php等）的项目构建，也完全兼容Ant、maven、gradle等多种第三方构建工具，同时跟svn、git能无缝集成，也支持直接与知名源代码托管网站，比如github、bitbucket直接集成。

主要用于：持续、自动地构建/测试软件项目。监控一些定时执行的任务。

## 1.1、安装部署

[下载Jenkins地址](http://mirrors.jenkins-ci.org/war/latest/)：http://mirrors.jenkins.io/war-stable/latest/jenkins.war

下载最新war包后，使用命令 java -jar jenkins.war --httpPort=8080 就可以启动，然后访问 http://ip:8080 就可以了

**我们把Jenkins放到tomcat里运行

jdk下载地址：https://download.oracle.com/otn-pub/java/jdk/12.0.1+12/69cfe15208a647278a19ef0990eea691/jdk-12.0.1_linux-x64_bin.rpm

```bash
#1.安装jdk
rpm -ivh jdk-12.0.1_linux-x64_bin.rpm

#2.配置环境变量
cat <<EOF>> /etc/profile
export JAVA_HOME=/usr/java/jdk-12.0.1
export PATH=\$JAVA_HOME/bin:\$PATH
export CLASSPATH=.:\$JAVA_HOME/lib/dt.jar:\$JAVA_HOME/lib/tools.jar
EOF

#3.重新加载环境变量
source /etc/profile

#4.解压tomcat
tar -zxvf apache-tomcat-9.0.21.tar.gz -C apache-tomcat-jenkins

#5.拷贝项目文件到tomcat工作目录
cp jenkins.war apache-tomcat-jenkins/webapps/

#6.密码存放在/root/.jenkins/secrets/initialAdminPassword目录
#7.浏览器访问Jenkins http://ip:8080/jenkins,等待准备就绪就可以使用密码登陆，接下来就需要安装插件

```





