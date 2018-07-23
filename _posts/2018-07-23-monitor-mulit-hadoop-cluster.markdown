---
layout: post
category: "read"
title:  "采集多个大数据集群的指标，并做历史和现状的展示"
tags: [elasticsearch,bootstrap,webmagic]
---

## 功能说明

使用WebMagic通过HTTP页面和JMX采集集群信息,监控多个集群的Hbase,Haddop指标静态页面展示，通过ElasticSearch和Kibina实现历史信息查询
    1.通过修改WebMagic源码认证模块，采集HTML和JMX信息
    2.提交请求到ElasticSearch 
    3.kibina查询历史信息
    4.提供python3下面的 python -m http.server 8080 服务
    
