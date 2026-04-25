
# 后续

## 安装openssh
```shell
现在已经进来了LFS中，即在LFS中操作

# 在LFS中安装libedit
tar -zxvf libedit-20251016-3.1.tar.gz
cd libedit-20251016-3.1
./configure --prefix=/usr && make && make install


# 在LFS中安装openssh
tar -zxvf openssh-10.1p1.tar.gz
cd openssh-10.1p1
./configure --prefix=/usr --sysconfdir=/etc/ssh \
--with-md5-passwords --with-privsep-path=/var/lib/sshd \
--with-default-path=/usr/bin --with-libedit --with-ssl-dir=/usr
关键点提醒：
--with-libedit：正如你之前要求的，启用高级命令行编辑功能
--with-privsep-path：这是安全隔离目录，必须存在

注意：缺少依赖报错：如果在 configure 时报错找不到 zlib 或 openssl，请确认你是否已经安装了这两个包

make && make install

# 查看ssh版本
-bash-5.2# ssh -V
OpenSSH_10.1P1, OpenSSL 3.4.1 11 Feb 2025


# 将SSH做成Systemd服务
1、创建必要目录和用户
install  -v -m700 -d /var/lib/sshd
chown    -v root:sys /var/lib/sshd
groupadd -g 50 sshd
useradd  -c 'sshd PrivSep' -d /var/lib/sshd -g sshd -s /bin/false -u 50 sshd

2、编写服务文件
vim /etc/systemd/system/sshd.service
[Unit]
Description=OpenSSH Daemon
After=network.target

[Service]
Type=forking
ExecStart=/usr/sbin/sshd
ExecReload=/bin/kill -HUP $MAINPID
KillMode=process
Restart=on-failure

[Install]
WantedBy=multi-user.target



# 允许root登陆
vim /etc/ssh/sshd_config
PermitRootLogin yes


systemctl daemon-reload
systemctl enable sshd && systemctl start sshd


```
![image](https://img2024.cnblogs.com/blog/1139005/202604/1139005-20260424142512837-1991143041.png)


```shell
# 现在就能用ssh连接上，所以下面的一切有了PS1的提示符

为什么SSH登录后 ip a 找不到命令？
这是一个非常经典的 环境变量（PATH）隔离问题。

根本原因：PATH 变量不一致
在Linux 中，ip 命令通常位于 /usr/sbin/ip 或 /sbin/ip
宿主机/本地登录：你的 PATH 可能包含了 /usr/sbin
SSH 登录：当你通过 SSH 进入时，系统会读取你在 LFS 内部设置的 /etc/profile 或 ~/.bash_profile。如果这些文件里定义的 PATH 只有 /usr/bin 而没有 /usr/sbin，系统就找不到ip命令

验证方法：
在 SSH 终端里执行：echo $PATH
你会发现结果里很可能缺少了 /usr/sbin


# 修改/添加PATH声明，如果不修改/添加则会出现在宿机上的LFS中和ssh连接进来后执行echo $PATH的结果就会不一样
-bash-5.2# echo "export PATH=/usr/bin:/usr/sbin" | tee /etc/profile
export PATH=/usr/bin:/usr/sbin

-bash-5.2# source /etc/profile

# 查看IP
-bash-5.2# ip a sh ens33
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:93:90:28 brd ff:ff:ff:ff:ff:ff
    altname enp2s1
    altname enx000c29939028
    inet 172.16.186.141/24 metric 1024 brd 172.16.186.255 scope global dynamic ens33
       valid_lft 1613sec preferred_lft 1613sec
    inet6 fe80::20c:29ff:fe93:9028/64 scope link proto kernel_ll 
       valid_lft forever preferred_lft forever





```






## 修改PS1
```shell
# 在宿主机操作时路径要加上$LFS,比如cat > $LFS/etc/bashrc << "EOF"...
(lfs chroot) I have no name!:/# cat > /etc/bashrc << "EOF"
# /etc/bashrc

# 颜色定义
CYAN='\[\e[36m\]'
GREEN='\[\e[32m\]'
PURPLE='\[\e[35;1m\]'
RESET='\[\e[0m\]'

# 苹果风格温和配色PS1
# \u:用户名 \h:主机名 \w:完整路径 (解决你看不见路径的问题)
export PS1="${CYAN}\u${RESET}@${GREEN}\h${RESET}:${PURPLE}\w${RESET}\$ "

alias ls='ls --color=auto'
alias ll='ls -l'
EOF


# 设置全局 profile (所有用户登录的入口)
(lfs chroot) I have no name!:/# cat > /etc/profile << "EOF"
if [ -f /etc/bashrc ]; then
  . /etc/bashrc
fi
EOF

# 设置 root 用户的个人配置
(lfs chroot) I have no name!:/# cat > /root/.bash_profile << "EOF"
if [ -f /etc/bashrc ]; then
  . /etc/bashrc
fi
EOF


-bash-5.2# source ~/.bash_profile

# 可以看到PS1已经改变
root@lfs:~$ cd /etc/
root@lfs:/etc$


```






# 设置网络
```shell
root@lfs:/etc$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute 
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:93:90:28 brd ff:ff:ff:ff:ff:ff
    altname enp2s1
    altname enx000c29939028
    inet 172.16.186.141/24 metric 1024 brd 172.16.186.255 scope global dynamic ens33
       valid_lft 1768sec preferred_lft 1768sec
    inet6 fe80::20c:29ff:fe93:9028/64 scope link proto kernel_ll 
       valid_lft forever preferred_lft forever
3: sit0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN group default qlen 1000
    link/sit 0.0.0.0 brd 0.0.0.0


root@lfs:/etc$ grep 172 /etc/resolv.conf 
nameserver 172.16.186.2

root@lfs:/etc$ ping -c2 172.16.186.2
PING 172.16.186.2 (172.16.186.2): 56 data bytes
64 bytes from 172.16.186.2: icmp_seq=0 ttl=128 time=0.455 ms
64 bytes from 172.16.186.2: icmp_seq=1 ttl=128 time=0.344 ms
--- 172.16.186.2 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max/stddev = 0.344/0.400/0.455/0.056 ms

root@lfs:/etc$ ping -c2 youku.com
PING youku.com (106.11.40.57): 56 data bytes
64 bytes from 106.11.40.57: icmp_seq=0 ttl=128 time=24.541 ms
64 bytes from 106.11.40.57: icmp_seq=1 ttl=128 time=24.418 ms
--- youku.com ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max/stddev = 24.418/24.480/24.541/0.061 ms


# 当前使用的DHCP获取的IP,改成用静态的IP,当静态IP异常时再用dhcp获取IP（文件名10-eth-static.network未改名）
root@lfs:~$ mv /etc/systemd/network/10-eth-dhcp.network  /etc/systemd/network/20-eth-dhcp.network
root@lfs:~$ systemctl restart systemd-networkd


```



# 解决"源码安装"焦虑的实战方案：DESTDIR机制
```shell
既然已经连上ssh了，我教你一招最能启发“包管理”思路的操作，叫 DESTDIR 安装法。这是所有 .deb 或 .rpm 制作的基石
试着在ssh里操作一次（以安装一个小工具如 vim 为例）：
编译时不要直接 make install
执行 make DESTDIR=/tmp/vim-pkg install
去 /tmp/vim-pkg 看看，你会发现它生成了一个完整的、带路径的目录结构
感悟： 如果你把这个文件夹打成 vim.tar.gz，发给别人，别人只需要解压到根目录 /，不就实现了类似 apt 的安装效果吗？所谓的包管理器，本质上就是一套"自动下载压缩包并解压到对应位置"的逻辑

该项未测试

```

