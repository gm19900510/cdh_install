# Cloudera Manager 技术架构
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200717083748821.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N0d3kyOTEzMTQ=,size_16,color_FFFFFF,t_70)
- Agent：安装在每台主机上。该代理负责启动和停止的过程，拆包配置，触发装置和监控主机。
- Management Service：由一组执行各种监控，警报和报告功能角色的服务。
- Database：存储配置和监视信息。通常情况下，多个逻辑数据库在一个或多个数据库服务器上运行。例如，Cloudera的管理服务器和监控角色使用不同的逻辑数据库。
- Cloudera Repository：软件由Cloudera 管理分布存储库。
- Clients：是用于与服务器进行交互的接口：
- Admin Console ：基于Web的用户界面与管理员管理集群和Cloudera管理。
- API ：与开发人员创建自定义的Cloudera Manager应用程序的API。

# Cloudera Manager 四大功能
1.管理：对集群进行管理，如添加、删除节点等操作。

2.监控：监控集群的健康情况，对设置的各种指标和系统运行情况进行全面监控。

3.诊断：对集群出现的问题进行诊断，对出现的问题给出建议解决方案。

4.集成：对hadoop的多组件进行整合。

# CDH 介绍
CDH (Cloudera’s Distribution, including Apache Hadoop)，是Hadoop众多分支中的一种，由Cloudera维护，基于稳定版本的Apache Hadoop构建，并集成了很多补丁，可直接用于生产环境。

# 环境介绍
查看系统：`cat /etc/redhat-release`
```bash
CentOS Linux release 7.7.1908 (Core)
```
查看机器名称：`hostname`
```bash
S0
```
完整集群环境：
IP     | 主机名
-------- | -----
192.168.3.9  | S0
192.168.3.18  | S1
192.168.3.161  | S2
192.168.3.136 | S3

# 基础环境配置

### SSH免密登录
> 所有节点都需要执行 生成秘钥 然后发送到其他所有节点 实现ssh免密码登录 

如下是主节点的操作： 
```bash
yum -y install openssh-clients                 #安装ssh
ssh-keygen -t rsa                              #一直按回车 生成秘钥
ssh-copy-id 192.168.3.18	                   #发送到192.168.3.18节点   
ssh-copy-id 192.168.3.161	                   #发送到192.168.3.161节点
ssh-copy-id 192.168.3.136                      #发送到192.168.3.136节点                
```
> 其他节点也如此操作，需修改发送到节点地址

### 配置主机
```bash
hostnamectl set-hostname S0
```
> S0为主机名称，具体的内容根据实际的IP和主机名自行修改

### 设置网络
```bash
vi /etc/sysconfig/network
```
### 设置`host`
```bash
vi /etc/hosts

配置以下内容：
192.168.3.9 S0
192.168.3.18 S1
192.168.3.161 S2
192.168.3.163 S3
```
> 具体的内容根据实际的IP和主机名自行修改

将hosts文件拷贝到其他机器
```bash
scp /etc/hosts root@192.168.3.18:/etc/
scp /etc/hosts root@192.168.3.161:/etc/
scp /etc/hosts root@192.168.3.136:/etc/
```
### 关闭防火墙
查看防火墙状态
```bash
firewall-cmd --state
```
停止firewall
```bash
systemctl stop firewalld.service
```
禁止firewall开机启动
```bash
systemctl disable firewalld.service 
```
### 关闭`selinux`
查看SELinux状态
```bash
/usr/sbin/sestatus -v 
```
> 如果SELinux status参数为enable，即开启状态

`getenforce` 也可以用这个命令检查

临时关闭selinux，命令：`setenforce 0`

永久关闭selinux，`修改配置文件需要重启机器`
```bash
vi /etc/selinux/config

#将SELINUX=enforcing改成SELINUX=disabled 
```
将selinux 文件拷贝到其他机器
```bash
scp /etc/sysconfig/selinux root@192.168.3.18:/etc/sysconfig/
scp /etc/sysconfig/selinux root@192.168.3.161:/etc/sysconfig/
scp /etc/sysconfig/selinux root@192.168.3.136:/etc/sysconfig/
```
>再次强调，`重启各服务器`
### 安装JDK
首先要准备java环境 安装jdk 设置JAVA_HOME环境变量
```bash
/usr/java/jdk1.8.0_251-amd64
```
> 注意： `jdk要安装在/usr/java/ 里` ，否则Cloudera Manager找不到会报错

也可选用`rpm -i jdk-8u251-linux-x64.rpm` 安装

配置环境变量
```bash
vi /etc/profile
```
```bash
export JAVA_HOME=/usr/java/jdk1.8.0_251-amd64
export JRE_HOME=${JAVA_HOME}/jre
export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib
export PATH=${JAVA_HOME}/bin:$PATH
```
重新加载profile使配置生效
```bash
source /etc/profile
```
环境变量配置完成，测试环境变量是否生效
```bash
echo $JAVA_HOME
java -version
```
### 安装NTP
```bash
yum install ntp -y
```
设置开机启动 
```bash
chkconfig ntpd on
```
设置时间同步 
```bash
ntpdate -u s2c.time.edu.cn
```
查看服务状态
```bash
ntpq -p
```

# 安装数据库服务
- 安装mysql
	```bash
	wget -i -c http://dev.mysql.com/get/mysql57-community-release-el7-10.noarch.rpm
	yum -y install mysql57-community-release-el7-10.noarch.rpm
	yum -y install mysql-community-server
	```
- 设置开机启动
	```bash
	systemctl enable mysqld.service
	```
- 启动服务并查看服务状态
	```bash
	systemctl start  mysqld.service
	systemctl status mysqld.service
	```
- 查看初始密码
	```bash
	grep "password" /var/log/mysqld.log
	```
	![在这里插入图片描述](https://img-blog.csdnimg.cn/20200717102615194.png)
- 进入数据库
	```bash
	mysql -uroot -p     # 回车后会提示输入密码，密码为上面的初始密码
	```
- 修改默认密码
	```bash
	ALTER USER 'root'@'localhost' IDENTIFIED BY 'new password';
	```
	里有个问题，新密码设置的时候如果设置的过于简单会报错：
	![在这里插入图片描述](https://img-blog.csdnimg.cn/2020071710293927.png)
	> 原因是因为MySQL有密码设置的规范，具体是与validate_password_policy的值有关

	如果不需要密码策略，在my.cnf文件中添加如下配置禁用即可，这样就可以设置简单密码了：
	```bash
	vi /etc/my.cnf
	```
	```bash
	validate_password = off
	```
	> yum安装的默认在/etc文件夹下，修改完后记得需要重新启动MySQL服务
- 设置远程登录
	```bash
	GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY '1qaz@wsx' WITH GRANT OPTION;
	FLUSH PRIVILEGES;
	```
	先设置刚才的密码可以远程登录，然后使用flush命令使配置立即生效。
	如果还不行可以尝试重启一下数据库
			![在这里插入图片描述](https://img-blog.csdnimg.cn/2020071710354413.png)
- **安装数据库驱动**
	```bash
	mkdir -p /usr/share/java
	cp mysql-connector-java-5.1.44-bin.jar /usr/share/java/mysql-connector-java.jar
	```

# 下载CM与CDH
CM我选的是5，下载地址：[http://archive.cloudera.com/cm5/cm/5/](http://archive.cloudera.com/cm5/cm/5/)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200717104906588.png)

CHD选择要与CM版本对应，下载地址：[http://archive.cloudera.com/cdh5/parcels/5.16.2/](http://archive.cloudera.com/cdh5/parcels/5.16.2/)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200717105115211.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N0d3kyOTEzMTQ=,size_16,color_FFFFFF,t_70)
> 需要下载3个文件，	注意： ==一定要修改sha1名称为sha==

将上述三个文件上传至主节点	
# Cloudera Manager Server的安装
> 在主节点进行以下配置，我的S0

创建安装目录并解压安装介质
```bash
mkdir /opt/cloudera-manager
tar xzf cloudera-manager*.tar.gz -C /opt/cloudera-manager
```
安装数据库驱动
```bash
mkdir -p /usr/share/java
cp mysql-connector-java-5.1.44-bin.jar /usr/share/java/mysql-connector-java.jar
```
> 一定要这个名字（mysql-connector-java.jar），否则会报错Unable to find JDBC driver for database type: MySQL

创建系统用户cloudera-scm
```bash
useradd --system --home=/opt/cloudera-manager/cm-5.16.2/run/cloudera-scm-server --no-create-home --shell=/bin/false --comment "Cloudera SCM User" cloudera-scm
```
创建server存储目录
```bash
mkdir /var/lib/cloudera-scm-server
chown cloudera-scm:cloudera-scm /var/lib/cloudera-scm-server
```
创建hadoop安装包存储目录
```bash
mkdir -p /opt/cloudera/parcels;
chown cloudera-scm:cloudera-scm /opt/cloudera/parcels
```
配置agent的server指向
```bash
vi /opt/cloudera-manager/cm-5.16.2/etc/cloudera-scm-agent/config.ini
#将server_host修改为cloudera manager server的主机名,对于本示例而言，也就是server主机。
```
==部署CDH离线安装包==
```bash
mkdir -p /opt/cloudera/parcel-repo;
chown cloudera-scm:cloudera-scm /opt/cloudera/parcel-repo;
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020071711054348.png)
> 将CDH相关文件上传至此目录下

上面由于使用外部mysql，此时，需要进行指定：
```bash
/opt/cloudera-manager/cm-5.16.2/share/cmf/schema/scm_prepare_database.sh mysql scm -hlocalhost -uroot -p1qaz@WSX --scm-host localhost scm scm 

#格式：scm_prepare_database.sh 数据库类型、要创建数据库名称、数据库服务器地址、具有创建权限的数据库用户名、具有创建权限的数据库密码、cm server服务器地址、访问新创建数据的用户名、访问新创建数据的密码
```
详情请参考：[https://www.cnblogs.com/xiqing/p/9645724.html](https://www.cnblogs.com/xiqing/p/9645724.html)

启动Cloudera Manager Server
```bash
/opt/cloudera-manager/cm-5.16.2/etc/init.d/cloudera-scm-server start
```
启动Cloudera Manager Agent
```bash
/opt/cloudera-manager/cm-5.16.2/etc/init.d/cloudera-scm-agent start
```
==在本地数据创建以下数据库==，用户后续进去安装
```sql
CREATE DATABASE amon DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
GRANT ALL ON amon.* TO 'amon'@'%' IDENTIFIED BY 'root';
CREATE DATABASE rman DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
GRANT ALL ON rman.* TO 'rman'@'%' IDENTIFIED BY 'root';
CREATE DATABASE hue DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
GRANT ALL ON hue.* TO 'hue'@'%' IDENTIFIED BY 'root';
CREATE DATABASE metastore DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
GRANT ALL ON metastore.* TO 'hive'@'%' IDENTIFIED BY 'root';
CREATE DATABASE sentry DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
GRANT ALL ON sentry.* TO 'sentry'@'%' IDENTIFIED BY 'root';
CREATE DATABASE nav DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
GRANT ALL ON nav.* TO 'nav'@'%' IDENTIFIED BY 'root';
CREATE DATABASE navms DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
GRANT ALL ON navms.* TO 'navms'@'%' IDENTIFIED BY 'root';
CREATE DATABASE oozie DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
GRANT ALL ON oozie.* TO 'oozie'@'%' IDENTIFIED BY 'root';
```
# Cloudera Manager Agent的安装
在除了server服务器外的其他的服务器都要执行以下步骤进行对agent的部署
创建安装目录并解压安装介质
```bash
mkdir /opt/cloudera-manager
tar xzf cloudera-manager*.tar.gz -C /opt/cloudera-manager
```
安装数据库驱动
```bash
mkdir -p /usr/share/java
cp mysql-connector-java-5.1.44-bin.jar /usr/share/java/mysql-connector-java.jar
```
创建系统用户cloudera-scm
```bash
useradd --system --home=/opt/cloudera-manager/cm-5.16.2/run/cloudera-scm-server --no-create-home --shell=/bin/false --comment "Cloudera SCM User" cloudera-scm
```
创建server存储目录
```bash
mkdir /var/lib/cloudera-scm-server
chown cloudera-scm:cloudera-scm /var/lib/cloudera-scm-server
```
创建hadoop安装包存储目录
```bash
mkdir -p /opt/cloudera/parcels;
chown cloudera-scm:cloudera-scm /opt/cloudera/parcels
```
配置agent的server指向
```bash
vi /opt/cloudera-manager/cm-5.16.2/etc/cloudera-scm-agent/config.ini
#将server_host修改为cloudera manager server的主机名,对于本示例而言，也就是server主机。
```
启动Cloudera Manager Agent
```bash
/opt/cloudera-manager/cm-5.16.2/etc/init.d/cloudera-scm-agent start
```
浏览器访问ip:7180 

用户名：admin 密码：admin

到此为止，cloudera manager就安装完成。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200717113133846.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N0d3kyOTEzMTQ=,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200717113157979.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N0d3kyOTEzMTQ=,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200717113226777.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N0d3kyOTEzMTQ=,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200717113309296.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N0d3kyOTEzMTQ=,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200717113406440.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N0d3kyOTEzMTQ=,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200717113441300.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200717113504384.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200717113522957.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N0d3kyOTEzMTQ=,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200717113622302.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N0d3kyOTEzMTQ=,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200717113700102.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N0d3kyOTEzMTQ=,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200717114306671.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N0d3kyOTEzMTQ=,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200717113746816.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N0d3kyOTEzMTQ=,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200717113809346.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N0d3kyOTEzMTQ=,size_16,color_FFFFFF,t_70)

# 问题解决

一、启动服务报错 pstree: 未找到命令
/opt/cm-5.16.2/etc/init.d/cloudera-scm-agent:行109: pstree: 未找到命令
/opt/cm-5.16.2/etc/init.d/cloudera-scm-server:行109: pstree: 未找到命令
原因：缺少psmisc工具
解决方法：`yum install -y psmisc`

二、HDFS namenode或datanode无法启动
删除全部节点的dfs目录：`rm -rf dfs`
在主节点以ROOT身份进行命名节点格式化：`hadoop namenode -format`
之后对dfs目录进行重新赋权：`chown hdfs:hadoop -R /dfs/`

三、hive客户端的hdfs权限认证问题

```bash
Permission denied: user=user, access=READ_EXECUTE, inode="/user/root":root:supergroup:drwx------
```
```bash
hadoop fs -chmod -R 777 /user
```
```bash
vi /etc/profile
```
添加：export HADOOP_USER_NAME=hdfs（hdfs为最高权限）
```bash
source /etc/profile
```
四、impala故障

```bash
select count(*) from impala_100yi;
Query: select count(*) from impala_100yi
Query submitted at: 2019-02-14 14:07:33 (Coordinator: http://cdh004:25000)
Query progress can be monitored at: http://cdh004:25000/query_plan?query_id=5248ba412c4dcffa:306f374700000000
WARNINGS: TransmitData() to 172.15.106.223:27000 failed: Invalid argument: Client connection negotiation failed: client connection to 172.15.106.223:27000: unable to find SASL plugin: PLAIN
```
问题处理
```bash
yum install gcc python-devel cyrus-sasl* -y 
```
然后重启集群的agent和集群服务

五、变更cloudera-scm-server地址

关闭全部cloudera-scm-agent节点

```bash
/opt/cloudera-manager/cm-5.16.2/etc/init.d/cloudera-scm-agent stop
```
关闭原server节点
```bash
/opt/cloudera-manager/cm-5.16.2/etc/init.d/cloudera-scm-server stop
```
更变全部节点的server指向

```bash
vi /opt/cloudera-manager/cm-5.16.2/etc/cloudera-scm-agent/config.ini
#将server_host修改为cloudera manager server的主机名,对于本示例而言，也就是server主机
```

更改数据地址
```bash
cd /opt/cloudera-manager/cm-5.16.2/etc/cloudera-scm-server
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020080417544887.png)
```bash
# Auto-generated by scm_prepare_database.sh on 2020年 07月 14日 星期二 18:07:20 CST
#
# For information describing how to configure the Cloudera Manager Server
# to connect to databases, see the "Cloudera Manager Installation Guide."
#
com.cloudera.cmf.db.type=mysql
com.cloudera.cmf.db.host=S0
com.cloudera.cmf.db.name=scm
com.cloudera.cmf.db.user=root
com.cloudera.cmf.db.setupType=EXTERNAL
com.cloudera.cmf.db.password=1qaz@wsx
```
在变更后的server节点
```bash
/opt/cloudera-manager/cm-5.16.2/etc/init.d/cloudera-scm-server restart
/opt/cloudera-manager/cm-5.16.2/etc/init.d/cloudera-scm-agent restart
```

启动全部cloudera-scm-agent节点

```bash
/opt/cloudera-manager/cm-5.16.2/etc/init.d/cloudera-scm-agent start
```
