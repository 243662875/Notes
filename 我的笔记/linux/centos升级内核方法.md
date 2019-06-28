### 方法1：rpm安装方式

rpm安装包可以通过这个网站下载：

 [这个是CentOS6 x64](http://elrepo.org/linux/kernel/el6/x86_64/RPMS/)	:	http://elrepo.org/linux/kernel/el6/x86_64/RPMS/

 [这个是CentOS7 x64](http://elrepo.org/linux/kernel/el7/x86_64/RPMS/)	:	http://elrepo.org/linux/kernel/el7/x86_64/RPMS/

[RPM-GPG-KEY-elrepo.org](https://www.elrepo.org/RPM-GPG-KEY-elrepo.org)	:	https://www.elrepo.org/RPM-GPG-KEY-elrepo.org

Kernel-lt是长期维护版一般选这个，Kernel-ml是Linus亲自维护的主分支版本

下载所需的内核版本 

```bash
yum install kernel-lt-4.4.103-1.el7.elrepo.x86_64.rpm -y		#yum安装内核包

#如果没有外网先安装key，下载地址：https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
rpm --import RPM-GPG-KEY-elrepo.org
rpm -ivh kernel-lt-4.4.103-1.el7.elrepo.x86_64.rpm -y

#设置默认启动项，0是按menuentry顺序。比如要默认从第四个菜单项启动，数字改为3，若改为 saved，则默认为上次启动项。
sed -i "s/GRUB_DEFAULT=saved/GRUB_DEFAULT=0/g" /etc/default/grub	
grub2-mkconfig -o /boot/grub2/grub.cfg
reboot		#重启机器

uname -r	#重启后查看内核版本

#注：装新内核是占用/boot空间的，可以使用yum remove kernel-ml-4.10.11-1.el6.elrepo.x86_64方式清理不用的kernel
```

### 方法2：源码安装方式

[下载linux内核官网地址](<https://www.kernel.org/>)	:	<https://www.kernel.org/>

一般选着stable(稳定的内核版本)，mainline(开发的内核版本)，longterm(之前的内核版本)

tarball：完整的代码

pgp：验证签名

patch：基于上一个版本的补丁

**编码内核选择界面（生成.config的方法）：**

| **make help**         |支持更新模式进行配置|
| :-------------------- | ------------------------------------------------------------ |
| make menuconfig   | 基于curses的文本窗口界面|
| make gconfig    | 基于GTK(GOME)环境窗口界面                                    |
| make xconfig     | 基于QT(KDE) 环境的窗口界面                                   |
| make oldconfig  | 透过已经存在的./.config文件内容，并使用该文件内设定值为默认值，只将新版本核心的新功能列出让用户选择，可以简化核心功能挑选过程。对与升级内核很好选择。 |
| make defconfig    | 基于内核为目标平台执行提供的“默认”配置进行配置               |
| make allyesconfig | 所有选项均回答为”yes”                                        |
| make allnoconfig | 所有选项均回答为”no”                                         |
| make mrproper    | 清理所有编译生成的文件、 config及某些备份文件                |
| make clean       | 清理大多数编译生成的文件，但会保留config文件等               |
| make distclean   | mrproper、 patches以及编辑器备份文件                         |

``` bash
# 准备编译环境

yum -y groups install "Development Tools"
yum -y install ncurses-devel elfutils-libelf-devel bc openssl-devel.x86_64
tar xvf linux-4.14.12.tar.xz
cp /boot/config-3.10.0-693.el7.x86_64 /kernel/linux-4.14.12
cd /kernel/linux-4.14.12/
make menuconfig
make
make modules_install
make install
# 安装好之后，会在/boot内生成对应的内核相关文件，模块目录也会生成/lib/modules/4.14.12

cat grub2/grub.cfg |grep Core
# 然后重启系统，检查内核
```

**卸载内核**

```bash
#删除/lib/modules/目录下不需要的内核库文件
rm -rf 4.14.12
# 删除源码目录
# 删除/usr/src/linux/目录下不需要的内核源码
# 删除/boot目录下启动的内核和内核映像文件
rm *-4*
rm: remove regular file ‘initramfs-4.14.12hunk-2018-1.0.img’? y
rm: remove regular file ‘initramfs-4.14.12hunk-2018-1.0.img.gz’? y
rm: remove regular file ‘System.map-4.14.12hunk-2018-1.0’? y
rm: remove regular file ‘vmlinuz-4.14.12hunk-2018-1.0’? y

#更改grub的配置文件，删除不需要的内核启动列表
vim /boot/grub2/grub.cfg
```

