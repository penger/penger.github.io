---
layout: post
category: "read"
title:  "spark一行拆多行然后插入到hive中"
tags: [spark,hive]
---

## 需求
### 分隔文件中的电话号码字段,然后将一行拆为多行的代码实现
### 源文件
```
170220003CD|200.0000|2617349500 2617349502~2617349507|2017/02/28 18:55:02|100.0000|80119|商业经营公司(内部使用－发卡主体)|商品|0023769|全渠道线上销售
```
### 目标结果
```
170220003CD|200.0000|2617349500|2017/02/28 18:55:02|100.0000|80119|商业经营公司(内部使用－发卡主体)|商品|0023769|全渠道线上销售
170220003CD|200.0000|2617349502|2017/02/28 18:55:02|100.0000|80119|商业经营公司(内部使用－发卡主体)|商品|0023769|全渠道线上销售
170220003CD|200.0000|2617349503|2017/02/28 18:55:02|100.0000|80119|商业经营公司(内部使用－发卡主体)|商品|0023769|全渠道线上销售
170220003CD|200.0000|2617349504|2017/02/28 18:55:02|100.0000|80119|商业经营公司(内部使用－发卡主体)|商品|0023769|全渠道线上销售
170220003CD|200.0000|2617349505|2017/02/28 18:55:02|100.0000|80119|商业经营公司(内部使用－发卡主体)|商品|0023769|全渠道线上销售
170220003CD|200.0000|2617349506|2017/02/28 18:55:02|100.0000|80119|商业经营公司(内部使用－发卡主体)|商品|0023769|全渠道线上销售
170220003CD|200.0000|2617349507|2017/02/28 18:55:02|100.0000|80119|商业经营公司(内部使用－发卡主体)|商品|0023769|全渠道线上销售
```
### spark 代码为
```
val file = sc.textFile("/tmp/demo2.txt")
file.map { line =>
val array = line.split("\\|")
val tels = array(2)
val telss=tels.split(" ")
println("hello -------------------------------------")
val content = line.replace(tels, "dianhuahaoma")
for (j <- telss) yield {
    if (j.contains("~")) {
        val start = BigInt(j.split("~")(0))
        val end = BigInt(j.split("~")(1))
        for (k <- start to end) yield {
            (k, content)
        }} else {
            Vector((telss(0), content))
        }
    }
}.flatMap(s=>s).flatMap(s=>s).map(s=>s._2.replace("dianhuahaoma",""+s._1).replace("|","\t")).saveAsTextFile("/tmp/abc28")
```
## shell script
```
#!/bin/bash
APP_PATH=$(cd $(dirname $0) && pwd)
FTP_PATH=$APP_PATH/ftp/
file_postfix_1=ConsumeSales_
file_postfix_2=PrepaidSales_

#处理日期bug 如果当前时间为 0331 那么 -1 months  得到的月份还是3
month=`date  +"%Y%m"`
month=$month"01"
month=`date -d "${month} -1 months" +"%Y%m"`
if [ $# -eq 1 ]
	then
	month=$1
fi


file1=${file_postfix_1}${month}.txt
file2=${file_postfix_2}${month}.txt
echo "file is :" $file1
echo "file is :" $file2
echo "ftp path is :"${FTP_PATH}

ftp -n <<- EOF
open 192.168.12.73
user blemall blemall
get ${file1} ${FTP_PATH}${file1}
get ${file2} ${FTP_PATH}${file2}
bye
EOF

################# 转码操作 ##################


iconv -f gbk -t utf8 -o ${FTP_PATH}${file1}.utf8  ${FTP_PATH}${file1}
iconv -f gbk -t utf8 -o ${FTP_PATH}${file2}.utf8  ${FTP_PATH}${file2}

#处理第一个文件
sed -i "s/|/\t/g" ${FTP_PATH}${file1}.utf8
hdfs dfs -rm /user/hive/warehouse/sourcedata.db/s22_afb_consume_sales/${month}
hdfs dfs -put ${FTP_PATH}${file1}.utf8  /user/hive/warehouse/sourcedata.db/s22_afb_consume_sales/${month}

#处理第二个文件
hdfs dfs -mkdir /tmp/ftp/
hdfs dfs -put ${FTP_PATH}${file2}.utf8   /tmp/ftp/
#清空上次运行结果
hdfs dfs -rm -r /tmp/ftp/result
#对数据进行处理
spark-submit --class AfbFileHandler  ${APP_PATH}/encode-1.0-SNAPSHOT.jar /tmp/ftp/PrepaidSales_201702.txt.utf8  /tmp/ftp/result
#删除上次运行的本月的数据
hdfs dfs -rm /user/hive/warehouse/sourcedata.db/s22_afb_prepaid_sales/${month}
hdfs dfs -mv /tmp/ftp/result/part-00000 /user/hive/warehouse/sourcedata.db/s22_afb_prepaid_sales/${month}

```
