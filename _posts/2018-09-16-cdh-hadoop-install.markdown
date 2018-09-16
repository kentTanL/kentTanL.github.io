---
layout:     post
title:      "CDH-Hadoop 安装"
date:       2018-09-16 23:20:00
author:     "Kent"
header-img: "img/post-bg-2016.jpg"
catalog: true
tags:
    - hadoop, yarn
---

[TOC]
[TOC]

# 一、 摘要

此文档主要用于安装 CDH，CDH是Cloudera的软件发行版，包含Apache Hadoop及相关项目。详情请参见官网介绍：
- 中文：[https://cn.cloudera.com/developers/inside-cdh.html](https://cn.cloudera.com/developers/inside-cdh.html)
- 英文：[https://www.cloudera.com/products/open-source/apache-hadoop/key-cdh-components.html](https://www.cloudera.com/products/open-source/apache-hadoop/key-cdh-components.html)

其中，需要我们手动安装的主要是 Cloudera Manager（后面简称为 CM），因为 Hadoop 相关服务会由 CM 通过界面的方式提供安装，其安装原理大致为：CM
通过 ssh/scp 将 Hadoop 相关服务安装包分发到各节点，然后通过脚本自动安装与启动即可。

所以以下主要包括一些环境的配置（比如 CM 安装 Hadoop 相关服务时需要分发软件包，此时就需要配置 ssh 无密登陆，另外针对可能引起 hadoop 集群问题的一些配置）以及数据库的安装（CM 以及 hive 需要使用数据库）等。具体路线大致为：

- 服务器基础配置（针对所有主机）
    - 配置 sudo 无密
    - 配置 ssh 无密登陆
    - 其它基础配置
- Cloudera Manager 安装
    - JDK 安装
    - 数据库安装
- Hadoop 服务以及 CM agent 安装（通过 CM 自动分发安装）
    - JDK 安装
    - CM Angent 安装
    - Hadoop 相关服务安装

# 二、安装

请注意本文档与实际安装环境的区别：
- 本文档安装版本为：CDH 5.8.3
- 系统版本：centos 7.2.1511
- 每台服务器 CPU / Physical memory / Disk 信息：
    - CM 服务器（两台）：CPU core 8, Physical memory 23.4 GiB, Disk ~= 195.9 GiB
    - master 节点（5 台）：CPU core 16, Physical memory 251.6 GiB, Disk ~= 4TiB 
    - salve 节点（35 台）：CPU core 16, Physical memory 251.6 GiB, Disk ~= 13.1TiB 
- 因为是内网安装，所以我们提前下载了安装包，并配置了 yum repository，repo 文件如下：
    ```
    $ cat /etc/yum.repos.d/cloudera-manager.repo
    [cloudera-manager]
    name = Cloudera Manager, Version 5.8.3
    baseurl = http://172.0.0.62/cm5/5.8.3/
    gpgkey = http://172.0.0.62/RPM-GPG-KEY-cloudera
    gpgcheck = 1
    ```

## 1. 基础环境配置

### 1\) 主机介绍

在安装之前，我们会预先配置主机名称，主机名称和 IP 通常是根据安装的服务而有区别的，比如这次有 42 台服务器，那么它们的名称与所安装的服务会有如下的对应关系（因为涉及敏感信息，所以假设前缀都为xxxx_，但实际上应该是有业务含义的前缀）：

主机名称 | 所安装服务
---|---
xxxx_cli01 | CM Web 服务器
xxxx_cli02 | MySQL or MariaDB <br> CM Event Server <br> CM Alert Publisher
xxxx_hdp(01 - 05) | HBase Master <br> HDFS NameNode <br> HDFS Balancer <br> HDFS httpFS <br> HDFS JournalNode <br> Hive Gateway <br> Hive Metastore Server <br> Hive Server2 <br> Spark History <br> Spark Gateway <br> Yarn ResourceManager <br> Yarn JobHistory Server <br> Zookeeper
xxxx_hdp(11 - 45) | HBase RegionServer <br> HDFS DataNode <br> Yarn NodeManager

可以看见，以 cli01/cli02 结尾的主要安装 CM 相关服务，以 hdp01 - hdp05 结尾的主要是 master-salve 架构中的一些 master 服务，而以 hdp11 - hdp45 结尾的则是 salve 服务，也是数据实际存放的节点。在了解了安装的服务器之后，我们再来进行安装，这样更能辨别所需的操作是在哪一个节点上面完成的。 

### 2\) 配置 sudo 无密

进入 1.configure-no-password-sudo 目录，并根据 readme.md 文件操作（请根据实际情况跳过）。
 
### 3\) 配置 SSH 无密登陆以及其它基础配置

进入 2.configure-and-check-system-env 目录，并根据 readme.md 文件操作（请根据实际情况跳过）。

其中基础配置主要包含以下配置：

- 配置 SSH 无密登陆
- 配置 host 文件：确保所有主机都能正确的解析自己以及集群内其它所有主机的主机名
- 配置 repostory 文件：用于内网安装
- 配置打开文件的最大句柄数：防止 HDFS too many open files 异常，本次为 32768，
- 关闭 SELinux，使其状态为 disabled：启用时可能限制 SSH 免密登陆
- 关闭防火墙
- 配置 NTP 服务：集群内所有节点的时间必须同步
- 配置 swappiness：使 vm.swappiness = 0，以避免使用 swap 分区
- 禁用透明大页面压缩（Transparent HugePages / THP）：使 transparent_hugepage=never，以提升系统性能
- 优化 TCP 连接设置


## 2. 检查系统环境配置

**注意：**

1. 通过以下命令获取 cluster 中各 server 的 name 是大写还是小写，hosts 文件中 host 列表要与它保持一致，否则后面会出问题； 

```
python -c 'import socket; print socket.getfqdn(), socket.gethostbyname(socket.getfqdn())'
```

2. /etc/hosts 设置，主机名全域名在前，FQDN 在后，例：

```
ipaddress    hostname.mercury.corp  hostname

172.0.0.10  hostname.mercury.corp  hostname
```

## 3. 安装依赖数据库

可选数据库（仅选其中之一即可）：

- MySQL
- MariaDB（推荐）

### 3.1 MySQL

#### 3.1.1 安装

```
$ sudo yum install mysql-server
$ sudo service mysqld start
```

#### 3.1.2 启动

> 请检查 mysql 默认引擎，如果不是innodb，请添加default-storage-engine=INNODB到my.cnf配置文件中（通常在/etc/mycnf），并重启 mysql 服务。

```
$ sudo service mysqld restart
```

#### 3.1.3 安装 MySQL JDBC Connector

> hadoop 相关服务需要连接 mysql 服务

```
$ sudo yum install mysql-connector-java
```

#### 3.1.4 确认 MySQL 服务有开机启动

```
$ sudo /sbin/chkconfig mysqld on
$ sudo /sbin/chkconfig --list mysqld

mysqld 0:off 1:off 2:on 3:on 4:on 5:on 6:off
```

#### 3.1.5 创建 Cloudera Manager 需要的数据库

```sql
create database cloudera_manager_service DEFAULT CHARACTER SET utf8;
grant all on cloudera_manager_service.* to 'cloudera-scm'@'%' identified by 'xxxx' with grant option; 

create database hive DEFAULT CHARACTER SET utf8;
grant all on hive.* to 'xxxx' identified by 'xxxx' ;
```

### 3.2 MariaDB（推荐）

#### 3.2.1 安装

官方文档：[http://www.cloudera.com/documentation/enterprise/5-8-x/topics/install_cm_mariadb.html#install_cm_mariadb_config](http://www.cloudera.com/documentation/enterprise/5-8-x/topics/install_cm_mariadb.html#install_cm_mariadb_config)

```
$ sudo yum clean all
$ sudo yum install mariadb-server
```

#### 3.2.2 配置

1\) 下面是 Cloudera 推荐的一个配置文件：

> 请将以下配置追加到原配置文件中，并注意分区 [mysqld] [mysqld_safe]

```
$ sudo vi /etc/my.cnf
```

```
[mysqld]
transaction-isolation = READ-COMMITTED
# Disabling symbolic-links is recommended to prevent assorted security risks;
# to do so, uncomment this line:
# symbolic-links = 0

key_buffer = 16M
key_buffer_size = 32M
max_allowed_packet = 32M
thread_stack = 256K
thread_cache_size = 64
query_cache_limit = 8M
query_cache_size = 64M
query_cache_type = 1

max_connections = 550
#expire_logs_days = 10
#max_binlog_size = 100M

#log_bin should be on a disk with enough free space. Replace '/var/lib/mysql/mysql_binary_log' with an appropriate path for your system
#and chown the specified folder to the mysql user.
log_bin=/var/lib/mysql/mysql_binary_log

binlog_format = mixed

read_buffer_size = 2M
read_rnd_buffer_size = 16M
sort_buffer_size = 8M
join_buffer_size = 8M

# InnoDB settings
innodb_file_per_table = 1
innodb_flush_log_at_trx_commit  = 2
innodb_log_buffer_size = 64M
innodb_buffer_pool_size = 4G
innodb_thread_concurrency = 8
innodb_flush_method = O_DIRECT
innodb_log_file_size = 512M

[mysqld_safe]
log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid
```

2\) 停止 Mariadb

```
$ sudo systemctl  start mariadb.service 
$ sudo systemctl  stop mariadb.service
```

3\) 备份日志

```
sudo mkdir /var/lib/mysql_bak;
sudo mv /var/lib/mysql/ib_logfile0 /var/lib/mysql_bak
sudo mv /var/lib/mysql/ib_logfile1 /var/lib/mysql_bak
```

4\) 设置开机启动

```
// Ensure the MariaDB server starts at boot.
$ sudo systemctl  start mariadb.service
$ sudo systemctl  enable mariadb.service
$ sudo systemctl  status mariadb.service
```

5\) 设置 MariaDB 账户密码

```
$ sudo /usr/bin/mysql_secure_installation
[...]
Enter current password for root (enter for none):
OK, successfully used password, moving on...
[...]
Set root password? [Y/n] y
New password:
Re-enter new password:
Remove anonymous users? [Y/n] Y
[...]
Disallow root login remotely? [Y/n] N
[...]
Remove test database and access to it [Y/n] Y
[...]
Reload privilege tables now? [Y/n] Y
All done!
```

#### 3.2.3 创建 Cloudera Manager 所需数据库

> 注：xxxx 为所使用的数据库密码，请根据实际情况进行更改，后面也会用到

```
$ mysql -u root –p

create database cloudera_manager_service DEFAULT CHARACTER SET utf8; 
grant all on cloudera_manager_service.* to 'cloudera-scm'@'%' identified by 'xxxx' with grant option;

create database hive DEFAULT CHARACTER SET utf8;
grant all on hive.* to 'xxxx' identified by 'xxxx' ;
FLUSH PRIVILEGES;
```

#### 3.2.4 安装 MariaDB JDBC Connector （服务需要）

1\) 下载 mysql jdbc driver
http://www.mysql.com/downloads/connector/j/5.1.html

2\) 解压下载文件，示例：

```
tar zxvf mysql-connector-java-5.1.40.tar.gz
```

3\) 拷贝并重命名 jdbc driver 到目标地址，示例：

```
$ sudo cp mysql-connector-java-5.1.40/ mysql-connector-java-5.1.40-bin.jar /usr/share/java/mysql-connector-java.jar
```

如果目标文件夹不存在与这个主机，你可以在拷贝 jar 文件前创建它，示例：

```
$ sudo mkdir -p /usr/share/java/
$ sudo cp mysql-connector-java-5.1.40-bin.jar /usr/share/java/mysql-connector-java.jar
```

> **注意：** 不要使用 yum install 命令安装 MySQL driver 包，因为他会安装 openJDK, 并使用 linux alternatives 命令将系统 JDK 设置为 openJDK。

## 4. 安装 Cloudera Manafer 服务

将 Cloudera Manager Server 安装在安装数据库的服务器上或者可以访问数据库的服务器上。这个服务器不需要是使用 Cloudera Manager 管理的集群的机器。

```
$ sudo yum clean all
$ sudo yum -y install cloudera-manager-server
$ sudo yum -y install cloudera-manager-daemons
```

## 5. 安装 JDK

```
$ sudo yum install oracle-j2sdk1.7
```


## 6. 装备 Cloudera Manager 服务所需的数据库

```
/usr/share/cmf/schema/scm_prepare_database.sh [options] database-type database-name username password
```

1\) Mysql 在本地主机上

```
$ sudo /usr/share/cmf/schema/scm_prepare_database.sh mysql cloudera_manager_service cloudera-scm xxxx
$ sudo /usr/share/cmf/schema/scm_prepare_database.sh mysql cloudera_manager cloudera-scm 
```

2\) Mysql 不在本地主机上

```
$ sudo /usr/share/cmf/schema/scm_prepare_database.sh -he3echdcli02 mysql cloudera_manager_service cloudera-scm xxxx
$ sudo /usr/share/cmf/schema/scm_prepare_database.sh -he3echdcli02 mysql cloudera_manager cloudera-scm xxxx
```

## 7. 启动 Cloudera Manager 服务

```
$ sudo service cloudera-scm-server start
```

或者

```
sudo systemctl start cloudera-scm-server.service
```

## 8. 进入 Cloudera Manager 管理控制台

http://<Server host>:<port>

<Server host> 是安装 Cloudera Manager Server 的服务器的主机名或 IP 地址，<port>是为 Cloudera Manager Server 配置的端口，默认端口 7180。

例：http://127.0.0.1:7180/

# 三、 使用 Cloudera Manager 自动安装 CDH 并配置

> 注：以下有部分步骤并未展示，因为它们可能仅仅是进行简单的默认操作（直接点击继续操作按钮）

## 1. 通过界面安装

在浏览器进入 http://\<cm host\>:7180，进入安装界面

### 1\) 选择 Cloudera Manager 版本

选择将要安装的 Cloudera Manager 版本，我们目前使用的 Express Edition（免费版）；

### 2\) 为 CDH 集群安装指定主机

通过主机名和 IP 地址范围搜索指定的集群主机：

![](/img/2018-06-19-cdh-hadoop-install/1.png)

点击继续，并等待系统自动检查完成：

![](/img/2018-06-19-cdh-hadoop-install/2.png)

### 3\) 选择存储库

因为使用的自定义存储库，所以需要做如下更改：

![](/img/2018-06-19-cdh-hadoop-install/3.png)

![](/img/2018-06-19-cdh-hadoop-install/4.png)


将存储库服务器的 IP 地址，添加到集群中每台服务的 host 文件中，例：

```
$ more /etc/hosts

# ...
172.16.0.10 archive.cloudera.com
# ...
```

在此之前，我们应该检查我们的存储库（填写的远程服务器的主机上）。例如：压缩包位于 /var/www/html/cdh5/parcels/5.4.0/ 目录，cm 位于 /var/www/html/cm5/redhat/6/x86_64/cm/5.4.0/，那么：

```
$ cd  /var/www/html/
$ sudo createrepo  cdh5/parcels/5.4.0/
$ sudo createrepo  cm5/redhat/6/x86_64/cm/5.4.0/
```

或者 

```
$ cd  /var/www/html/
$ sudo createrepo  --update cdh5/parcels/5.4.0
$ sudo createrepo --update  cm5/redhat/6/x86_64/cm/5.4.0/
```

### 4\) JDK 安装选项

![](/img/2018-06-19-cdh-hadoop-install/5.png)

### 5\) 启动单用户运行选项

不使用单用户模式运行，直接点击继续

### 6\) 提供 SSH 凭证

提供 SSH 凭证，用以分发向各节点分发安装包，其中用户为 linux 登录用户。

![](/img/2018-06-19-cdh-hadoop-install/6.png)

### 7\) 安装 JDK 和 Agent

点击继续，系统将自动在各节点安装 JDK 与 angent，等待其完成

![](/img/2018-06-19-cdh-hadoop-install/7.png)

### 8\) 分发 Parcel

继续下一步，系统会自动将 Hadoop 相关服务安装包分发到各节点，等待其完成。

![](/img/2018-06-19-cdh-hadoop-install/8.png)

### 9\) 检查主机正确性

安装完成之后，系统会自动检查主机正确性：

![](/img/2018-06-19-cdh-hadoop-install/9.png)

这里有一个问题，应该是之前配置的问题，后面再解决吧，继续下一步

### 10\) 选择需要安装的服务

选择所需的 Hadoop 相关服务，这里选择了 HBase、HDFS、Spark、YARN、Zookeeper：

![](/img/2018-06-19-cdh-hadoop-install/10.png)

### 11\) 自定义角色分配

选择完需要安装的服务后，就需要为各服务选择对应的主机。根据主机名， Cloudera Manager 的服务安装在以 cli01/cli02 结尾的两台服务器上，其它服务的主节点（除去数据节点，比如 DataNode 和 RegionServer）均安装在以 01 - 05 结尾的服务器上，剩下的譬如 HDFS 的 DataNode、HBase 的 RegionServer、YARN 的 NodeManger 则安装在以 11 ~ 45 结尾的节点上，如下：

![](/img/2018-06-19-cdh-hadoop-install/11.png)

> 注意：上图中 Spark 的 Gateway 请选择与 HiveServer2 相同的主机，否则后面会报错，这里截图时没截对。

### 12\) 数据库设置

接下来，我们需要为 Hive 和 Activity Monitor 设置数据库，数据库就是我们之前创建的数据库。设置完成后点击测试链接并成功后方可继续：

![](/img/2018-06-19-cdh-hadoop-install/12.png)

### 13\) 集群的基础配置

继续，需要配置一些基础配置（请根据实际情况更改，比如接受的 DataNode 失败的卷设置为 0，图片里为 4）：

![](/img/2018-06-19-cdh-hadoop-install/13.png)

### 14\) 自动安装

![](/img/2018-06-19-cdh-hadoop-install/14.png)

## 2. 异常处理

如果在最后的步骤中发生了检查异常，而服务已经安装好的情况，请直接进入 CM 控制台，并根据服务的先后顺序依次安装或配置必须的配置，再进行依次启动。

## 3. 主界面

因为浏览器是中文版，所以尽管系统是英文版，最终还是显示的中文版。

![](/img/2018-06-19-cdh-hadoop-install/15.png)

至此，基本安装完成。