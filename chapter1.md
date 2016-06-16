# First Chapter
# Cobbler自动化安装


[Cobbler官网](http://cobbler.github.io)

![](/Users/wanyongzhen/Library/Containers/com.tencent.qq/Data/Library/Application Support/QQ/Users/1041282946/QQ/Temp.db/D673EB33-01F1-4571-8236-77E70B155755.png)

## Cobbler介绍
Cobbler是一个Linux服务器安装的服务，可以通过`网络启动(PXE)`的方式来快速安装、重装物理服务器和虚拟机，同时还可以管理DHCP、DNS等，是用来实现运维自动化的必备神器。Cobbler可用使用命令行方式管理，也提供了基于Web的界面管理工具(cobbler-web)，还提供了API接口，可用方便二次开发使用。Cobbler是较早前的kickstart的升级版，优点是比较`容易配置`，还自带`web界面易于管理`。Cobbler内置了一个轻量级配置管理系统，但它也支持和其他配置管理系统集成，如puppet，暂时不支持saltstack。  

## 环境部署准备

```
[root@linux-node1 ~]# cat /etc/redhat-release 
CentOS Linux release 7.2.1511 (Core) 
[root@linux-node1 ~]# getenforce 
Disabled
[root@linux-node1 ~]# systemctl status firewalld.service
firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; disabled; vendor preset: enabled)
   Active: inactive (dead)
[root@linux-node1 ~]# hostname
linux-node1.example.com
[root@linux-node1 ~]# ifconfig eth0|awk -F '[ :]+' 'NR==2{print $3}'   
192.168.56.11
```
## 安装配置Cobbler

```
[root@linux-node1 ~]# yum install cobbler cobbler-web dhcp tftp-server pykickstart httpd -y
[root@linux-node1 ~]# systemctl restart httpd
[root@linux-node1 ~]# systemctl restart cobblerd
[root@linux-node1 ~]# cobbler check
The following are potential configuration items that you may want to fix:

1 : The 'server' field in /etc/cobbler/settings must be set to something other than localhost, or kickstarting features will not work.  This should be a resolvable hostname or IP for the boot server as reachable by all machines that will use it.
2 : For PXE to be functional, the 'next_server' field in /etc/cobbler/settings must be set to something other than 127.0.0.1, and should match the IP of the boot server on the PXE network.
3 : change 'disable' to 'no' in /etc/xinetd.d/tftp
4 : some network boot-loaders are missing from /var/lib/cobbler/loaders, you may run 'cobbler get-loaders' to download them, or, if you only want to handle x86/x86_64 netbooting, you may ensure that you have installed a *recent* version of the syslinux package installed and can ignore this message entirely.  Files in this directory, should you want to support all architectures, should include pxelinux.0, menu.c32, elilo.efi, and yaboot. The 'cobbler get-loaders' command is the easiest way to resolve these requirements.
5 : enable and start rsyncd.service with systemctl
6 : debmirror package is not installed, it will be required to manage debian deployments and repositories
7 : The default password used by the sample templates for newly installed machines (default_password_crypted in /etc/cobbler/settings) is still set to 'cobbler' and should be changed, try: "openssl passwd -1 -salt 'random-phrase-here' 'your-password-here'" to generate new one
8 : fencing tools were not found, and are required to use the (optional) power management features. install cman or fence-agents to use them

Restart cobblerd and then run 'cobbler sync' to apply changes.
```
看上面的结果，一个一个解决  
第1、2、6个问题，顺便修改其他功能：

```
[root@linux-node1 ~]# sed -i 's/server: 127.0.0.1/server: 192.168.56.11/' /etc/cobbler/settings   
[root@linux-node1 ~]# sed -i 's/manage_dhcp: 0/manage_dhcp: 1/' /etc/cobbler/settings
[root@linux-node1 ~]# sed -i 's/pxe_just_once: 0/pxe_just_once: 1/' /etc/cobbler/settings
[root@linux-node1 ~]# openssl passwd -1 -salt '91guoxin' '123456'
$1$91guoxin$prqpocNRfiZKPDgyIW4851
[root@linux-node1 ~]# vim /etc/cobbler/settings 
default_password_crypted: "$1$91guoxin$prqpocNRfiZKPDgyIW4851"
```
第3个问题：

```
[root@linux-node1 ~]# cobbler get-loaders
[root@linux-node1 ~]# ls /var/lib/cobbler/loaders/
COPYING.elilo     COPYING.yaboot  grub-x86_64.efi  menu.c32    README
COPYING.syslinux  elilo-ia64.efi  grub-x86.efi     pxelinux.0  yaboot
[root@linux-node1 ~]# vim /etc/xinetd.d/tftp 
disable                 = no
[root@linux-node1 ~]# systemctl restart tftp
[root@linux-node1 ~]# systemctl restart cobblerd
```
修改Cobbler的dhcp模板，不要直接修改dhcp本身的配置文件，因为Cobbler会覆盖

```
[root@linux-node1 ~]# vim /etc/cobbler/dhcp.template 
subnet 192.168.56.0 netmask 255.255.255.0 {
     option routers             192.168.56.2;
     option domain-name-servers 192.168.56.2;
     option subnet-mask         255.255.255.0;
     range dynamic-bootp        192.168.56.100 192.168.56.254;
```
执行`cobbler sync`

```
[root@linux-node1 ~]# cobbler sync
task started: 2016-05-23_080450_sync
task started (id=Sync, time=Mon May 23 08:04:50 2016)
running pre-sync triggers
cleaning trees
removing: /var/lib/tftpboot/grub/images
copying bootloaders
trying hardlink /var/lib/cobbler/loaders/pxelinux.0 -> /var/lib/tftpboot/pxelinux.0
trying hardlink /var/lib/cobbler/loaders/menu.c32 -> /var/lib/tftpboot/menu.c32
trying hardlink /var/lib/cobbler/loaders/yaboot -> /var/lib/tftpboot/yaboot
trying hardlink /usr/share/syslinux/memdisk -> /var/lib/tftpboot/memdisk
trying hardlink /var/lib/cobbler/loaders/grub-x86.efi -> /var/lib/tftpboot/grub/grub-x86.efi
trying hardlink /var/lib/cobbler/loaders/grub-x86_64.efi -> /var/lib/tftpboot/grub/grub-x86_64.efi
copying distros to tftpboot
copying images
generating PXE configuration files
generating PXE menu structure
rendering DHCP files
generating /etc/dhcp/dhcpd.conf
rendering TFTPD files
generating /etc/xinetd.d/tftp
cleaning link caches
running post-sync triggers
running python triggers from /var/lib/cobbler/triggers/sync/post/*
running python trigger cobbler.modules.sync_post_restart_services
running: dhcpd -t -q
received on stdout: 
received on stderr: 
running: service dhcpd restart
received on stdout: 
received on stderr: Redirecting to /bin/systemctl restart  dhcpd.service

running shell triggers from /var/lib/cobbler/triggers/sync/post/*
running python triggers from /var/lib/cobbler/triggers/change/*
running python trigger cobbler.modules.scm_track
running shell triggers from /var/lib/cobbler/triggers/change/*
*** TASK COMPLETE ***
```
再次执行`cobbler check`

```
[root@linux-node1 ~]# cobbler check
The following are potential configuration items that you may want to fix:

1 : enable and start rsyncd.service with systemctl
2 : debmirror package is not installed, it will be required to manage debian deployments and repositories
3 : fencing tools were not found, and are required to use the (optional) power management features. install cman or fence-agents to use them

Restart cobblerd and then run 'cobbler sync' to apply changes.
```
剩余三个问题： 
  
* 1.可用不用理会，因为我们不用rsync同步ISO  
* 2.和debian系统相关，不需要  
* 3.fence设备相关，不需要

设置开机自启动：

```
[root@linux-node1 ~]# systemctl enable httpd.service
[root@linux-node1 ~]# systemctl enable tftp.service
[root@linux-node1 ~]# systemctl enable cobblerd.service
[root@linux-node1 ~]# systemctl enable dhcpd.service
```
## Cobbler的命令行管理

查看命令帮助

```
[root@linux-node1 ~]# cobbler
usage
=====
cobbler <distro|profile|system|repo|image|mgmtclass|package|file> ... 
        [add|edit|copy|getks*|list|remove|rename|report] [options|--help]
cobbler <aclsetup|buildiso|import|list|replicate|report|reposync|sync|validateks|version|signature|get-loaders|hardlink> [options|--help]
[root@linux-node1 ~]# cobbler import --help
Usage: cobbler [options]

Options:
  -h, --help            show this help message and exit
  --arch=ARCH           OS architecture being imported
  --breed=BREED         the breed being imported
  --os-version=OS_VERSION
                        the version being imported
  --path=PATH           local path or rsync location
  --name=NAME           name, ex 'RHEL-5'
  --available-as=AVAILABLE_AS
                        tree is here, don't mirror
  --kickstart=KICKSTART_FILE
                        assign this kickstart file
  --rsync-flags=RSYNC_FLAGS
                        pass additional flags to rsync
```
Cobbler命令小结：

```
cobbler check           	# 核对当前设置是否有问题    
cobbler list				# 列出所有的cobbler元素  
cobbler report				# 列出元素的详细信息  
cobbler sync				# 同步配置到数据目录，更改配置最好都要执行一下  
cobbler reposync 			# 同步yum仓库  
cobbler distro				# 查看导入的发行版系统信息  
cobbler system				# 查看添加的系统信息  
cobbler profile				# 查看配置信息
```
导入镜像

```
[root@linux-node1 ~]# mount /dev/cdrom /mnt/
mount: /dev/sr0 is write-protected, mounting read-only
# 挂载CentOS 7.2的系统镜像，光盘挂载的方式传输效率较低，此处仅为实验
[root@linux-node1 ~]# cobbler import --path=/mnt/ --name=CentOS-7.2 --arch=x86_64
task started: 2016-05-23_082109_import
task started (id=Media import, time=Mon May 23 08:21:09 2016)
Found a candidate signature: breed=redhat, version=rhel6
Found a candidate signature: breed=redhat, version=rhel7
Found a matching signature: breed=redhat, version=rhel7
Adding distros from path /var/www/cobbler/ks_mirror/CentOS-7.2-x86_64:
creating new distro: CentOS-7.2-x86_64
trying symlink: /var/www/cobbler/ks_mirror/CentOS-7.2-x86_64 -> /var/www/cobbler/links/CentOS-7.2-x86_64
creating new profile: CentOS-7.2-x86_64
associating repos
checking for rsync repo(s)
checking for rhn repo(s)
checking for yum repo(s)
starting descent into /var/www/cobbler/ks_mirror/CentOS-7.2-x86_64 for CentOS-7.2-x86_64
processing repo at : /var/www/cobbler/ks_mirror/CentOS-7.2-x86_64
need to process repo/comps: /var/www/cobbler/ks_mirror/CentOS-7.2-x86_64
looking for /var/www/cobbler/ks_mirror/CentOS-7.2-x86_64/repodata/*comps*.xml
Keeping repodata as-is :/var/www/cobbler/ks_mirror/CentOS-7.2-x86_64/repodata
*** TASK COMPLETE ***
--path 镜像路径
--name 为安装源定义一个名字
--arch 执行安装源是32位、64位、ia64，目前支持的选项有：x86 | x86_64 | ia64
[root@linux-node1 ~]# cobbler distro list
   CentOS-7.2-x86_64
[root@linux-node1 ~]# ls /var/www/cobbler/ks_mirror/
CentOS-7.2-x86_64  config
# 镜像存放目录，Cobbler会将镜像中的所有安装文件拷贝到本地一份，放在/var/www/cobbler/ks_mirror下的CentOS-7.2-x86_64目录下
```
指定ks.cfg文件及调整内核参数

```
[root@linux-node1 ~]# ls /var/lib/cobbler/kickstarts/
default.ks    esxi5-ks.cfg      legacy.ks     sample_autoyast.xml  sample_esx4.ks   sample_esxi5.ks  sample_old.seed
esxi4-ks.cfg  install_profiles  pxerescue.ks  sample_end.ks        sample_esxi4.ks  sample.ks        sample.seed
# 自带很多
[root@linux-node1 ~]# cd /var/lib/cobbler/kickstarts/
[root@linux-node1 kickstarts]# rz -y
rz waiting to receive.
Starting zmodem transfer.  Press Ctrl+C to cancel.
Transferring Cobbler-CentOS-7.1-x86_64.cfg...
  100%       1 KB       1 KB/sec    00:00:01       0 Errors 
# 上传准备好的ks文件
[root@linux-node1 kickstarts]# mv Cobbler-CentOS-7.1-x86_64.cfg CentOS-7.2-x86_64.cfg 
[root@linux-node1 kickstarts]# cobbler list
distros:
   CentOS-7.2-x86_64

profiles:
   CentOS-7.2-x86_64

systems:

repos:

images:

mgmtclasses:

packages:

files:
[root@linux-node1 kickstarts]# cobbler distro report --name=CentOS-7.2-X86_64
Name                           : CentOS-7.2-x86_64
Architecture                   : x86_64
TFTP Boot Files                : {}
Breed                          : redhat
Comment                        : 
Fetchable Files                : {}
Initrd                         : /var/www/cobbler/ks_mirror/CentOS-7.2-x86_64/images/pxeboot/initrd.img
Kernel                         : /var/www/cobbler/ks_mirror/CentOS-7.2-x86_64/images/pxeboot/vmlinuz
Kernel Options                 : {}
Kernel Options (Post Install)  : {}
Kickstart Metadata             : {'tree': 'http://@@http_server@@/cblr/links/CentOS-7.2-x86_64'}
Management Classes             : []
OS Version                     : rhel7
Owners                         : ['admin']
Red Hat Management Key         : <<inherit>>
Red Hat Management Server      : <<inherit>>
Template Files                 : {}
```
编辑profile，修改关联的ks文件

```
[root@linux-node1 kickstarts]# cobbler profile edit --name=CentOS-7.2-x86_64 --kickstart=/var/lib/cobbler/kickstarts/CentOS-7.2-x86_64.cfg
```
修改内核参数

```
cobbler profile edit --name=CentOS-7.2-x86_64 --kopts='net.ifnames=0 biosdevname=0'

```

安装系统  
新建一台虚拟机：
![](/Users/wanyongzhen/Library/Containers/com.tencent.qq/Data/Library/Application Support/QQ/Users/1041282946/QQ/Temp.db/0F2AFC36-45C4-4E2E-AEE2-1F416772084D.png)

修改Cobbler提示

```
[root@linux-node1 ~]# vim /etc/cobbler/pxe/pxedefault.template 
MENU TITLE Cobbler | http://www.wanyuetian.com
[root@linux-node1 ~]# cobbler sync
```
## 定制化安装
可能有人想kickstart怎样能够指定某台服务器使用指定ks文件，kickstart实现这功能可能比较复杂，但是Cobbler就很简单了。区分一台服务器的最简单的方法就是物理MAC地址。物理服务器的MAC地址在服务器上的标签上写了。

```
[root@linux-node1 ~]# cobbler system add --name=cobblertest --mac=00:0C:29:76:73:2D --profile=CentOS-7.2-x86_64 --ip-address=192.168.56.100 --subnet=255.255.255.0 --gateway=192.168.56.2 --interface=eth0 --static=1 --hostname=cobblertest.example.com --name-servers="192.168.56.2"
[root@linux-node1 ~]# cobbler sync
```
再次开机，如下图所示：

![](/Users/wanyongzhen/Library/Containers/com.tencent.qq/Data/Library/Application Support/QQ/Users/1041282946/QQ/Temp.db/C544F8BD-0692-43B0-8E6C-E6B558CFF9E3.png)


## Cobblerd的web管理界面

[https://192.168.56.11/cobbler_web](https://192.168.56.11/cobbler_web)  
默认用户名：`cobbler`  
默认密码：`cobbler`

```
/etc/cobbler/users.conf       # Web服务授权配置文件
/etc/cobbler/users.digest     # 用于web访问的用户名密码配置文件
[root@cobbler loaders]# cat /etc/cobbler/users.digest
cobbler:Cobbler:a2d6bae81669d707b72c0bd9806e01f3
# 设置Cobbler web用户登陆密码
# 在Cobbler组添加cobbler用户，提示输入2遍密码确认
[root@linux-node1 ~]# htdigest /etc/cobbler/users.digest "Cobbler" cobbler
Changing password for user cobbler in realm Cobbler
New password: 
Re-type new password:
[root@linux-node1 ~]# cobbler sync
[root@linux-node1 ~]# systemctl restart httpd
[root@linux-node1 ~]# systemctl restart cobblerd
```




￼




￼
￼



















