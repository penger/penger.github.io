---
layout: post
category: "read"
title:  "使用presto 连接多个数据库"
tags: [python]
---

## 场景
使用presto客户端连接oracle和mysql并进行join
### 一些步骤 
```
安装目录为 /opt/presto168
启动脚本为 bin/launcher start  //run 为可以看到日志,如果 start看不到日志
停止脚本为 bin/launcher stop
日志目录为 /var/presto_data/var/log/
编辑配置文件 /opt/presto168/etc/catalog/querymysql.propertities

oracle 遇到的坑
1.jdk需要1,8
2.oracle jar包 https://github.com/marcelopaesrech/presto-oracle
3.最后引入idea编译

./presto --schema oracle --catalog testoracle
```
## 图片

![install steps ](../img/presto_install/1.png)
![install steps ](../img/presto_install/2.png)
![install steps ](../img/presto_install/3.png)
