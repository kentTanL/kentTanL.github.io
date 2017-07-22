---
layout:     post
title:      "Centos 7.2.1511 安装 Docker 遇到的问题与解决方案"
subtitle:   "Centos 7.2.1511 安装 Docker 遇到的问题与解决方案"
date:       2017-07-20 22:37:00
author:     "Kent"
header-img: "img/post-bg-2016.jpg"
catalog: true
tags:
    - classLoader
---

[TOC]

## 一、前言

最近在一个集群上安装 Docker，本来多么简单的事却因为网络原因以及系统版本变得曲折了，也由于找到了不适合的解决方案，饶了一个弯，所以特此记录一下安装过程，防止重复踩坑。

==**系统版本：Centos 7.2.1511**==

## 二、下载与上传

由于服务器不能连接外网，所以服务器使用的是公司自带的 yum 仓库，但是仓库内没有 Docker 安装包，所以需要自己下载。

下载地址：[https://yum.dockerproject.org/repo/main/centos/7/Packages/](https://yum.dockerproject.org/repo/main/centos/7/Packages/)

下载文件：

- [docker-engine-1.13.1-1.el7.centos.x86_64.rpm](https://yum.dockerproject.org/repo/main/centos/7/Packages/docker-engine-1.13.1-1.el7.centos.x86_64.rpm)
- [docker-engine-selinux-1.13.1-1.el7.centos.noarch.rpm  ](https://yum.dockerproject.org/repo/main/centos/7/Packages/docker-engine-selinux-1.13.1-1.el7.centos.noarch.rpm)

下载完成后，上传到服务器等待安装

## 三、安装

依次输入如下命令：

```shell
$ sudo yum -y localinstall docker-engine-selinux-1.13.1-1.el7.centos.noarch.rpm
$ sudo yum -y localinstall docker-engine-1.13.1-1.el7.centos.x86_64.rpm
```

这里使用`yum -y localinstall`命令，而不是`rpm -ivh`，是因为它会自动从仓库中下载并安装相应的依赖包，那么如果指定的仓库中有系统对应的版本，安装就应该是没问题的，但是我所使用的仓库中没有`7.2.1511`版本的安装包，在该目录下有一个`readme`文件（通过浏览器打开），内容如下：

```shell
This directory (and version of CentOS) is deprecated.  For normal users,
you should use /7/ and not /7.2.1511/ in your path. Please see this FAQ
concerning the CentOS release scheme:

https://wiki.centos.org/FAQ/General

If you know what you are doing, and absolutely want to remain at the 7.2.1511
level, go to http://vault.centos.org/ for packages. 

Please keep in mind that 7.2.1511 no longer gets any updates, nor
any security fix's.
```
嗯，大概意思就是让我使用版本7目录下的...

==由于对应版本的安装包不存在时发生的异常见**附录1**==

## 四、修改 yum 获取的版本

使用如下命令进入修改界面：

```shell
sudo vi /etc/yum.repos.d/CentOS-Base.repo
```

原文件如下：

```shell
[base]
name=CentOS-$releasever - Base

baseurl=http://s7repo01/centos/7.2.1511/os/$basearch/
gpgcheck=1
gpgkey=http://s7repo01/centos/RPM-GPG-KEY-CentOS-7

#released updates
[updates]
name=CentOS-$releasever - Updates
baseurl=http://s7repo01/centos/7.2.1511/updates/$basearch/
gpgcheck=1
gpgkey=http://s7repo01/centos/RPM-GPG-KEY-CentOS-7

#additional packages that may be useful
[extras]
name=CentOS-$releasever - Extras
baseurl=http://s7repo01/centos/7.2.1511/extras/$basearch/
gpgcheck=1
gpgkey=http://s7repo01/centos/RPM-GPG-KEY-CentOS-7

#additional packages that extend functionality of existing packages
[centosplus]
name=CentOS-$releasever - Plus
baseurl=http://s7repo01/centos/7.2.1511/centosplus/$basearch/
gpgcheck=1
enabled=0
gpgkey=http://s7repo01/centos/RPM-GPG-KEY-CentOS-7
```

将所有的`baseurl`属性修改为版本7对应的目录路径，`baseurl`属性修改后如下：
```shell
baseurl=http://s7repo01/centos/7/centosplus/$basearch/
```

注：只修改版本号即可，对应的仓库地址可能与此不同，能访问则不必修改。


## 五、继续安装

继续使用如下命令安装：

```shell
$ sudo yum -y localinstall docker-engine-selinux-1.13.1-1.el7.centos.noarch.rpm
$ sudo yum -y localinstall docker-engine-1.13.1-1.el7.centos.x86_64.rpm
```

如果出现某个仓库不能连接的情况，可使用如下命令禁用掉，例（查看具体的错误信息可获知仓库名称，这里是`dockerrepo`）：

```shell
sudo yum-config-manager --disable dockerrepo
```

重新输入命令，安装成功

## 附录1：安装中的问题

```shell
Loaded plugins: fastestmirror
Repository cloudera-manager is listed more than once in the configuration
Examining docker-engine-selinux-1.13.1-1.el7.centos.noarch.rpm: docker-engine-selinux-1.13.1-1.el7.centos.noarch
Marking docker-engine-selinux-1.13.1-1.el7.centos.noarch.rpm to be installed
Resolving Dependencies
--> Running transaction check
---> Package docker-engine-selinux.noarch 0:1.13.1-1.el7.centos will be installed
--> Processing Dependency: selinux-policy-base >= 3.13.1-102 for package: docker-engine-selinux-1.13.1-1.el7.centos.noarch
http://s7repo01/centos/7.2.1511/os/x86_64/repodata/repomd.xml: [Errno 14] HTTP Error 404 - Not Found
Trying other mirror.
To address this issue please refer to the below knowledge base article 

https://access.redhat.com/articles/1320623

If above article doesn't help to resolve this issue please create a bug on https://bugs.centos.org/

http://s7repo01/centos/7.2.1511/extras/x86_64/repodata/repomd.xml: [Errno 14] HTTP Error 404 - Not Found
Trying other mirror.
http://s7repo01/centos/7.2.1511/updates/x86_64/repodata/repomd.xml: [Errno 14] HTTP Error 404 - Not Found
Trying other mirror.
Loading mirror speeds from cached hostfile

... 省略

Error: Package: docker-engine-selinux-1.13.1-1.el7.centos.noarch (/docker-engine-selinux-1.13.1-1.el7.centos.noarch)
           Requires: selinux-policy-base >= 3.13.1-102
           Installed: selinux-policy-targeted-3.13.1-60.el7.noarch (@anaconda)
               selinux-policy-base = 3.13.1-60.el7
           Available: selinux-policy-minimum-3.13.1-60.el7.noarch (base)
               selinux-policy-base = 3.13.1-60.el7
           Available: selinux-policy-minimum-3.13.1-60.el7_2.3.noarch (updates)
               selinux-policy-base = 3.13.1-60.el7_2.3
           Available: selinux-policy-minimum-3.13.1-60.el7_2.7.noarch (updates)
               selinux-policy-base = 3.13.1-60.el7_2.7
           Available: selinux-policy-minimum-3.13.1-60.el7_2.9.noarch (updates)
               selinux-policy-base = 3.13.1-60.el7_2.9
           Available: selinux-policy-mls-3.13.1-60.el7.noarch (base)
               selinux-policy-base = 3.13.1-60.el7
           Available: selinux-policy-mls-3.13.1-60.el7_2.3.noarch (updates)
               selinux-policy-base = 3.13.1-60.el7_2.3
           Available: selinux-policy-mls-3.13.1-60.el7_2.7.noarch (updates)
               selinux-policy-base = 3.13.1-60.el7_2.7
           Available: selinux-policy-mls-3.13.1-60.el7_2.9.noarch (updates)
               selinux-policy-base = 3.13.1-60.el7_2.9
           Available: selinux-policy-targeted-3.13.1-60.el7_2.3.noarch (updates)
               selinux-policy-base = 3.13.1-60.el7_2.3
           Available: selinux-policy-targeted-3.13.1-60.el7_2.7.noarch (updates)
               selinux-policy-base = 3.13.1-60.el7_2.7
           Available: selinux-policy-targeted-3.13.1-60.el7_2.9.noarch (updates)
               selinux-policy-base = 3.13.1-60.el7_2.9
Error: Package: docker-engine-selinux-1.13.1-1.el7.centos.noarch (/docker-engine-selinux-1.13.1-1.el7.centos.noarch)
           Requires: selinux-policy-targeted >= 3.13.1-102
           Installed: selinux-policy-targeted-3.13.1-60.el7.noarch (@anaconda)
               selinux-policy-targeted = 3.13.1-60.el7
           Available: selinux-policy-targeted-3.13.1-60.el7_2.3.noarch (updates)
               selinux-policy-targeted = 3.13.1-60.el7_2.3
           Available: selinux-policy-targeted-3.13.1-60.el7_2.7.noarch (updates)
               selinux-policy-targeted = 3.13.1-60.el7_2.7
           Available: selinux-policy-targeted-3.13.1-60.el7_2.9.noarch (updates)
               selinux-policy-targeted = 3.13.1-60.el7_2.9
 You could try using --skip-broken to work around the problem
 You could try running: rpm -Va --nofiles --nodigest
```

使用`rpm`命令安装，可看见需要的依赖包

```shell
sudo rpm -ivh docker-engine-selinux-1.13.1-1.el7.centos.noarch.rpm 

warning: docker-engine-selinux-1.13.1-1.el7.centos.noarch.rpm: Header V4 RSA/SHA512 Signature, key ID 2c52609d: NOKEY
error: Failed dependencies:
        policycoreutils-python is needed by docker-engine-selinux-1.13.1-1.el7.centos.noarch
        selinux-policy-base >= 3.13.1-102 is needed by docker-engine-selinux-1.13.1-1.el7.centos.noarch
        selinux-policy-targeted >= 3.13.1-102 is needed by docker-engine-selinux-1.13.1-1.el7.centos.noarch
```


## 附录2：参考链接

http://www.cnblogs.com/mchina/archive/2013/01/04/2842275.html \
http://www.178linux.com/11404 \
http://www.jianshu.com/p/1fc580b71ea4 \
https://my.oschina.net/hanhanztj/blog/504915
