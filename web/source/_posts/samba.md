---
title: 搭建Samba服务器
date: 2017-03-13 21:31:10
tags:
---

## 目录：
一、samba的安装
二、可以匿名用户登录
三、认证用户登录
四、用户帐号的映射
五、客户端访问控制
六、隐藏共享
七、问题集

-----------------------------------
### 一、samba的安装
``` bash
yum list|grep samba4
yum install samba4* -y
启动：
 /etc/init.d/smb restart
 /etc/init.d/nmb restart
设置自动启动：
chkconfig  smb on
chkconfig  nmb on
```

### 二、可以匿名用户登录
``` bash
 vim /etc/samba/smb.conf
[global]
security = share
[public]
comment= public
path = /public
public = yes
writable = yes

mkdir public
chmod 777 public
/etc/init.d/smb restart
/etc/init.d/nmb restart
```

测试：
``` bash
smbclient //10.10.54.81/public

\\10.10.54.81\public          --window在计算机网络地址栏输入

Ubuntu
smb://10.10.54.81/public     --linux
```

### 三、认证用户登录

(1)创建samba用户
``` bash
 groupadd public
 useradd nan -g public     --创建系统用户
 id nan               --查看用户信息
 uid=509(nan) gid=509(public) groups=509(public)
 pdbedit -a -u nan     --创建samba用户
 pdbedit -L          --查看创建的samba用户
 nan:509:

-u 用户名 -a 创建
```

//修改密码
``` bash
smbpasswd nan
```

//删除
``` bash
 pdbedit -x lisn
``` 

//查看samba进程链接
``` bash
smbstatus [-pS] [-u username]

 smbstatus -u public
```

//测试配置文件
``` bash
 testparm /etc/samba/smb.conf
Load smb config files from /etc/samba/smb.conf
rlimit_max: increasing rlimit_max (1024) to minimum Windows limit (16384)
Processing section "[public]"
WARNING: The security=share option is deprecated
Loaded services file OK.
Server role: ROLE_STANDALONE
Press enter to see a dump of your service definitions
```

(2)修改配置文件
``` bash
vim /etc/samba/smb.conf
[global]
security = user
config file=/etc/samba/smb.conf.%U

[public]
comment= public
path = /public
write list= public
writable = yes
browseable=yes          --设置显示共享文件夹,默认显示

 chgrp public /public
或者
chown root.public /public

 chmod 755 /public/

重启：
 /etc/init.d/smb restart
 /etc/init.d/nmb restart

测试:
smbclient //10.10.54.81/public -U public
```

### 四、用户帐号的映射
帐号映射配置：
``` bash
 vim /etc/samba/smbusers
nan = aa bb
```

``` bash
配置文件:
 vim /etc/samba/smb.conf

[global]
username map=/etc/samba/smbusers     --帐号映射
[public]
comment= public
path = /public
writable = yes
browseable = yes

```

重启
``` bash
 /etc/init.d/smb restart
 /etc/init.d/nmb restart
```

测试：
使用帐号aa/bb登录


### 五、客户端访问控制
hosts allow:只允许配置的IP地址/端可以访问samba服务端
hosts deny:只拒绝配置的IP地址/端可以访问samba服务端

[global]
security = user
hosts deny =10.10.54.80  --只禁止10.10.54.80访问
hosts allow =10.10.54.80 --只允许10.10.54.80访问
[public]
comment= public
path = /public
public = yes
write list=+nan,@public --允许nan用户，public组写入权限

测试：
当hosts deny =10.10.54.80，10.10.54.80不能访问
当hosts allow =10.10.54.80，只有10.10.54.80能访问


### 六、隐藏共享
需求：/security目录值可以boss用户浏览和登录，/pub目录只可以@yanfa,boss可以访问和浏览

(1)创建用户
 useradd boss
 groupadd yanfa
 useradd lsn -g yanfa
 pdbedit -a -u boss

(2)修改配置文件
需要针对boss用户单独创建配置文件smb.conf.boss
 cp smb.conf smb.conf.boss
 vim /etc/samba/smb.conf
[global]
security = user
config file = /etc/samba/smb.conf.%U
[public]
comment= yanfa access
path = /public
write list= @yanfa,boss

 vim /etc/samba/smb.conf.boss
[global]
security = user
[public]
comment= yanfa access
path = /public
write list= @yanfa,boss
[security]
comment = only boss access
path = /security
writable = yes

 /etc/init.d/nmb restart
 /etc/init.d/smb restart

测试：
smbclient -L 10.10.54.226 -U boss --boss用户
smbclient -L 10.10.54.226 -U wenl --研发组人

挂载
mount -t cifs //10.10.54.81/softs /mnt/

设置自动挂载：

cat /etc/fstab
//192.168.27.233/public /mnt/data/ cifs defaults,username=XXXX,password=XXXX  0 0


setsebool -P samba_export_all_rw on

### 六、问题集

1.不知道什么原因导致samba的nmb
nmb状态显示：
dead but pid file exists

根据网上
!ps -efww|grep smbd|grep -v grep|cut -c 9-15|xargs kill 
没用!!!!

解决方法：
卸载samba更换为samba4
再重新配置，OK!!!




