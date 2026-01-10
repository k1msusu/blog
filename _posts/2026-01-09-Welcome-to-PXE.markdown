---
layout: post
title: "Welcome to PXE!!!"
date:	2026-01-09 22:09:30
categories: jekyll update
---
## 批量下发机房里的新系统

```
# 首先由 HTCP 动态分发 IP ，指定引导 pxelinux.0
# 把 xinetd 作为守护进程
# 由 Tftp 批量传输引导文件
# 由 http 批量部署大型镜像文件
# 关闭防火墙，

```

### 一、安装 PXE 启动所需要的依赖

```
# 在 centos7 中 dhcp 包含 dhcp-server, tftp-server 和 xinetd 是从属关系
# 安装之间设置网络为 static 而非 dhcp ，防止频繁更换 ip
# systemctl-config-kickstart 需要具有图形界面的系统
yum  install -y dhcp xinetd  tftp-server system-config-kickstart syslinux httpd
```

### 二、配置网络和挂载数据

```
vim /etc/dhcp/dhcp.conf # 找到模板文件并修改内容
# A slightly different configuration for an internal subnet.
# 192.168.100.138 是本机 IP
subnet 192.168.100.0 netmask 255.255.255.0 { 
  range 192.168.100.150 192.168.100.200; 
  option subnet-mask 255.255.255.0;  #  获取 Ip 后，识别子网掩码
  option domain-name-servers 192.168.100.254;  # DNS 配置
  option domain-name "internal.example.org";
  option routers 192.168.100.2;  # 网关
  option broadcast-address 192.168.100.255;
  default-lease-time 600;
  max-lease-time 7200;
  allow booting;  # pxe 专属配置
  allow bootp;  # pxe 专属配置
  next-server 192.168.100.138;  # 本机静态IP，TFTP服务地址
  filename "pxelinux.0";
}

```

```
systemctl restart httpd
vim /etc/yum.repos.d/pxe.repo
# 加载源文件，防止出现意外
[development]
name=pxe
baseurl=http://192.168.100.133/pub
enabled=1
gpgcheck=0

# 挂载镜像
mkdir -p /var/www/html/pub
mount /dev/cdrom /var/www/html/pub/

# 修改 tftp 配置
vim /etc/xinetd.d/tftp
systemctl restart xinetd
systemctl restart tftp
systemctl restart tftp.socket
```

### 三、建立系统引导文件

```
cp /usr/share/syslinux/pxelinux.0 /var/lib/tftpboot
mkdir -p /var/lib/tftpboot/pxelinux.cfg
cp /var/www/html/pub/isolinux/isolinux.cfg /var/lib/tftpboot/pxelinux.cfg/default
# 更改系统启动 ks 文件路径
vim /var/lib/tftpboot/pxelinux.cfg/default/isolinux.cfg
=================# 更改 label linux 内容
label linux
  menu label ^Install CentOS 7
  kernel vmlinuz
  append initrd=initrd.img ks=http://192.168.100.133/pub/ks/123.ks
  menu default
menu separator # insert an empty line
# 切换主机使用 system-config-kickstart 工具获取 123.ks
mkdir -p /var/www/html/pub/ks
cp /var/www/html/pub/isolinux/* /var/lib/tftpboot/
# 如果 DHPC 出问题，源主机 systemctl restart dhcp
``

```
Questions From AI
```
✔ dhcp.conf 缺失 allow booting; allow bootp; PXE 专属配置
✔ TFTP 配置文件 disable=yes 未改为 no，TFTP 服务无法启动
✔ pxelinux.cfg/default 是文件，原文创建成了目录
✔ 未关闭防火墙 / SELINUX，端口被拦截
```