---
layout: post
category: "zookeeper"
title:  "zookeeper 的运维技巧"
tags: [zeppline]
## 配置方法:
---
Zookeeper是一个分布式协调框架，有不错的性能，也经过许多公司的验证，所以在很多场景都有使用。大家一般用Zookeeper来实现服务发现(类似DNS)，配置管理，分布式锁，leader选举等。在这些场景中，Zookeeper成为了一个被依赖的核心组件，Zookeeper的稳定性是需要特别关注的。

去哪儿网也在很多场景依赖Zookeeper，所以我们也一直在摸索怎么更好的运维稳定的Zookeeper集群。在过去的几年我们也踩过一些坑，也因为Zookeeper导致了故障。现在将我们运维Zookeeper集群的一些经验分享，也欢迎大家提供更好的建议。



那么在打算运维一套Zookeeper集群之前，我们先了解一些Zookeeper的基本原理。

集群里分三种角色: Leader, Follower和Observer。Leader和Follower参与投票，Observer只会『听』投票的结果，不参与投票。

投票集群里的节点数要求是奇数

一个集群容忍挂掉的节点数的等式为 N = 2F + 1，N为投票集群节点数，F为能同时容忍失败节点数。比如一个三节点集群，可以挂掉一个节点，5节点集群可以挂掉两个...

一个写操作需要半数以上的节点ack，所以集群节点数越多，整个集群可以抗挂点的节点数越多(越可靠)，但是吞吐量越差。

Zookeeper里所有节点以及节点的数据都会放在内存里，形成一棵树的数据结构。并且定时的dump snapshot到磁盘。

Zookeeper的Client与Zookeeper之间维持的是长连接，并且保持心跳，Client会与Zookeeper之间协商出一个Session超时时间出来(其实就是Zookeeper Server里配置了最小值，最大值，如果client的值在这两个值之间则采用client的，小于最小值就是最小值，大于最大值就用最大值)，如果在Session超时时间内没有收到心跳，则该Session过期。

Client可以watch Zookeeper那个树形数据结构里的某个节点或数据，当有变化的时候会得到通知。



有了这些了解后，那么我们心里其实有底了。



1. 最小生产集群

要确保Zookeeper能够稳定运行，那么就需要确保投票能够正常进行，最好不要挂一个节点整个就不work了，所以我们一般要求最少5个节点部署。



2. 网络

除了节点外，我们还要看不能一台物理机器，一个机柜或一个交换机挂掉然后影响了整个集群，所以节点的网络结构也要考虑。这个可能就比很多应用服务器的要求更加严格。



3. 分Group，保护核心Group

要确保Zookeeper整个集群可靠运行，就是要确保投票集群可靠。那在我们这里，将一个Zookeeper集群划分为多个小的Group，我们称Leader+Follower为核心Group，核心Group我们一般是不向外提供服务的，然后我们会根据不同的业务再加一些Observer，比如一个Zookeeper集群为服务发现，消息，定时任务三个不同的组件提供服务，那么我们为建立三个Observer Group，分别给这三个组件使用，而Client只会连接分配给它的Observer Group，不去连接核心Group。这样核心Group就不会给Client提供长连接服务，也不负责长连接的心跳，这大大的减轻了核心Group的压力，因为在实际环境中，一个Zookeeper集群要为上万台机器提供服务，维持长连接和心跳还是要消耗一定的资源的。因为Observer是不参与投票的所以加Observer并不会降低整体的吞吐量，而且Observer挂掉不会影响整个集群的健康。

但是这里要注意的是，分Observer Group只能解决部分问题，因为毕竟所有的写入还是要交给核心Group来处理的，所以对于写入量特别大的应用来说，还是需要进行集群上的隔离，比如Storm和Kafka就对Zookeeper压力比较大，你就不能将其与服务发现的集群放在一起。



4. 内存

因为Zookeeper将所有数据都放在内存里，所以对JVM以及机器的内存也要预先计划，如果出现Swap那将严重的影响Zookeeper集群的性能，所以我一般不怎么推荐将Zookeeper用作通用的配置管理服务。因为一般配置数据还是挺大的，这些全部放在内存里不太可控。



5. 日志清理

因为Zookeeper要频繁的写txlog(Zookeeper写的一种顺序日志)以及定期dump内存snapshot到磁盘，这样磁盘占用就越来越大，所以Zookeeper提供了清理这些文件的机制，但是这种机制并不太合理，它只能设置间隔多久清理，而不能设置具体的时间段。那么就有可能碰到高峰期间清理，所以建议将其关闭:autopurge.purgeInterval=0。然后使用crontab等机制，在业务低谷的时候清理。



6. 日志，jvm配置

从官网直接下载的包如果直接启动运行是很糟糕的，这个包默认的配置日志是不会轮转的，而且是直接输出到终端。我们最开始并不了解这点，然后运行一段时间后发现生成一个庞大的zookeeper.out的日志文件。除此之外，这个默认配置还没有设置任何jvm相关的参数(所以堆大小是个默认值)，这也是不可取的。那么有的同学说那我去修改Zookeeper的启动脚本吧。最好不要这样做，Zookeeper会加载conf文件夹下一个名为zookeeper-env.sh的脚本，所以你可以将一些定制化的配置写在这里，而不是直接去修改Zookeeper自带的脚本。

#!/usr/bin/env bash



JAVA_HOME= #java home

ZOO_LOG_DIR= #日志文件放置的路径

ZOO_LOG4J_PROP="INFO,ROLLINGFILE" #设置日志轮转

JVMFLAGS="jvm的一些设置，比如堆大小，开gc log等"



7. 地址

在实际环境中，我们可能因为各种原因比如机器过保，硬件故障等需要迁移Zookeeper集群，所以Zookeeper的地址是一个很头痛的事情。这个地址有两方面，第一个是提供给Client的地址，建议这个地址通过配置的方式下发，不要让使用方直接使用，这一点我们前期做的不好。另外一个是集群配置里，集群之间需要通讯，也需要地址。我们的处理方式是设置hosts:

192.168.1.20 zk1

192.168.1.21 zk2

192.168.1.22 zk3

在配置里:

server.1=zk1:2081:3801

server.2=zk2:2801:3801

server.3=zk3:2801:3801

这样在需要迁移的时候，我们停老的节点，起新的节点只需要修改hosts映射就可以了。比如现在server.3需要迁移，那我们在hosts里将zk3映射到新的ip地址。但是对于java有一个问题是，java默认会永久缓存DNS cache，即使你将zk3映射到别的ip，如果并不重启server.1, server.2，它是不会解析到新的ip的，这个需要修改$JAVA_HOME/jre/lib/security/java.security文件里的networkaddress.cache.ttl=60，将其修改为一个比较小的数。

对于这个迁移的问题，我们还遇到一个比较尴尬的情况，在最后的坑里会有提及。



8. 日志位置

Zookeeper主要产生三种IO: txlog(每个写操作，包括新Session都会记录一条log)，Snapshot以及运行的应用日志。一般建议将这三个IO分散到三个不同的盘上。不过我们倒是一直没有这么实验过，我们的Zookeeper也是运行在虚拟机(一般认为虚拟机IO较差)上。



9. 监控

我们对Zookeeper做了这样一些监控:

a. 是否可写。 就是一个定时任务定时的去创建节点，删节点等操作。这里要注意的是Zookeeper是一个集群，我们监控的时候我还是希望对单个节点做监控，所以这些操作的时候不要连接整个集群，而是直接去连接单个节点。

b. 监控watcher数和连接数 特别是这两个数据有较大波动的时候，可以发现使用方是否有误用的情况

c. 网络流量以及client ip 这个会记录到监控系统里，这样很快能发现『害群之马』



10. 一些使用建议

a. 不要强依赖Zookeeper，也就是Zookeeper出现问题业务已然可以正常运行。Zookeeper是一个分布式的协调框架，主要做的事情就是分布式环境的一致性。这是一个非常苛刻的事情，所以它的稳定性受很多方面的影响。比如我们常常使用Zookeeper做服务发现，那么服务发现其实是不需要严格的一致性的，我们可以缓存server list，当Zookeeper出现问题的时候已然可以正常工作，在这方面etcd要做的更好一些，Zookeeper如果出现分区，少数派是不能提供任何服务的，读都不可以，而etcd的少数派仍然可以提供读服务，这在服务发现的时候还是不错的。



b. 不要将很多东西塞到Zookeeper里，这个上面已经提到过。



c. 不要使用Zookeeper做细粒度锁，比如很多业务在订单这个粒度上使用Zookeeper做分布式锁，这会频繁的和Zookeeper交互，对Zookeeper压力较大，而且一旦出现问题影响面广。但是可以使用粗粒度的锁(其实leader选举也是一种锁)。



d. 不建议做通用配置的第二个理由是，通用配置要提供给特别多特别多系统使用，而且一些公共配置甚至所有系统都会使用，一旦这样的配置发生变更，Zookeeper会广播给所有的watcher，然后所有Client都来拉取，瞬间造成非常大的网络流量，引起所谓的『惊群』。而自己实现通用配置系统的时候，一般会对这种配置采取排队或分批通知的方式。



11. 一些坑

a. zookeeper client 3.4.5 ping时间间隔算法有问题，在遇到网络抖动等原因导致一次ping失败后会断开连接。3.4.6解决了这个问题 Bug1751。



b. zookeeper client如果因为网络抖动断开了连接，如果后来又重连上了，zookeeper client会自动的将之前订阅的watcher等又全部订阅一遍，而Zookeeper默认对单个数据包的大小有个1M的限制，这往往就会超限，最后导致一直不断地的重试。这个问题在较新的版本得到了修复。Bug706



c. 抛出UnresolvedAddressException异常导致Zookeeper选举线程退出，整个集群无法再选举，处于崩溃的边缘。这个问题是，某次OPS迁移机器，将老的机器回收了，所以老的机器的IP和机器名不复存在，最后抛出UnresolvedAddressException这个异常，而Zookeeper的选举线程(QuorumCnxManager类里的Listener)只捕获了IOException，导致该线程退出，该线程一旦退出只要现在的leader出现问题，需要重新选举，则不会选出新的leader来，整个集群就会崩溃。Bug2319(PS，这个bug是我report的)



原文链接是那次处理Zookeeper选举线程退出的处理办法
