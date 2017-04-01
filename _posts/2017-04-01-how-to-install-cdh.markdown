---
layout: post
category: "read"
title:  "CDH 安装"
tags: [cdh]
---

## 需求
CDH 集群安装步骤

310.201.481.1 xG4(__H8
310.201.481.2 bJ8&%NH8

310.201.481.46 rootroot
310.201.481.47 rootroot
310.201.481.48 rootroot


vi /etc/sysctl.conf
vm.swappiness = 0

echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag

每一个slave节点执行
1.启动ntpd
echo "server 310.201.96.9" >/etc/ntp.conf
service ntpd status
chkconfig ntpd on
2.修改hosts
vi /etc/sysconfig/network
NETWORKING=yes
HOSTNAME=s46.bl.bigdata
HOSTNAME=s47.bl.bigdata
HOSTNAME=s48.bl.bigdata

scp /etc/hosts root@310.201.481.2:/etc/hosts



3.免密码登录
ssh-keygen -t rsa

两个namenode 的 id_rsa.pub 文件合并为一个 authorized_keys
scp ~/.ssh/authorized_keys root@310.201.481.2:/root/.ssh/

4.重启机器
reboot

5.关闭防火墙

ssh 310.201.481.48 service iptables stop
ssh 310.201.481.48 chkconfig iptables off
ssh 310.201.481.48 chkconfig --list|grep iptables
iptables        0:off   1:off   2:off   3:off   4:off   5:off   6:off


6.规划

418.1	cm nn fc jn hm
418.2	nn fc jn hm spark-history-server hive-metastore
418.46	zk nm
418.47	zk nm
418.48	zk nm


7.安装cm

	7.1 安装mysql http://www.cloudera.com/documentation/enterprise/5-4-x/topics/cm_ig_mysql.html
		按照文档装mysql
		1.yum install mysql-server (修改root 密码为: 111111)
		2.修改 /etc/my.cnf
		3.yum install mysql-connector-java
	7.2 安装cm
		1.安装szrz :yum install( yum install lrzsz)
		2.配置 yum 源
			rz 文件到 /etc/yum.repos.d/ 文件夹下
		3.安装 CM 组件
			yum install cloudera-manager-daemons
			yum install cloudera-manager-server 
			yum install cloudera-manager-agent  	
		4.http://www.cloudera.com/documentation/enterprise/5-4-x/topics/cm_ig_installing_configuring_dbs.html
		cm的mysql库 ./scm_prepare_database.sh mysql -uroot -p111111  scm scm scm
			1) 默认不输入host
		5.启动
			service cloudera-scm-server start
			service cloduera-scm-agent start
		6.访问:
			http://310.201.481.2:7180      用户名:admin  密码: admin
8.界面安装cdh
	1.搜索主机 :输入IP地址
	2.选择存储库: 
		选择方法: 使用Parcel(建议) 点击 更多选项
			  修改 远程parcel存储URL: 第一行修改为:
			  http://310.201.481.102/test/cdh/
			  其他默认        next
	3.安装 Oracle 的 jdk	          next
	4.启动当用户模式	无操作    next
	...
	见图片
	5. scp /etc/yum.repos.d/cloudera-manager.repo 310.201.481.1:/etc/yum.repos.d/
				

	卸载原有的mysql 安装mysql的驱动


9.修改48.1 和48.2的密码为原密码



注意:出现hive无法建立candy测试的情况,需要将mysql驱动放到 /usr/share/java/ 下
    :需要保证节点能够访问公司网络的yum源
    :安装cm管理节点的时候要将server节点纳入cm管理
    :需要提前搭建 本地的yum源并启动httpd服务
    :需要设置 mysqld 开机启动




数据库相关

Activity Monitor	amon	amon	amon_password
Reports Manager	rman	rman	rman_password
Hive Metastore Server	metastore	hive	hive_password
Sentry Server	sentry	sentry	sentry_password


create database amon DEFAULT CHARACTER SET utf8;
grant all on amon.* TO 'amon'@'%' IDENTIFIED BY 'amon';


create database metastore DEFAULT CHARACTER SET utf8;
grant all on metastore.* TO 'hive'@'%' IDENTIFIED BY 'hive';


create database hue DEFAULT CHARACTER SET utf8;
grant all on hue.* TO 'hue'@'%' IDENTIFIED BY 'hue';


![install steps ](img/cdh_install/1.png)
![install steps ](img/cdh_install/2.png)
![install steps ](img/cdh_install/3.png)
![install steps ](img/cdh_install/4.png)
![install steps ](img/cdh_install/5.png)
![install steps ](img/cdh_install/6.png)
![install steps ](img/cdh_install/7.png)
![install steps ](img/cdh_install/8.png)
![install steps ](img/cdh_install/9.png)
![install steps ](img/cdh_install/finish.png)

