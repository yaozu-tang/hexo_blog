---
title: 装新系统备忘
date: 2020-03-10 23:59:48
tags:
- linux
- ctf
categories: Linux运维
---

做CTF的 pwn 题很多时候本地和远程一致会方便很多

不可避免的会有很多个不同版本的Ubuntu虚拟机

每次安装都要查各种包 很麻烦

所以这里做一个备忘

<!--more-->



## 系统升级

### 先换源

一开始装好虚拟机没有vmware-tools 剪切板无法共享

所以得手动输入命令

目前最简短的办法是

`wget https://mirror.tuna.tsinghua.edu.cn/help/ubuntu/`

然后在index.html中找

```
<script id="apt-template" type="x-tmpl-markup">
# 默认注释了源码镜像以提高 apt update 速度，如有需要可自行取消注释
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ {{release_name}} main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ {{release_name}} main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ {{release_name}}-updates main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ {{release_name}}-updates main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ {{release_name}}-backports main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ {{release_name}}-backports main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ {{release_name}}-security main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ {{release_name}}-security main restricted universe multiverse

# 预发布软件源，不建议启用
# deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ {{release_name}}-proposed main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ {{release_name}}-proposed main restricted universe multiverse
</script>
```

复制粘贴到`/etc/apt/sources.list`

然后用vim `:%s/{{release_name}}/xxx/g`替换掉即可



`sudo apt update && sudo apt upgrade -y`

### 安装vmware-tools

`sudo apt install open-vm-tools-desktop`

`reboot`

### 更换语言为中文(随意)

手动在设置里面换

然后重启



## pwn环境

### 下载`glibc`源码

`sudo apt install glibc-source`

这个是`tar.xz`文件 手动在`/usr/src/glibc`里面解压

`cd /usr/src/glibc `

`sudo tar xvJf glibc-x.xx.tar.xz`



### 下载`glibc debug symbol`

一般自带了x64的`libc-dbg`

`sudo apt install libc6-dbg`

虽然32位的题目少 还是备着比较好

`sudo apt install libc6:i386`

`sudo apt install libc6-dbg:i386`



### 安装GDB

一般自带`gdb`

`sudo apt install gdb`

然后装`pwndbg`和`gef`

一般用户态用`pwndbg` 调`kernel`用`gef`（在这里感谢吴炜师兄避免神坑）

按照`pwndbg`[官网](https://github.com/pwndbg/pwndbg)

```shell
sudo apt install git
git clone https://github.com/pwndbg/pwndbg
cd pwndbg
./setup.sh
```

由于`git clone`的速度感人 可以选择性挂梯子

按照`gef`[官网](https://github.com/hugsy/gef)

```
wget -O ~/.gdbinit-gef.py -q https://github.com/hugsy/gef/raw/master/gef.py
echo source ~/.gdbinit-gef.py >> ~/.gdbinit
```

注意`~/.gdbinit`只用留一个就可以了 看情况用哪个



基本也就这样了 后面就是`ipython`、公钥之类杂七杂八的了

看自己喜好吧
