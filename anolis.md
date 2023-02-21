## 查看当前内核版本

```ruby
 uname -a

[root@localhost etc]# cat /etc/anolis-release
Anolis OS release 7.9
```



禁网

```doc
[root@localhost etc]# vim /etc/sysconfig/network-scripts/ifcfg-eth0
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=none
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=eth0
UUID=5860a963-e0be-496d-849d-187bab7f7bb7
DEVICE=eth0
ONBOOT=yes
IPADDR=172.16.52.236
PREFIX=24
GATEWAY=172.16.52.254
#DNS1=202.120.2.101
DNS1=202.120.2.102
IPV6_PRIVACY=no
~               
[root@localhost etc]# service network restart
```

#### 准备工作



```ruby
所有节点依赖安装
yum install -y chkconfig python  bind-utils psmisc libxslt zlib sqlite cyrus-sasl-plain cyrus-sasl-gssapi fuse portmap fuse-libs redhat-lsb
yum install -y vim wget ntp net-tools

所有节点关闭防火墙
systemctl stop firewalld
systemctl disable  firewalld
————————————————————————————————

所有节点
vim /etc/hosts
172.16.52.236 node-1
172.16.52.238 node-2
172.16.52.240 node-3
————————————————————————————————————
所有节点永久更改主机名
hostnamectl set-hostname node-1
hostnamectl set-hostname node-2
hostnamectl set-hostname node-3
查看当前主机名hostname

————————————————————————————————
所有节点设置
vim /etc/security/limits.conf
#添加配置,*代表所有的用户都使用该配置
* soft nofile 65536
* hard nofile 131072
* soft nproc 65536
* hard nproc 65536

——————————————————————————
关闭禁用所有节点在传统 Linux 系统使用访问控制方式的基础上，附加使用 SELinux 可增强系统安全 。
/etc/selinux/config
SELINUX=disabled
重新启动后生效。
——————————————————————————
所有节点免密登录

ssh-keygen -t rsa 
直接回车生成 密钥   
然后copy到 其它节点
ssh-copy-id node-2
ssh-copy-id node-3


#所有节点修改
vim /etc/sysctl.conf
vm.max_map_count=655360
fs.file-max=655360
sysctl -p 查看是否修改成功

[root@node-1 soft]# vim /etc/sysctl.conf
[root@node-1 soft]# sysctl -p
vm.max_map_count = 655360
fs.file-max = 655360
[root@node-1 soft]#

修改Linux swappiness参数(所有节点）
为了避免服务器使用swap功能而影响服务器性能，一般都会把vm.swappiness修改为0（cloudera建议10以下）
vim /etc/sysctl.conf
在最后添加vm.swappiness=0
[root@node-1 soft]# echo "vm.swappiness = 0" >> /etc/sysctl.conf
[root@node-1 soft]# cat /etc/sysctl.conf 
# sysctl settings are defined through files in
# /usr/lib/sysctl.d/, /run/sysctl.d/, and /etc/sysctl.d/.
#
# Vendors settings live in /usr/lib/sysctl.d/.
# To override a whole file, create a new file with the same in
# /etc/sysctl.d/ and put new settings there. To override
# only specific settings, add a file with a lexically later
# name in /etc/sysctl.d/ and put new settings there.
#
# For more information, see sysctl.conf(5) and sysctl.d(5).
#
#

vm.max_map_count=655360
fs.file-max=655360
vm.swappiness = 0
[root@node-2 soft]# 

# Cloudera建议将交换空间设置为0，过多的交换空间会引起GC耗时的激增。

每台机器关闭 chrony 服务，该服务会影响到NTP服务的开机启动
  systemctl stop chronyd &&  systemctl disable chronyd 
```



####1.搭建本地镜像源

```ruby
查询服务器是否已经安装httpd
 rpm -qa|grep httpd
有就不用安装了。没有就要安装。
——————————————————————————
挂载即解压
mount -t iso9660 -o loop /home/CentOS-7.5-x86_64-DVD-1804.iso  /home/mirrors
    
镜像地址cd  /home/mirrors
 
——————————————————————————————————————————————————
cd /etc/yum.repos.d/
[root@localhost yum.repos.d]# pwd
/etc/yum.repos.d
设置本地yum源

[root@localhost yum.repos.d]# vim sa-base.repo

[sa-local]
#起个名字
name=anolis7.9-sa
#镜像挂载的path
baseurl=file:///home/mirrors/
#启用该镜像
enabled=1
#启用校验 1：校验/0：不校验
gpgcheck=1
#校验的密钥路径（看看你本机有没有这个文件）
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-ANOLIS

：wq编辑保存


----  vim 下面所有的 enabled=0
    -rw-r--r--. 1 root root 836 11月 30 2021 AnolisOS-Debuginfo.repo
-rw-r--r--  1 root root 175 2月   8 15:33 AnolisOS-extras.repo
-rw-r--r--  1 root root 163 2月   8 15:33 AnolisOS-os.repo
-rw-r--r--. 1 root root 169 11月 30 2021 AnolisOS-Plus.repo
-rw-r--r--. 1 root root 780 11月 30 2021 AnolisOS-Source.repo
-rw-r--r--. 1 root root 178 11月 30 2021 AnolisOS-updates.repo

    ----

[root@localhost yum.repos.d]# yum clean all
已加载插件：fastestmirror, langpacks
正在清理软件源： sa-local
Cleaning up list of fastest mirrors
[root@localhost yum.repos.d]# yum makecache
已加载插件：fastestmirror, langpacks
Determining fastest mirrors
sa-local                                                                                                                                                                                                                  | 3.8 kB  00:00:00     
(1/4): sa-local/group_gz                                                                                                                                                                                                  | 152 kB  00:00:00     
(2/4): sa-local/primary_db                                                                                                                                                                                                | 6.1 MB  00:00:00     
(3/4): sa-local/filelists_db                                                                                                                                                                                              | 6.7 MB  00:00:00     
(4/4): sa-local/other_db                                                                                                                                                                                                  | 2.2 MB  00:00:00     
元数据缓存已建立
[root@localhost yum.repos.d]# 
    


#清除
yum clean all
#重建缓存
yum makecache
#查看
yum list

```

####2.安装httpd

```ruby
[root@localhost yum.repos.d]# yum install -y httpd
[root@localhost yum.repos.d]# systemctl status httpd
[root@localhost yum.repos.d]# systemctl start httpd
[root@node-1 ~]# systemctl is-enabled httpd
disabled
[root@node-1 ~]# systemctl enable httpd
Created symlink from /etc/systemd/system/multi-user.target.wants/httpd.service to /usr/lib/systemd/system/httpd.service.
[root@node-1 ~]# systemctl is-enabled httpd
enabled
[root@node-1 ~]# 

```

####3.建立局域网下的yum源

```ruby
[root@node-1 home]# mkdir -p /home/mirrors
挂载即解压
mount -t iso9660 -o loop /home/CentOS-7.5-x86_64-DVD-1804.iso  /home/mirrors

[root@localhost html]# mkdir anolis7.9
[root@localhost html]# cd anolis7.9/
    建立软链接
[root@localhost anolis7.9]# ln -s /home/mirrors/   /var/www/html/anolis7.9/anolis7.9
删除软连接rm -f anolis7.9



http://172.16.52.236/anolis7.9/anolis7.9/
http://172.16.52.236/

所有节点
设置局域网下的yum源
vim /etc/yum.repos.d/sa.repo

[sa-local]
name=sa-local
#baseurl=file:///home/mirrors
baseurl=http://172.16.52.236/anolis7.9/anolis7.9/
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-ANOLIS

#清除
yum clean all
#重建缓存
[root@node-2 yum.repos.d]# yum makecache
已加载插件：fastestmirror, langpacks
Determining fastest mirrors
sa-local                                                                                                                                                                                                                 | 3.8 kB  00:00:00     
(1/4): sa-local/group_gz                                                                                                                                                                                                 | 152 kB  00:00:00     
(2/4): sa-local/primary_db                                                                                                                                                                                               | 6.1 MB  00:00:00     
(3/4): sa-local/filelists_db                                                                                                                                                                                             | 6.7 MB  00:00:00     
(4/4): sa-local/other_db                                                                                                                                                                                                 | 2.2 MB  00:00:00     
元数据缓存已建立
#查看
yum list

yum repolist all # 查看当前已开启的yum源

[root@node-2 yum.repos.d]# yum repolist all 
已加载插件：fastestmirror, langpacks
Loading mirror speeds from cached hostfile
源标识                                                                                                        源名称                                                                                                                 状态
Plus/x86_64                                                                                                   AnolisOS-7.9 - Plus                                                                                                    禁用
Plus-debuginfo/x86_64                                                                                         AnolisOS-7.9 - Plus Debuginfo                                                                                          禁用
Plus-source                                                                                                   AnolisOS-7.9 - Plus Source                                                                                             禁用
extras/x86_64                                                                                                 AnolisOS-7.9 - extras                                                                                                  禁用
extras-debuginfo/x86_64                                                                                       AnolisOS-7.9 - extras Debuginfo                                                                                        禁用
extras-source                                                                                                 AnolisOS-7.9 - extras Source                                                                                           禁用
os/x86_64                                                                                                     AnolisOS-7.9 - os                                                                                                      禁用
os-debuginfo/x86_64                                                                                           AnolisOS-7.9 - os Debuginfo                                                                                            禁用
os-source                                                                                                     AnolisOS-7.9 - os Source                                                                                               禁用
sa-local                                                                                                      sa-local                                                                                                               启用: 7,754
updates/x86_64                                                                                                AnolisOS-7.9 - updates                                                                                                 禁用
updates-debuginfo/x86_64                                                                                      AnolisOS-7.9 - updates Debuginfo                                                                                       禁用
updates-source                                                                                                AnolisOS-7.9 - updates Source                                                                                          禁用
repolist: 7,754
[root@node-2 yum.repos.d]# 
    
(vim下复制行    Esc进入非编辑下，上下方向键光标定位所在行位置 按y两次，在进入 按上下方向键 定位按p 粘贴)

```

**小贴士**

```ruby
yum makecache 报错时查看当前已开启的yum源yum repolist all # 
Could not retrieve mirrorlist http://mirrorlist.centos.org/?release=7&arch=x86_64&repo=extras&infra=stock error was
14: curl#6 - "Could not resolve host: mirrorlist.centos.org; 未知的错误"
网络离线下把不是自己的源 全部禁用或都删除

```

####4.同步时间服务器

```ruby
解决方案：
        各服务器时间同步
    内网就以一台服务器作为时间服务器，其他机器去同步即可
    离线环境下如果没有ntp服务器,
    可以选择一台机器作为时间服务器,
    其他机器作为客户端向该机器获取时间。
    所有机器上都要安装和启动ntp,
    时间服务器和客户端的配置不同,
    服务器配置server 127.127.1.1,
    客户端要配置服务端的真实ip。


当前线上时间服务器安装在 node-3 240上
1.yum安装NTP服务
yum -y install ntp

2. 启动ntp服务
systemctl start ntpd

3. 设置开机自启
systemctl enable ntpd


#240做为服务节点给其它的机器进行同步修改配置
vim /etc/ntp.conf

restrict 172.16.52.0 mask 255.255.255.0 nomodify notrap

#server 0.centos.pool.ntp.org iburst
#server 1.centos.pool.ntp.org iburst
#server 2.centos.pool.ntp.org iburst
#server 3.centos.pool.ntp.org iburst
server 127.127.1.1 

改好配置后 - 一定要重启！！！！！！！！！
systemctl restart ntpd 重启
————————————————————————————————————————————————

#所有其它节点全部皆同步240
yum install -y ntp
vim /etc/ntp.conf
#server 0.centos.pool.ntp.org iburst
#server 1.centos.pool.ntp.org iburst
#server 2.centos.pool.ntp.org iburst
#server 3.centos.pool.ntp.org iburst
serrver node-3

Esc ：wq
systemctl restart ntpd 重启

ntpq -p #查看ntp同步状态,offset是否接近0，越接近说明时间越一致
date #查看时间
[root@cdh-1 ~]# systemctl stop ntpd
[root@cdh-1 ~]# ntpdate node-3  # 手动同步本地时间 服务器

所有机器启动ntp服务
#启动服务(需要启动才能自动同步)
systemctl start ntpd


————————
ntpq -p
#说明
#主机前的 ？ 代表时间服务器不可达,除了* ， + 所代表的服务器外，其余均存在一定的问题。
#remote ：响应这个请求的NTP服务器的名称。
#refid ：NTP服务器使用的上一级ntp服务器。
#st ：remote远程服务器的级别. 由于NTP是层型结构,有顶端的服务器,多层的Relay Server再到客户端.所以服务器从高到低级别可以设定为1-16. 为了减缓负荷和网络堵塞,原则上应该避免直接连接到级别为1的服务器的.
#when: 上一次成功请求之后到现在的秒数。
#poll : 本地机和远程服务器多少时间进行一次同步(单位为秒). 在一开始运行NTP的时候这个poll值会比较小,那样和服务器同步的频率也就增加了,可以尽快调整到正确的时间范围，之后poll值会逐渐增大,同步的频率也就会相应减小
#reach: 这是一个八进制值,用来测试能否和服务器连接.每成功连接一次它的值就会增加
#delay: 从本地机发送同步要求到ntp服务器的round trip time
#offset ：主机通过NTP时钟同步与所同步时间源的时间偏移量，单位为毫秒（ms）。offset越接近于0,主机和ntp服务器的时间越接近
#jitter: 这是一个用来做统计的值. 它统计了在特定个连续的连接数里offset的分布情况. 简单地说这个数值的绝对值越小，主机的时间就越精确
```

####5.jdk安装



```ruby
rpm -qa|grep java
[root@node-2 soft]# rpm -e --nodeps java-1.8.0-openjdk-headless-1.8.0.262.b10-1.an7.x86_64

[root@node-2 soft]# rpm -e --nodeps java-1.8.0-openjdk-1.8.0.262.b10-1.an7.x86_64

[root@node-2 soft]# rpm -e --nodeps javapackages-tools-3.4.1-11.an7.noarch
删错了 从镜像中拿这个javapackages-tools-3.4.1-11.an7.noarch 文件 ，然后 进行rpm -i javapackages-tools-3.4.1-11.an7.noarch  就还原了。
#所有节点安装
[root@node-1 soft]# pwd
/home/soft
[root@node-1 soft]# ll
总用量 135512
-rw-r--r-- 1 root root 138762230 2月   8 14:01 jdk-8u361-linux-x64.tar.gz

[root@node-1 soft]# tar -zxvf jdk-8u361-linux-x64.tar.gz
  ( 想偷懒用后面cdh镜像里面自带的 yum install  oracle-j2sdk1.8.x86_64)

scp -r jdk1.8.0_361   root@node-2:/home/soft/
scp -r jdk1.8.0_361   root@node-3:/home/soft/


vim /etc/profile
export JAVA_HOME=/home/soft/jdk1.8.0_361
export PATH=/bin:/usr/bin:$PATH:$JAVA_HOME/bin
source /etc/profile
[root@node-1 soft]# source /etc/profile
[root@node-1 soft]# java -version
java version "1.8.0_361"
Java(TM) SE Runtime Environment (build 1.8.0_361-b09)
Java HotSpot(TM) 64-Bit Server VM (build 25.361-b09, mixed mode)

```

#### 6.安装mysql

```ruby

for i in $(rpm -qa|grep mysql);do rpm -e $i --nodeps ;done
###查看是否存在 mariadb ，有则卸载。
[root@node-1 soft]# rpm -qa|grep mariadb
mariadb-libs-5.5.68-1.an7.x86_64

[root@node-1 soft]# rpm -e  --nodeps  mariadb-libs-5.5.68-1.an7.x86_64

安装mysql  在 240 node-3 机器上
下载地址 https://downloads.mysql.com/archives/community/
(mysql-5.7.36-1.el7.x86_64.rpm-bundle.tar)

使用rpm 包进行安装
cd /home/soft
mkdir mysql5.7.36


tar -xvf mysql-5.7.36-1.el7.x86_64.rpm-bundle.tar  -C mysql5.7.36/
[root@node-3 soft]# cd mysql5.7.36/
[root@node-3 mysql5.7.36]# rm -f mysql-community-embedded-*
[root@node-3 mysql5.7.36]# rm -f mysql-community-test-5.7.36-1.el7.x86_64.rpm

[root@node-3 mysql5.7.36]# rpm -Uvh --force --nodeps *rpm
启动mysql
[root@node-3 mysql5.7.36]# systemctl start mysqld
[root@node-3 mysql5.7.36]# systemctl enable mysqld
首次查看登录密码
[root@node-3 mysql5.7.36]# grep password /var/log/mysqld.log
2023-02-09T03:30:06.274489Z 1 [Note] A temporary password is generated for root@localhost: o&mfghuOt3#H

[root@node-3 mysql5.7.36]# mysql -uroot -p 


mysql> alter user 'root'@'localhost' identified by 'Py1q2w3e.';
Query OK, 0 rows affected (0.00 sec)



mysql> grant all privileges on *.* to 'root'@'%'identified by 'Py1q2w3e.' with grant option;
Query OK, 0 rows affected, 1 warning (0.00 sec)



### 配置root可以通过第三方的外部ip进行访问登录授权。
### update mysql.user set host='%' where user='root'



mysql> flush privileges;
Query OK, 0 rows affected (0.00 sec)

mysql> exit
Bye

Mysql 查看版本号  mysql -V
[root@node-3 mysql5.7.36]# mysql -V
mysql  Ver 14.14 Distrib 5.7.36, for Linux (x86_64) using  EditLine wrapper

下载上传 mysql驱动包
mkdir -p /usr/share/java
[root@cdh-2 opt]# scp /usr/share/java/mysql-connector-java.jar    root@172.16.52.240:/usr/share/java/
 
    
 #安装Cloudera Manager前 先去mysql 补充新建一些库  

    
[root@node-3 mysql5.7.36]# mysql -uroot -pPy1q2w3e.

CREATE DATABASE scm DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE amon DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE rman DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE hue DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE metastore DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE sentry DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE nav DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE navms DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE oozie DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE hive DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
GRANT ALL ON scm.* TO 'scm'@'%' IDENTIFIED BY 'Py1q2w3e.';
GRANT ALL ON amon.* TO 'amon'@'%' IDENTIFIED BY 'Py1q2w3e.';
GRANT ALL ON rman.* TO 'rman'@'%' IDENTIFIED BY 'Py1q2w3e.';
GRANT ALL ON hue.* TO 'hue'@'%' IDENTIFIED BY 'Py1q2w3e.';
GRANT ALL ON metastore.* TO 'hive'@'%' IDENTIFIED BY 'Py1q2w3e.';
GRANT ALL ON sentry.* TO 'sentry'@'%' IDENTIFIED BY 'Py1q2w3e.';
GRANT ALL ON nav.* TO 'nav'@'%' IDENTIFIED BY 'Py1q2w3e.';
GRANT ALL ON navms.* TO 'navms'@'%' IDENTIFIED BY 'Py1q2w3e.';
GRANT ALL ON oozie.* TO 'oozie'@'%' IDENTIFIED BY 'Py1q2w3e.';
GRANT ALL ON hive.* TO 'hive'@'%' IDENTIFIED BY 'Py1q2w3e.';
mysql> flush privileges;
Query OK, 0 rows affected (0.00 sec)
mysql> exit
Bye
    
```



#### 6-2 mysql8.0.25

```ruby
#下载地址 https://downloads.mysql.com/archives/community/
[root@node-2 soft]# wget https://downloads.mysql.com/archives/get/p/23/file/mysql-8.0.25-1.el7.x86_64.rpm-bundle.tar

[root@node-2 soft]# tar -xvf mysql-8.0.25-1.el7.x86_64.rpm-bundle.tar  -C mysql8.0.25/
[root@node-2 mysql8.0.25]# rpm -Uvh --force --nodeps *rpm

启动mysql
[root@node-2 mysql8.0.25]# systemctl start mysqld
[root@node-2 mysql8.0.25]# systemctl enable mysqld
首次查看登录密码
[root@node-2 mysql8.0.25]#  grep password /var/log/mysqld.log
2023-02-09T08:28:25.310106Z 6 [Note] [MY-010454] [Server] A temporary password is generated for root@localhost: R&Tu/p1846io

    
[root@node-2 mysql8.0.25]# mysql -uroot -p


mysql> alter user 'root'@'localhost' identified by 'Py1q2w3e.';
Query OK, 0 rows affected (0.00 sec)
mysql8最新套路  必须分3步来实现设置用户权限：
（1）先创建用户
（2）再对该用户分配用户权限
（3）最后刷新权限
mysql> create user test@'localhost' identified by 'Py123456.';
Query OK, 0 rows affected (0.01 sec)

mysql> grant all privileges on test.* to test@'localhost';
Query OK, 0 rows affected (0.00 sec)

mysql> flush privileges;
Query OK, 0 rows affected (0.00 sec)

mysql> create user root@'%' identified by 'Py1q2w3e.';
Query OK, 0 rows affected (0.01 sec)

mysql> grant all privileges on *.* to root@'%';
Query OK, 0 rows affected (0.01 sec)

mysql> flush privileges;
Query OK, 0 rows affected (0.01 sec)


mysql> exit
Bye

Mysql 查看版本号  mysql -V
[root@node-2 mysql8.0.25]# mysql -V
mysql  Ver 8.0.25 for Linux on x86_64 (MySQL Community Server - GPL)


下载上传 mysql驱动包
mkdir -p /usr/share/java
[root@cdh-2 opt]# scp /usr/share/java/mysql-connector-java.jar    root@172.16.52.240:/usr/share/java/
 
    
 #安装Cloudera Manager前 先去mysql 补充新建一些库  

    
[root@node-2 /]# mysql -uroot -pPy1q2w3e.

CREATE DATABASE scm DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE amon DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE rman DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE hue DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE metastore DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE sentry DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE nav DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE navms DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE oozie DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE hive DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;

create user scm@'%' identified by 'Py1q2w3e.';
grant all privileges on scm.* to scm@'%';
flush privileges;
create user hive@'%' identified by 'Py1q2w3e.';
grant all privileges on hive.* to hive@'%';
grant all privileges on metastore.* to hive@'%';
create user amon@'%' identified by 'Py1q2w3e.';
grant all privileges on amon.* to amon@'%';
create user rman@'%' identified by 'Py1q2w3e.';
grant all privileges on rman.* to rman@'%';
create user hue@'%' identified by 'Py1q2w3e.';
grant all privileges on hue.* to hue@'%';
create user oozie@'%' identified by 'Py1q2w3e.';
grant all privileges on oozie.* to oozie@'%';
    flush privileges;
GRANT ALL ON scm.* TO 'scm'@'%' IDENTIFIED BY 'Py1q2w3e.';
GRANT ALL ON amon.* TO 'amon'@'%' IDENTIFIED BY 'Py1q2w3e.';
GRANT ALL ON rman.* TO 'rman'@'%' IDENTIFIED BY 'Py1q2w3e.';
GRANT ALL ON hue.* TO 'hue'@'%' IDENTIFIED BY 'Py1q2w3e.';
GRANT ALL ON metastore.* TO 'hive'@'%' IDENTIFIED BY 'Py1q2w3e.';
GRANT ALL ON sentry.* TO 'sentry'@'%' IDENTIFIED BY 'Py1q2w3e.';
GRANT ALL ON nav.* TO 'nav'@'%' IDENTIFIED BY 'Py1q2w3e.';
GRANT ALL ON navms.* TO 'navms'@'%' IDENTIFIED BY 'Py1q2w3e.';
GRANT ALL ON oozie.* TO 'oozie'@'%' IDENTIFIED BY 'Py1q2w3e.';
GRANT ALL ON hive.* TO 'hive'@'%' IDENTIFIED BY 'Py1q2w3e.';
mysql> flush privileges;
Query OK, 0 rows affected (0.00 sec)
mysql> exit
Bye
    
```





#### 7.createRepo

```ruby
[root@node-1 soft]# yum install -y createrepo

[root@node-1 soft]# cd /var/www/html/
[root@node-1 html]# mkdir cdh
[root@node-1 html]# mkdir clouderaManger
[root@node-1 html]# ll
总用量 0
drwsr-sr-x 2 root root 38 2月   8 17:22 anolis7.9
drwxr-xr-x 2 root root  6 2月   9 13:48 cdh
drwxr-xr-x 2 root root  6 2月   9 13:48 clouderaManger
[root@node-1 html]# cd cdh/
#上传cdh6.3.0 parcel
[root@node-1 cdh]# ll
总用量 2036852
-rw-r--r-- 1 root root 2085690155 2月   9 13:50 CDH-6.3.0-1.cdh6.3.0.p0.1279813-el7.parcel
-rw-r--r-- 1 root root         40 2月   9 13:50 CDH-6.3.0-1.cdh6.3.0.p0.1279813-el7.parcel.sha
-rw-r--r-- 1 root root         64 2月   9 13:50 CDH-6.3.0-1.cdh6.3.0.p0.1279813-el7.parcel.sha256
-rw-r--r-- 1 root root      33887 2月   9 13:50 manifest.json


[root@node-1 html]# cd clouderaManger/
[root@node-1 clouderaManger]# ll
总用量 1380436
-rw-r--r-- 1 root root      14041 2月   9 13:53 allkeys.asc
-rw-r--r-- 1 root root   10483568 2月   9 13:53 cloudera-manager-agent-6.3.1-1466458.el7.x86_64.rpm
-rw-r--r-- 1 root root 1203832464 2月   9 13:53 cloudera-manager-daemons-6.3.1-1466458.el7.x86_64.rpm
-rw-r--r-- 1 root root      11488 2月   9 13:53 cloudera-manager-server-6.3.1-1466458.el7.x86_64.rpm
-rw-r--r-- 1 root root      10996 2月   9 13:53 cloudera-manager-server-db-2-6.3.1-1466458.el7.x86_64.rpm
-rw-r--r-- 1 root root   14209868 2月   9 13:53 enterprise-debuginfo-6.3.1-1466458.el7.x86_64.rpm
-rw-r--r-- 1 root root  184988341 2月   9 13:53 oracle-j2sdk1.8-1.8.0+update181-1.x86_64.rpm

[root@node-1 clouderaManger]# createrepo .
Spawning worker 0 with 1 pkgs
Spawning worker 1 with 1 pkgs
Spawning worker 2 with 1 pkgs
Spawning worker 3 with 1 pkgs
Spawning worker 4 with 1 pkgs
Spawning worker 5 with 1 pkgs
Spawning worker 6 with 0 pkgs
Spawning worker 7 with 0 pkgs
Workers Finished
Saving Primary metadata
Saving file lists metadata
Saving other metadata
Generating sqlite DBs
Sqlite DBs complete
[root@node-1 clouderaManger]# ll
总用量 1380440
-rw-r--r-- 1 root root      14041 2月   9 13:53 allkeys.asc
-rw-r--r-- 1 root root   10483568 2月   9 13:53 cloudera-manager-agent-6.3.1-1466458.el7.x86_64.rpm
-rw-r--r-- 1 root root 1203832464 2月   9 13:53 cloudera-manager-daemons-6.3.1-1466458.el7.x86_64.rpm
-rw-r--r-- 1 root root      11488 2月   9 13:53 cloudera-manager-server-6.3.1-1466458.el7.x86_64.rpm
-rw-r--r-- 1 root root      10996 2月   9 13:53 cloudera-manager-server-db-2-6.3.1-1466458.el7.x86_64.rpm
-rw-r--r-- 1 root root   14209868 2月   9 13:53 enterprise-debuginfo-6.3.1-1466458.el7.x86_64.rpm
-rw-r--r-- 1 root root  184988341 2月   9 13:53 oracle-j2sdk1.8-1.8.0+update181-1.x86_64.rpm
drwxr-xr-x 2 root root       4096 2月   9 13:55 repodata



[root@cdh-1 clouderaManger]# 

[root@node-1 clouderaManger]# vim /etc/yum.repos.d/cm.repo
[clouderaManager]
name=cm6.3.1
baseurl=http://172.16.52.236/clouderaManger
enabled=1
gpgcheck=0

[root@node-1 clouderaManger]# yum clean all
已加载插件：fastestmirror, langpacks
正在清理软件源： clouderaManager sa-local
Cleaning up list of fastest mirrors
[root@node-1 clouderaManger]# yum makecache
已加载插件：fastestmirror, langpacks
Determining fastest mirrors
clouderaManager                                                                                                                                                                                                          | 2.9 kB  00:00:00     
sa-local                                                                                                                                                                                                                 | 3.8 kB  00:00:00     
(1/7): clouderaManager/filelists_db                                                                                                                                                                                      | 118 kB  00:00:00     
(2/7): clouderaManager/other_db                                                                                                                                                                                          | 1.0 kB  00:00:00     
(3/7): clouderaManager/primary_db                                                                                                                                                                                        | 8.6 kB  00:00:00     
(4/7): sa-local/group_gz                                                                                                                                                                                                 | 152 kB  00:00:00     
(5/7): sa-local/filelists_db                                                                                                                                                                                             | 6.7 MB  00:00:00     
(6/7): sa-local/primary_db                                                                                                                                                                                               | 6.1 MB  00:00:00     
(7/7): sa-local/other_db                                                                                                                                                                                                 | 2.2 MB  00:00:00     
元数据缓存已建立



[root@node-1 clouderaManger]# yum repolist
已加载插件：fastestmirror, langpacks
Loading mirror speeds from cached hostfile
源标识                                                                                                                 源名称                                                                                                              状态
clouderaManager                                                                                                        cm6.3.1                                                                                                                 6
sa-local                                                                                                               anolis7.9-sa                                                                                                        7,754
repolist: 7,760


#所有节点都要 将地址配置到yum源中，进入到 /etc/yum.repos.d/
[root@node-1 yum.repos.d]# scp cm.repo  node-2:/etc/yum.repos.d/
cm.repo                                                                                                                                                                                                       100%   96    66.4KB/s   00:00    
[root@node-1 yum.repos.d]# scp cm.repo  node-3:/etc/yum.repos.d/
cm.repo                

[root@node-2 root]# yum clean all
[root@node-2 root]# yum makecache
[root@node-3 mysql5.7.36]# yum clean all
[root@node-3 mysql5.7.36]# yum makecache
```

#### 8.安装cloudera-manager

```ruby
#主节点安装server
[root@node-1 yum.repos.d]#  yum install -y cloudera-manager-agent cloudera-manager-daemons cloudera-manager-server 
 
#其它 从节点安装  agent 
[root@node-2 java]# yum install cloudera-manager-daemons cloudera-manager-agent -y
[root@node-3 java]# yum install cloudera-manager-daemons cloudera-manager-agent -y

[root@node-1 opt]# cd /opt/cloudera/cm/schema/
[root@node-1 schema]# ll
总用量 60
drwxr-xr-x 4 root root  8192 2月   9 14:15 mysql
drwxr-xr-x 4 root root  8192 2月   9 14:15 oracle
drwxr-xr-x 4 root root 12288 2月   9 14:15 postgresql
-rw-r--r-- 1 root root  1437 9月  25 2019 scm_database_functions.sh
-rwxr-xr-x 1 root root 12450 9月  25 2019 scm_prepare_database.sh

#Cloudera Manager Server有一个配置数据库的脚本，执行该脚本就好
如果MySQL数据库与CDH Server在同一台主机上，执行如下命令
 /opt/cloudera/cm/schema/scm_prepare_database.sh mysql scm scm
否则执行下面：
 ./scm_prepare_database.sh mysql -hnode-2   --scm-host node-1 scm scm Py1q2w3e.  [8版本有效]
./scm_prepare_database.sh mysql -h node-3 -uroot -pPy1q2w3e. --scm-host node-1 scm scm Py1q2w3e. 【5版本可以】
 （对应于：数据库类型、数据库服务器地址、数据库用户名、数据库密码、CMServer所在节点…….） 第一个scm 库名，第二个用户名，第三个，是scm库密码

[root@node-1 schema]# ./scm_prepare_database.sh mysql -h node-3 -uroot -pPy1q2w3e. --scm-host node-1 scm scm Py1q2w3e.
JAVA_HOME=/home/soft/jdk1.8.0_361
Verifying that we can write to /etc/cloudera-scm-server
[                          main] DbProvisioner                  ERROR Unable to find the MySQL JDBC driver. Please make sure that you have installed it as per instruction in the installation guide.
[                          main] DbProvisioner                  ERROR Stack Trace:
java.lang.ClassNotFoundException: com.mysql.jdbc.Driver
	at java.net.URLClassLoader.findClass(URLClassLoader.java:387)[:1.8.0_361]
	at java.lang.ClassLoader.loadClass(ClassLoader.java:418)[:1.8.0_361]
	at sun.misc.Launcher$AppClassLoader.loadClass(Launcher.java:355)[:1.8.0_361]
	at java.lang.ClassLoader.loadClass(ClassLoader.java:351)[:1.8.0_361]
	at java.lang.Class.forName0(Native Method)[:1.8.0_361]
	at java.lang.Class.forName(Class.java:264)[:1.8.0_361]
	at com.cloudera.enterprise.dbutil.DbProvisioner.executeSql(DbProvisioner.java:283)[db-common-6.3.1.96818eaab0a222aa84a7854b8d22c0c7.jar:]
	at com.cloudera.enterprise.dbutil.DbProvisioner.doMain(DbProvisioner.java:104)[db-common-6.3.1.96818eaab0a222aa84a7854b8d22c0c7.jar:]
	at com.cloudera.enterprise.dbutil.DbProvisioner.main(DbProvisioner.java:123)[db-common-6.3.1.96818eaab0a222aa84a7854b8d22c0c7.jar:]
--> Error 1, giving up (use --force if you wish to ignore the error)

####error 少驱动 #############
[root@node-3 mysql5.7.36]# scp /usr/share/java/mysql-connector-java.jar    root@node-2:/usr/share/java/


####再次报错----之前主机名写错了。######
The last packet successfully received from the server was 261 milliseconds ago.  The last packet sent successfully to the server was 253 milliseconds ago.
# rpm -e 卸载cloudera所有.rpm  ,进入mysql 删除 原来的 scm库 重新建立scm库 ，重新安装node-1 [root@node-1 schema]# rpm -qa|grep cloudera
cloudera-manager-agent-6.3.1-1466458.el7.x86_64
cloudera-manager-server-6.3.1-1466458.el7.x86_64
cloudera-manager-daemons-6.3.1-1466458.el7.x86_64
[root@node-1 schema]# for i in $(rpm -qa|grep cloudera);do rpm -e $i --nodeps ;done
[root@node-1 schema]# rpm -qa|grep cloudera
[root@node-1 schema]# 
[root@node-1 schema]# rm -rf /opt

#重新安装node-1  cloudera-manager
[root@node-1 schema]# ./scm_prepare_database.sh mysql -hnode-2 -uroot -pPy1q2w3e.  --scm-host node-1 scm scm scm
JAVA_HOME=/home/soft/jdk1.8.0_361
Verifying that we can write to /etc/cloudera-scm-server
Loading class `com.mysql.jdbc.Driver'. This is deprecated. The new driver class is `com.mysql.cj.jdbc.Driver'. The driver is automatically registered via the SPI and manual loading of the driver class is generally unnecessary.
[                          main] DbProvisioner                  ERROR Exception when creating/dropping database with user 'root' and jdbc url 'jdbc:mysql://node-2/?useUnicode=true&characterEncoding=UTF-8'
java.sql.SQLSyntaxErrorException: You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near 'identified by 'scm'' at line 1
	at com.mysql.cj.jdbc.exceptions.SQLError.createSQLException(SQLError.java:120)[mysql-connector-java.jar:8.0.25]
	at com.mysql.cj.jdbc.exceptions.SQLExceptionsMapping.translateException(SQLExceptionsMapping.java:122)[mysql-connector-java.jar:8.0.25]
	at com.mysql.cj.jdbc.StatementImpl.executeInternal(StatementImpl.java:762)[mysql-connector-java.jar:8.0.25]
	at com.mysql.cj.jdbc.StatementImpl.execute(StatementImpl.java:646)[mysql-connector-java.jar:8.0.25]
	at com.cloudera.enterprise.dbutil.DbProvisioner.executeSql(DbProvisioner.java:299)[db-common-6.3.1.96818eaab0a222aa84a7854b8d22c0c7.jar:]
	at com.cloudera.enterprise.dbutil.DbProvisioner.doMain(DbProvisioner.java:104)[db-common-6.3.1.96818eaab0a222aa84a7854b8d22c0c7.jar:]
	at com.cloudera.enterprise.dbutil.DbProvisioner.main(DbProvisioner.java:123)[db-common-6.3.1.96818eaab0a222aa84a7854b8d22c0c7.jar:]
[                          main] DbProvisioner                  ERROR Stack Trace:
java.sql.SQLSyntaxErrorException: You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near 'identified by 'scm'' at line 1
	at com.mysql.cj.jdbc.exceptions.SQLError.createSQLException(SQLError.java:120)[mysql-connector-java.jar:8.0.25]
	at com.mysql.cj.jdbc.exceptions.SQLExceptionsMapping.translateException(SQLExceptionsMapping.java:122)[mysql-connector-java.jar:8.0.25]
	at com.mysql.cj.jdbc.StatementImpl.executeInternal(StatementImpl.java:762)[mysql-connector-java.jar:8.0.25]
	at com.mysql.cj.jdbc.StatementImpl.execute(StatementImpl.java:646)[mysql-connector-java.jar:8.0.25]
	at com.cloudera.enterprise.dbutil.DbProvisioner.executeSql(DbProvisioner.java:299)[db-common-6.3.1.96818eaab0a222aa84a7854b8d22c0c7.jar:]
	at com.cloudera.enterprise.dbutil.DbProvisioner.doMain(DbProvisioner.java:104)[db-common-6.3.1.96818eaab0a222aa84a7854b8d22c0c7.jar:]
	at com.cloudera.enterprise.dbutil.DbProvisioner.main(DbProvisioner.java:123)[db-common-6.3.1.96818eaab0a222aa84a7854b8d22c0c7.jar:]
--> Error 1, giving up (use --force if you wish to ignore the error)
————————————————————————————————————————————————————————————————————————————————————————




最后最终
[root@node-1 schema]# ./scm_prepare_database.sh mysql -hnode-2   --scm-host node-1 scm scm Py1q2w3e.
JAVA_HOME=/home/soft/jdk1.8.0_361
Verifying that we can write to /etc/cloudera-scm-server
Creating SCM configuration file in /etc/cloudera-scm-server
Executing:  /home/soft/jdk1.8.0_361/bin/java -cp /usr/share/java/mysql-connector-java.jar:/usr/share/java/oracle-connector-java.jar:/usr/share/java/postgresql-connector-java.jar:/opt/cloudera/cm/schema/../lib/* com.cloudera.enterprise.dbutil.DbCommandExecutor /etc/cloudera-scm-server/db.properties com.cloudera.cmf.db.
Loading class `com.mysql.jdbc.Driver'. This is deprecated. The new driver class is `com.mysql.cj.jdbc.Driver'. The driver is automatically registered via the SPI and manual loading of the driver class is generally unnecessary.
[                          main] DbCommandExecutor              INFO  Successfully connected to database.
All done, your SCM database is configured correctly!
[root@node-1 schema]# 



```

#### 9.CM配置

```ruby
[root@node-1 schema]#vim /etc/cloudera-scm-agent/config.ini

[General]
# Hostname of the CM server.
server_host=node-1




```

