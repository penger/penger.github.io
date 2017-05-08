---
layout: post
category: "read"
title:  "zeppline的使用"
tags: [zeppline]
## 配置方法:
---

```
1.下载zepplin包
2.放到 hive 用户下(如果用其他启动会在执行spark类型的任务时候无法访问hdfs文件)
3.访问 mysql,oracle 选择jdbc interpreter,将mysql,oracle的jar包放到类加载中
4.访问impala,hive的时候选择jdbc interpreter
	修改 zepplin-daemon.sh 添加cdh的jar到lib中
5.访问spark 需要cp conf/zep*-env.sh 文件添加 spark_home
```
## 项目截图
![install steps ](../img/zeppelin/1.png)