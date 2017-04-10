---
layout: post
category: "read"
title:  "根据模板shell生成shell(执行hive,删除oracle,sqoop导出)"
tags: [linux,shell,hive,sqoop]
---

## 场景
各种不同格式的html(主要是跨行跨列),需要按照原有的格式导出为Excel
## 思路
1.拷贝template文件为 传入参数的命名的shell脚本
2.执行大小写转换,sed替换,oracle删除
3.生成shell脚本并且mv到指定目录,赋权
4.提供web接口调用

### shell_creator.sh 
```
#!/bin/bash
########################################################################################################
#
#	2016-7-5 15:52:55
#       update: 2016-7-27 16:32:26
#	通过提供参数,获取常规的etl脚本
#	有四个参数: 表名 oracle是否是增量 hive导入的时候时候是分区   oracle的删除字段(如果oracle是增量)
#
########################################################################################################
APP_PATH=/home/bl/app/new_template
LOG_PATH=$APP_PATH/log/shell_creator.log
TARGET_PATH=/home/bl/app/new_template/etl_shell
TEMPLATE=$APP_PATH/shell.template
time=`date +"%Y-%m-%d %H:%M:%S"`
echo $time $@ >> $LOG_PATH
if [ ! $# -eq 5 ] 
then
	echo "should have five args [tablename is_inc is_partition delete_colomn is_month ]"
	echo "eg.   m_member_points_d_offline 1 1 mmm 0"
	exit
fi
table_name=$1
upper_table_name=$(echo $table_name | tr '[a-z]' '[A-Z]')
is_inc=$2
is_partation=$3
delete_column=$4
is_month=$5

target_file=$TARGET_PATH"/"$table_name".sh"
#如果存在,那么删除原有文件
if [ -f $target_file ]
then
	rm $target_file
fi

cp $TEMPLATE $TARGET_PATH/$1".sh"

#替换表名称
sed -i "s/replaceable_table/`echo ${table_name}`/g" ${target_file}
sed -i "s/REPLACEABLE_TABLE/`echo ${upper_table_name}`/g" ${target_file} 

#替换 delete_content  和 partition_content 

to_delete_content=""
to_partition_content=""


#如果是增量,那么要求增量删除

if [ $is_inc -eq 0 ] 
then
	to_delete_content="clearoracletb.sh $table_name"
elif [ $is_inc -eq 1 ]
then
	to_delete_content="customize_clearoracletb.sh $table_name $delete_column \$\{task_date\}"
else
	#注释和oracle相关的操作
	sed -i "s/oper_oracle_replacement /#/g" ${target_file}
fi

#放开oracle相关的注释
sed -i "s/oper_oracle_replacement //g" ${target_file}
#oracle删除逻辑替换
sed -i "s/delete_content/`echo ${to_delete_content}`/g" ${target_file}


#分区删除处理

if [ $is_partation -eq 1 ] 
then
	to_partition_content="dt\=\$\{task_date\}\/"
fi

#分区导入替换
sed -i "s/partition_content/`echo ${to_partition_content}`/g" ${target_file}

#对月任务进行单独的替换
if [ $is_month -eq 1 ]
then
	sed -i "s/task_date/task_month/g" ${target_file}
	sed -i "s/SDATE/SMONTH/g" ${target_file}
	sed -i "s/calc_day_replacement/date -d \"-1 months\" +\"%Y%m\"/" ${target_file}
else
	sed -i "s/calc_day_replacement/date -d \"-1 days\" +\"%Y%m%d\"/" ${target_file}
fi


chmod +x $target_file

url="http://10.201.48.49:8080/z_bigdata/email?op=task&subject=createShell&content=$table_name"

curl --data-urlencode -s `echo $url`

cp ${target_file} /home/bl/ETL/
echo "--------------------------------------脚本如下---------------------------------------------"
echo "<font color='green'>"
cat ${target_file}
echo "</font>"
```
### shell.template
```
#!/bin/bash 
task_date=`calc_day_replacement` 
if [ $# -eq 1 ]
	then
	task_date=$1
fi

if [ `expr length ${task_date}` -eq 8 ]
then
	x_date=`date -d "${task_date}" +"%Y-%m-%d"`
else
	x_date="2099-99-99"
fi

echo $task_date

exit_code=0

SQL_PATH="/home/bl/ETL/sql"

LOG_PATH="/home/bl/ETL/log/$task_date"
LOG_FILE="$LOG_PATH/replaceable_table.log"
LOG_ERROR="$LOG_PATH/error.log"

mkdir -p $LOG_PATH

check_if_not_zero_exit(){
	$@ >> $LOG_FILE 2>&1
	exit_code=$?
	time=`date +"%Y-%m-%d %H:%M:%S"`
	echo  $time $@ &>> $LOG_FILE 2>&1
	echo  " ----------------------$exit_code" &>> $LOG_FILE 2>&1
	if [ ! $exit_code -eq 0 ]
	then
		echo  " run failed !!! " &>> $LOG_FILE 2>&1
		echo $time $@ "error" &>> $LOG_ERROR 2>&1
		exit $exit_code
	fi
}

#检查hdfs文件

#执行hive语句
check_if_not_zero_exit  hive -hivevar SDATE=${task_date} -hivevar TX_DATE=${x_date} -hivevar TXDATE=${task_date} -f $SQL_PATH/replaceable_table.q


#删除oracle中的数据 增量删除 inc_*   
oper_oracle_replacement check_if_not_zero_exit  ssh oracle@10.201.000.111 "/home/oracle/bin/delete_content "
#数据从hive导入到oracle
oper_oracle_replacement check_if_not_zero_exit  sqoop export --connect jdbc:oracle:thin:@10.201.000.111:1521/dbservice --username un --password pw --table IDMDATA.REPLACEABLE_TABLE  --export-dir /user/hive/warehouse/idmdata.db/replaceable_table/partition_content  --fields-terminated-by '\001' --input-null-string '\\N' --input-null-non-string '\\N' --m 1
```
### customize_clearoracletb.sh
```
# !/bin/sh
##
# DO NOT DELETE THIS FILE, SINCE IT WAS USED BY SOME OOZIE ACTION!
##
export ORACLE_HOME=/u01/app/oracle/product/11.2.0/db_1
export PATH=$PATH:$ORACLE_HOME/bin
cdt=`echo \'$3\' `
#echo "------------------"
#echo $1 $2 $3 $cdt
#echo "------------------"
where_content="$2=$cdt"
echo $where_content

sqlplus idmdata/bigdata915@hdporcl <<EOF
delete from idmdata.$1 where $where_content;
commit;
EOF

```
## 示例
![install steps ](../img/shell_create/1.png)
