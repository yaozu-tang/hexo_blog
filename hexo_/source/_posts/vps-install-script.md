---
title: VPS（国外）一键(不)部署脚本
date: 2020-02-19 23:20:48
tags: ["linux", "vps", "script"]
categories: Linux运维
---

每次部署vps都要平白耗掉人生中的5个小时 为什么不自动化呢？

于是花了大量时间写了这个丑陋python脚本

<!--more-->

```python
#!/usr/bin/env python
# -*- coding: UTF-8 -*-
import os
import subprocess
import re
import time
from secrets import *

"""
Only for ubuntu 16.04
Authored by V1me
Suggestions:
    First: SSH iptables
    Then: webdav_cli
    Then: CN
    Then: update_kernel
    Then: BBR
    Then: v2ray
    Then: github_key
    Then: nginx
    Then: acme
    Then: blog
    Then: profile
"""
CN            = 0
SSH           = 0
iptables      = 0
v2ray         = 0
github_key    = 0
update_kernel = 0
BBR           = 0
webdav_cli    = 0
nginx         = 0
acme          = 0
blog          = 0
profile       = 0
zsh           = 0

def run(cmd, *args, **kwargs):
    subprocess.run(cmd, shell=True, universal_newlines=True, executable='/bin/bash', *args, **kwargs)

bash_cmd = '/bin/bash -i -c '

# Support zh-CN
if CN:
    run(r'wget -N --no-check-certificate https://raw.githubusercontent.com/FunctionClub/LocaleCN/master/LocaleCN.sh && bash LocaleCN.sh')
    run(['/bin/bash -i -c reboot'])

# SSH config
if SSH:
    if 'authorized_keys' not in os.popen('ls ~/.ssh/').read():
        run('mkdir -p ~/.ssh')
        run('wget -O ~/.ssh/authorized_keys {0}'.format(pub_key_url))  # replace the url to your pub-key

    with open(r'/etc/ssh/sshd_config', 'r') as sshd_config:
        config = sshd_config.read()
        config = re.sub(r'^.*?Port \d+$', r'Port 60022', config, flags=re.MULTILINE)
        config = re.sub(r'^.*?PasswordAuthentication (?:yes|no)$', r'PasswordAuthentication no', config, flags=re.MULTILINE)
        config = re.sub(r'^.*?RSAAuthentication (?:yes|no)$', r'RSAAuthentication yes', config, flags=re.MULTILINE)
        config = re.sub(r'^.*?PubkeyAuthentication (?:yes|no)$', r'PubkeyAuthentication yes', config, flags=re.MULTILINE)
        if not re.search('ClientAliveInterval 120', config):
            config += '\nClientAliveInterval 120\nClientAliveCountMax 720\n'

    with open(r'/etc/ssh/sshd_config', 'w') as sshd_config:
        sshd_config.write(config)

    if run('systemctl restart sshd'):
        raise Exception('SSH Server Restart Failed')

    print('[-] SSH: Success')


# iptables
if iptables:
    run('wget -O /etc/apt/sources.list https://gist.githubusercontent.com/rohitrawat/60a04e6ebe4a9ec1203eac3a11d4afc1/raw/fcdfde2ab57e455ba9b37077abf85a81c504a4a9/sources.list')
    run('apt update && apt upgrade -y')
    run('apt install iptables -y')
    run('iptables -P INPUT DROP')
    run('iptables -P OUTPUT ACCEPT')
    run('iptables -A INPUT -i lo -j ACCEPT')
    run('iptables -A INPUT -p all -m state --state ESTABLISHED,RELATED -j ACCEPT')
    run('iptables -A INPUT -p icmp -j ACCEPT')
    run('iptables -A INPUT -p tcp -m tcp --dport 60022 -j ACCEPT')
    run('iptables -A INPUT -p tcp -m tcp --dport 20 -j ACCEPT')
    run('iptables -A INPUT -p tcp -m tcp --dport 21 -j ACCEPT')
    run('iptables -A INPUT -p tcp --dport 20000:30000 -j ACCEPT')
    run('iptables -A INPUT -p tcp -m tcp --dport 25 -j ACCEPT -s 127.0.0.1')
    run('iptables -A INPUT -p tcp -m tcp --dport 25 -j REJECT')
    run('iptables -A INPUT -p tcp -m tcp --dport 53 -j ACCEPT')
    run('iptables -A INPUT -p udp -m udp --dport 53 -j ACCEPT')
    run('iptables -A INPUT -p tcp -m tcp --dport 80 -j ACCEPT')
    run('iptables -A INPUT -p tcp -m tcp --dport 443 -j ACCEPT')
    run('iptables -A INPUT -p all -m state --state INVALID,NEW -j DROP')
    run('iptables-save > /etc/iptables')

    print('[-] iptables: Success')

# v2ray
if v2ray:
    run('wget -O ~/go.sh https://install.direct/go.sh')
    run('bash ~/go.sh')
    if not os.path.exists('/home/nutstore/Roar-s-config-files'):
        raise Exception('Install WebDav Client First')
    run('cp /home/nutstore/Roar-s-config-files/v2ray/config_server_1.json /etc/v2ray/config.json')
    run('systemctl restart v2ray')
    run('systemctl enable v2ray')
    print('[-] v2ray: Success')

# github key
if github_key:
    run('apt install git -y')
    run(r'git config --global user.name "Roarcannotprogramming"')
    run(r'git config --global user.email "z1991998920@gmail.com"')
    run(r'ssh-keygen -t rsa -C "z1991998920@gmail.com" -N "" -f ~/.ssh/id_rsa -q')
    time.sleep(0.3)
    run(r'eval $(ssh-agent -s)')
    time.sleep(0.3)
    run(r'ssh-add ~/.ssh/id_rsa')
    with open(os.path.expanduser("~") + r'/.ssh/id_rsa.pub', 'r') as f:
        print('RSA Pub Key: ')
        print(f.read())

    print('[-] github_key: Success (Need add the pub key to github manually)')

# update_kernel
if update_kernel:
    run('mkdir -p ~/kernel')
    run('wget -O ~/kernel/linux-headers-4.12.10-041210_4.12.10-041210.201708300614_all.deb https://kernel.ubuntu.com/~kernel-ppa/mainline/v4.12.10/linux-headers-4.12.10-041210_4.12.10-041210.201708300614_all.deb')
    run('wget -O ~/kernel/linux-headers-4.12.10-041210-generic_4.12.10-041210.201708300614_amd64.deb https://kernel.ubuntu.com/~kernel-ppa/mainline/v4.12.10/linux-headers-4.12.10-041210-generic_4.12.10-041210.201708300614_amd64.deb')
    run('wget -O ~/kernel/linux-image-4.12.10-041210-generic_4.12.10-041210.201708300614_amd64.deb https://kernel.ubuntu.com/~kernel-ppa/mainline/v4.12.10/linux-image-4.12.10-041210-generic_4.12.10-041210.201708300614_amd64.deb')
    run('dpkg -i ~/kernel/*.deb')
    run(['/bin/bash -i -c reboot'])

# BBR
if BBR:
    if 'deb' not in os.popen('ls ~/kernel').read():
        raise Exception('Update Kernel First')
    run(r'apt install gcc g++ -y')
    run(r'apt install curl build-essential libc6-dev make -y')
    run(r'mkdir -p ~/bbr')
    os.chdir(os.path.expanduser("~") + r'/bbr')
    run(r'wget -O ~/bbr/tcp_tsunami.c https://gist.github.com/anonymous/ba338038e799eafbba173215153a7f3a/raw/55ff1e45c97b46f12261e07ca07633a9922ad55d/tcp_tsunami.c')
    run(r'echo "obj-m:=tcp_tsunami.o" > Makefile', cwd=os.getcwd())
    run('make -C /lib/modules/{1}/build M={0} modules CC=/usr/bin/gcc'.format(os.getcwd(), os.uname().release), cwd=os.getcwd())
    run(r'chmod +x ~/bbr/tcp_tsunami.ko')
    run('cp -rf ~/bbr/tcp_tsunami.ko /lib/modules/{0}/kernel/net/ipv4'.format(os.uname().release))
    run(r'insmod tcp_tsunami.ko', cwd=os.getcwd())
    run(r'depmod -a')
    run(r'echo "net.core.default_qdisc=fq" >> /etc/sysctl.conf')
    run(r'echo "net.ipv4.tcp_congestion_control=tsunami" >> /etc/sysctl.conf')
    os.chdir(os.path.expanduser("~"))
    run(['/bin/bash -i -c reboot'])

# WebDav Client
if webdav_cli:
    run('apt install davfs2 -y') # need manually config
    with open('/etc/davfs2/davfs2.conf', 'a') as f:
        f.write('ignore_dav_header 1\n')
    run('mkdir -p /home/nutstore')
    with open(r'/etc/davfs2/secrets', 'a') as f:
        f.write('https://dav.jianguoyun.com/dav/ {username} {password}\n'.format(username=nutstore_email, password=nutstore_key))
    with open('/etc/rc.local', 'r') as f:
        content = f.read()
        content = re.sub(r'^exit 0$', 'mount -t davfs https://dav.jianguoyun.com/dav/ /home/nutstore\nexit 0', content, flags=re.MULTILINE)
    with open('/etc/rc.local', 'w') as f:
        f.write(content)
    with open(os.path.expanduser("~") + r'/.bashrc', 'a') as f:
        f.write('\n' + r'alias reboot="umount /home/nutstore;reboot"' + '\n')
    run('source ~/.bashrc')
    run('mount -t davfs https://dav.jianguoyun.com/dav/ /home/nutstore')

# nginx
if nginx:
    run('apt install nginx -y')
    run('mkdir -p /etc/nginx/ssl/v1me.cn/')
    if not os.path.exists('/home/nutstore/Roar-s-config-files'):
        raise Exception('Install WebDav Client First')
    run('cp /home/nutstore/Roar-s-config-files/v2ray/nginx_v2ray* /etc/nginx/sites-available/')
    run('ln -s /etc/nginx/sites-available/nginx_v2ray_end_1.conf /etc/nginx/sites-enabled/nginx_v2ray_end_1.conf')
    run('service nginx restart')

#acme.sh
if acme:
    run('apt install socat curl -y')
    run('curl  https://get.acme.sh | sh')
    run('echo \"export CF_Key=\\\"{0}\\\"\" >> {1}'.format(cloudflare_key, os.path.expanduser("~") + '/.bashrc'))
    run('echo \"export CF_Email=\\\"{0}\\\"\" >> {1}'.format(cloudflare_email, os.path.expanduser("~") + '/.bashrc'))
    run('export CF_Key=\"{0}\";export CF_Email=\"{1}\";'.format(cloudflare_key, cloudflare_email) + os.path.expanduser("~") + r'/.acme.sh/acme.sh --issue -d v1me.cn -d *.v1me.cn --dns dns_cf')
    run(os.path.expanduser("~") + r'/.acme.sh/acme.sh  --installcert  -d v1me.cn --key-file /etc/nginx/ssl/v1me.cn/v1me.cn.key --fullchain-file /etc/nginx/ssl/v1me.cn/fullchain.cer --reloadcmd "service nginx force-reload"')

if blog:
    run('cp /home/nutstore/Roar-s-config-files/nginx/nginx_blog.conf /etc/nginx/sites-available/')
    run('ln -s /etc/nginx/sites-available/nginx_blog.conf /etc/nginx/sites-enabled/nginx_blog.conf')
    run('service nginx restart')
    run('useradd git')
    run('echo \"git:{0}\" | chpasswd'.format(pswd_user_git))
    run('mkdir -p /home/git/blog.git')
    run('chmod 740 /etc/sudoers')
    run('echo \"git ALL=(ALL:ALL) ALL\" >> /etc/sudoers')
    run('chmod 440 /etc/sudoers')
    run('git init --bare', cwd='/home/git/blog.git')
    run('mkdir -p /var/www/blog')
    run('mkdir -p /home/git/.ssh')
    run('cp ~/.ssh/authorized_keys /home/git/.ssh/authorized_keys')
    with open('/home/git/blog.git/hooks/post-receive', 'w') as f:
        f.write('''#!/bin/bash
GIT_REPO=/home/git/blog.git
TMP_GIT_CLONE=/tmp/blog
PUBLIC_WWW=/var/www/blog
rm -rf ${TMP_GIT_CLONE}
git clone $GIT_REPO $TMP_GIT_CLONE
rm -rf ${PUBLIC_WWW}/*
cp -rf ${TMP_GIT_CLONE}/* ${PUBLIC_WWW}
''')
    run('chmod +x /home/git/blog.git/hooks/post-receive')
    run('chown -R git:git  /var/www/blog')
    run('chown -R git:git  /home/git')

if profile:
    run('mkdir -p /home/v1me/proj')
    run('git clone {0}'.format(profile_github_url), cwd='/home/v1me/proj')
    run('apt install -y make build-essential libssl-dev zlib1g-dev libbz2-dev libreadline-dev libsqlite3-dev wget curl llvm libncurses5-dev  libncursesw5-dev xz-utils tk-dev')
    run('mkdir -p ~/python3.8')
    run('wget -O ~/python3.8/Python-3.8.1.tgz https://www.python.org/ftp/python/3.8.1/Python-3.8.1.tgz')
    run('tar zxvf ~/python3.8/Python-3.8.1.tgz', cwd=os.path.expanduser("~")+ r'/python3.8')
    run('./configure --prefix=/usr/local/python3.8  --enable-optimizations --enable-loadable-sqlite-extensions && make -j8 && sudo make install', cwd=os.path.expanduser("~")+ r'/python3.8/Python-3.8.1')

if zsh:
    run('sudo apt-get install zsh git -y')
    run('chsh -s /bin/zsh')
    run(r'sh -c "$(curl -fsSL https://raw.github.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"')
    run('echo \"export CF_Key=\\\"{0}\\\"\" >> {1}'.format(cloudflare_key, os.path.expanduser("~") + '/.zshrc'))
    run('echo \"export CF_Email=\\\"{0}\\\"\" >> {1}'.format(cloudflare_email, os.path.expanduser("~") + '/.zshrc'))
    run('apt install autojump')
    run('echo \". /usr/share/autojump/autojump.sh\" >> ~/.zshrc')
    run('git clone https://github.com/zsh-users/zsh-syntax-highlighting.git', cwd='/root/.oh-my-zsh/custom/plugins')
    run('git clone git://github.com/zsh-users/zsh-autosuggestions $ZSH_CUSTOM/plugins/zsh-autosuggestions')
    with open(os.path.expanduser("~") + r'/.zshrc', 'r') as f:
        config = f.read()
    config = re.sub(r'^plugins=.*?$', 'plugins=(git extract z zsh-autosuggestions)', config, flags=re.MULTILINE)
    config = re.sub(r'^ZSH_THEME=.*?$', r'ZSH_THEME="ys"', config, flags=re.MULTILINE)
    config += '\n' + r'source $ZSH_CUSTOM/plugins/zsh-autosuggestions/zsh-autosuggestions.zsh' + '\n' + r'source $ZSH_CUSTOM/plugins/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh' + '\n' + r'alias reboot="umount /home/nutstore;reboot"' + '\n'
    with open(os.path.expanduser("~") + r'/.zshrc', 'w') as f:
        f.write(config)

    subprocess.run('source ~/.zshrc', shell=True, universal_newlines=True, executable='/bin/zsh')
```

