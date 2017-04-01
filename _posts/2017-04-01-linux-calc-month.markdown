---
layout: post
category: "read"
title:  "linux date 时间处理遇到的问题"
tags: [linux]
---

## 场景
有一个脚本需要每天传入上一个月yyyyMM 格式的时间 ,当当天为2017-03-31 的时候发现传入的日期是 201703
### 测试如下
```
date -d "20170331 -1 months " +"%Y%m%d"
20170303
date -d "20170331 -29 days " +"%Y%m%d"          
20170302
date -d "20170331 -28 days " +"%Y%m%d" 
20170303
date -d "20160331 -28 days " +"%Y%m%d"    
20160303
date -d "20180331 -28 days " +"%Y%m%d" 
20180303
date -d "20180731 -28 days " +"%Y%m%d" 
20180703
date -d "20180930 -28 days " +"%Y%m%d"  
20180902
date -d "20160331 -1 months " +"%Y%m%d"     
20160302
date -d "20160430 -31 days " +"%Y%m%d"     
20160330
```
### 结论
```
如果当前月份为n,那么通过<font color='red'>date -d "-1 months" +"%Y%m%d"</font> 处理得到的结果是当年n-1月的天数
例如:闰年三月份减去一个月为29天,非闰年三月份为减去28天

### 处理方法
```
#处理日期bug 如果当前时间为 0331 那么 -1 months  得到的月份还是3
month=`date  +"%Y%m"`
month=$month"01"
month=`date -d "${month} -1 months" +"%Y%m"`
if [ $# -eq 1 ]
	then
	month=$1
fi
```