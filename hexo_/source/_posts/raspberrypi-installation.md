---
title: 树莓派安装过程
date: 2020-02-26 03:08:44
tags: ["linux", "Raspberry Pi", "NAS"]
categories: Linux运维
description: 记录树莓派的一次安装
---

## 下载镜像

直接在官网下载即可 

![树莓派官网](https://res.cloudinary.com/v1mecn/image/upload/v1582639754/blog/树莓派官网_zifv2m.png)

下载Raspbian: 一个基于debian的树莓派系统

但是速度很慢 建议挂梯子 

也支持p2p下载 

[官网地址](https://www.raspberrypi.org/downloads/)

验证SHA-256 (注意: 他给的是`.zip`文件的哈希值)

解压 得到`.img`镜像文件



## 写入镜像

推荐使用 [rufus](https://rufus.ie/) 来制作安装盘

![rufus](https://res.cloudinary.com/v1mecn/image/upload/v1582640295/blog/rufus_rvaodr.png)

 

和制作linux系统安装盘一样 选取设备(写入镜像的盘 也就是sd卡) 

引导类型直接按旁边的"选择"按钮选择镜像即可

后面都是默认的 点击"开始" 几十秒钟即可做好

## 开启 SSH

树莓派默认不开启ssh

开启只需要在`boot`下新建一个`ssh`(没有后缀名)文件即可

![boot目录](https://res.cloudinary.com/v1mecn/image/upload/v1582640658/blog/D8_K_A_3M_HXH78S_7OTXXX_i5ssho.png)



## 开启 wifi

这一步既可以在安装之前执行也可以在安装之后执行

笔者选择在安装之前执行

同样的 在`boot`下创建`wpa_supplicant.conf`文件(这个文件在安装的过程中会被copy到`/etc/wpa_supplicant/wpa_supplicant.conf`)

```
country=CN
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1

network={
    ssid="your-ssid"
    psk="your-password"
    priority=1
}
```

将wifi的ssid(wifi名称)和password替换成自己的即可

这里默认认证模式是`WPA-PSK` 如果你的wifi不是`WPA-PSK`认证方式需要自己补充参数

详细如下

```
#ssid: wifi名
#psk: 密码
#priority:连接优先级，数字越大优先级越高（不可以是负数）
#scan_ssid:连接隐藏WiFi时需要指定该值为1

# No Password
network={
ssid="your-ssid"
key_mgmt=NONE
}

# WEP
network={
ssid="your-ssid"
key_mgmt=NONE
wep_key0="your-password"
}

# WPA/WPA2
network={
ssid="your-ssid"
key_mgmt=WPA-PSK
psk="your-password"
}
```

链接wifi命令: `sudo ifup wlan0`

断开wifi命令: `sudo ifdown wlan0`



## 连接树莓派

将sd卡插入树莓派

开机

#### ssh连接

进入家里路由网关(TP-Link是`192.168.0.1` 不通型号路由器都会在说明书中说明)

查看连接的设备可以看到有raspberrypi

![路由网关](https://res.cloudinary.com/v1mecn/image/upload/v1582644174/blog/37_W_F_1WQ2696UL_BLAV_4_zdlaxu.png)

记住ip地址

在linux shell中执行

`ssh pi@your-ip-address`

密码是`raspberry`

进入root用户 之后的操作均在root下进行

`sudo su -`

#### 修改apt源

这里选用清华的tuna源(我是LUG的叛徒) [tuna raspbian源使用帮助](https://mirror.tuna.tsinghua.edu.cn/help/raspbian/)

```
# 编辑 `/etc/apt/sources.list` 文件，删除原文件所有内容，用以下内容取代：
deb http://mirrors.tuna.tsinghua.edu.cn/raspbian/raspbian/ buster main non-free contrib
deb-src http://mirrors.tuna.tsinghua.edu.cn/raspbian/raspbian/ buster main non-free contrib

# 编辑 `/etc/apt/sources.list.d/raspi.list` 文件，删除原文件所有内容，用以下内容取代：
deb http://mirrors.tuna.tsinghua.edu.cn/raspberrypi/ buster main ui
```

raspberrypi原生lite系统是没有vim的，所以先用vi吧 2333

#### 更新软件

`apt update && apt upgrade -y`

`apt install vim -y`

#### 修改pi用户和root用户的默认密码

`passwd pi`

`passwd root`

#### 一键安装

参考我前面一篇博客

#### 安装zsh

这个也写在一键脚本了

#### 文件共享

局域网文件共享 有了这个 加个硬盘 树莓派就是一个家用NAS了

###### 安装`samba`

`apt install samba samba-common-bin –y`

备份config文件

`cp /etc/samba/smb.conf /etc/samba/smb.conf.bak`

在`/etc/samba/smb.conf`尾部加入

```
[public]
   comment = public storage
   path = /mnt/udisk
   valid users = pi
   read only = no
   create mask = 0777
   directory mask = 0777
   guest ok = no
   browseable = yes
```

###### 挂载硬盘

直接插入硬盘(USB)

`mkdir -p /home/disk`

`mount /dev/sda1 /home/disk`

