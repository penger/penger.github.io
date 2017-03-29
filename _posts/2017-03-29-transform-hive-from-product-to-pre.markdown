---
layout: post
category: "read"
title:  "transform hive from product to pre"
tags: [shell,hive]
---

## 需求
生产集群和测试集群 数据传输
A集群机器为A上面有源hive表(分区表,非分区表) B集群上为空白集群
思路:
>根据提供的库名,表名得到时候在A集群存在hdfs文件
>生成hive表的建表语句保存为文件,SCP到B集群中的机器
>执行B机器的建表语句,如果含有分区信息创建分区表
>通过SCP命令将HDFS文件远程拷贝到B集群中

前提:A集群提前创建一个sudo用户,此用户可以免密码登录到B集群中的机器
     程序运行的临时文件存放于 /tmp 目录下并且提供日志记录

脚本如下:
#### A 机器上
tongbu.sh
```bash
#!/bin/bash
####################################
#	gongpeng
#	2017年1月6日 11:04:15
###################################
APP_PATH=$(cd $(dirname $0) && pwd)
#echo "app path is :" $APP_PATH 
TEMP_PATH=$APP_PATH/tmp
RUNNING_LOG_PATH=$APP_PATH/logs/tongbu/running.log
HISTORY_LOG=$APP_PATH/logs/tongbu/history.log

# delete running log create by last task
rm $RUNNING_LOG_PATH

log(){
	record_time=`date +"%F %T"`
	echo $@
	echo $record_time $@ >>$RUNNING_LOG_PATH
	echo $record_time $@ >>$HISTORY_LOG
}
# recode args 
log $@

db_name=$1
table_name=$2
# default partition table
is_partition=1
lines=1
log "step 1 : check if table is exists (check if path in hdfs exists)"
echo -e "\n\n"
hdfs_dir=`sudo -u hive hdfs dfs -ls /user/hive/warehouse/$db_name.db/$table_name`
check_hdfs=$?
if [[ ! $hdfs_dir =~ "dt=" ]]
then
	is_partition=0
else	
	lines=`echo $hdfs_dir |grep -o 'dt='|wc -l`
	partition_value=${hdfs_dir:0-8}
	echo "partition_value is : ${partition_value}"
fi
echo $hdfs_dir
if [ ! $check_hdfs -eq 0 ]
then
	echo "/user/hive/warehouse/$db_name.db/$table_name  not exist task finished "
	exit
fi
log "step 2 : touch create table file "
simple_file_name=$db_name"_"$table_name".q"
create_file=$TEMP_PATH/$simple_file_name
echo -e "drop table $db_name.$table_name; \n" >$create_file
sudo -u hive hive -e "show create table ${db_name}.${table_name}"  1>>$create_file 2>/dev/null
echo -e ";\n" >>$create_file
if [ $is_partition -eq 1 ]
then
	echo -e "alter table $db_name.$table_name add partition (dt='${partition_value}');" >>$create_file
fi
log "step 3 : copy to /tmp dir at 10.201.48.101"
scp $create_file bl@10.201.48.101:~/app/tmp/
log "step 4 : excute job at 101 to create table "
ssh bl@10.201.48.101  "sh /home/bl/app/create_hive_table.sh $simple_file_name $db_name $table_name $lines $partition_value"
log "step 5 : scp hdfs data"
if [ $lines -gt 1 ]
then
	echo "warning multi partition skip ! "
	sudo -u hive hdfs dfs -cp /user/hive/warehouse/$db_name.db/$table_name/dt=${partition_value}/    hdfs://10.201.48.101:8020/user/hive/warehouse/$db_name.db/$table_name/
else
	sudo -u hive hdfs dfs -cp /user/hive/warehouse/$db_name.db/$table_name/    hdfs://10.201.48.101:8020/user/hive/warehouse/$db_name.db/
fi

log "step 6 : check hive records"
ssh bl@10.201.48.101  "sh /home/bl/app/check_hive_records.sh $db_name $table_name"
log "------------finish-------------- "


```
#### B 机器上

create_hive_table.sh
```bash
#!/bin/bash
echo "$@" > /home/bl/app/args.log
sql_file_path=/home/bl/app/tmp
sql_file_name=$1
db_name=$2
table_name=$3
partitons_number=$4
partitons_value=$5
sql_file=$sql_file_path/$sql_file_name
echo "sql file is : ${sql_file}" 
sudo -u hive hive -f $sql_file  &> /dev/null 
echo "delete pre hdfs path: /user/hive/warehouse/${db_name}.db/${table_name}/"
if [ $partitons_number -eq 1 ]
then
	sudo -u hdfs hdfs dfs -rm -R -skipTrash /user/hive/warehouse/${db_name}.db/${table_name}/ &> /dev/null
else
	sudo -u hdfs hdfs dfs -rm -R -skipTrash /user/hive/warehouse/${db_name}.db/${table_name}/dt=${partition_value}/ &> /dev/null
fi

```

check_hive_records.sh
```bash
#!/bin/bash
db_name=$1
table_name=$2
echo "------------------------------query result from pre hive ----------------------------"
query_result=`sudo -u hive hive -e "select * from ${db_name}.${table_name} limit 1"  2> /dev/null `
echo $query_result
```

待完善,脚本中的 HDFS 访问地址为写死的,由于hdfs-site.xml 中的foundation 并非 nameservice,为了防止namenode切换,应该另写一段程序获取active状态的namenode


```js
function () { return "This code is highlighted as Javascript!"}
```