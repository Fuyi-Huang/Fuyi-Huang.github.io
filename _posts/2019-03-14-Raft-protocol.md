---
title: Raft协议总结
tags: 
    - Raft 
    - summary
article_header:
  type: cover
  image:
    src: assets/images/header_images/zelda.jpg
---

## 前言

Raft是一种分布式共识算法，类似的还有Paxos。
这里有一个动画[http://thesecretlivesofdata.com/raft/](http://thesecretlivesofdata.com/raft/)，对于理解Raft协议做得很好。


Raft广泛应用于分布式存储中，例如分布式存储系统tikv, etcd, consul。
在Raft集群中，每一个服务器可能是三种身份中的一个：leader, follower, candidate。
leader与客户端交互，通过日志复制的方式将客户端的信息同步信息到他的follower服务器里。follower服务器则负责同步来自leader服务器的日志来保持与leader服务器信息同步。candidate是在follower状态下失去了leader服务器的连接之后，状态变为candidate，发起一次新的leader的选举，candidate服务器如果被选举成功就会变成leader服务器。

下面将分两方面总结Raft协议：

  1. leader选举

  2. leader服务器和follower服务器之间的信息同步

### leader选举

Raft里面有两个超时时间，一个是election timeout(选举超时)，另一个是heartbeat timeout(心跳超时)。

首先是heartbeat timeout(心跳超时)。leader服务器每当有信息更新都会发送日志到他的follower服务器，没有更新也会发送心跳到follower服务器。如果follower服务器超过heartbeat timeout时间没有收到来自leader服务器的信息则会认为leader服务器挂了。那follower服务器便会变更为candidate服务器，发起一轮投票来选举新的leader服务器。

election timeout是candidate服务器发起投票的时间间隔，它一般是150ms到300ms的随机数。假设现在只有一个服务器变成了candidate服务器，candidate向其他的follower发起投票，如果follower在这一轮选举中没有投票过，则会给该candidate投票。如果candidate服务器得到大多数的票数**N/ 2 + 1**(它自己也算一票，所以需要得到其他**N/2**服务器的票数)，那该candidate服务器便被选举为新的leader。

如果有多个follower成功candidate发起投票，选举过程是一样的，一个follower在一轮投票中只能投一票，candidate需要得到至少**N/ 2 + 1**的票数才能成为leader。如果有多个candidate，那第一轮投票大概是会失败的。Raft协议里规定election timeout是150ms到300ms之间的随机数。所以每个candidate的election timeout都是不同的，election timeout较小的candidate有较大的概率会成为新的leader。



### 日志复制

当客户端发起一个更新请求时，leader现在它自己的服务器更新，如果成功了，则在下一次心跳的时候发送日志到follower服务器，如果大多数follower服务器append日志成功就会返回成功给leader。如果leader收到大多数**N / 2** follower返回成功，leader则返回成功给客户端，并commit该更新，并在下一次心跳发送commit信息。

未完待续...