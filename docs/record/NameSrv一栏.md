# NameSrv

## what

NameSrv是什么？

![](image/rocketmq_architecture_1.png)

从架构图看NameSrv用于储存Broker的路由数据，同时也被生产者和消费者端端作服务发现，来获取这条Message该发往哪一个broker，从哪一台broker上拉取(接受推送的)数据。

## Why

为什么需要NameSrv？

首先NameSrv是无状态的，意味着NameSrv可以无限扩容。此外，NameSrv的出现使得Broker不再关注生产者端和消费者端的链接方式，当Broker横向扩容时候，只需要告诉老大哥，老大哥自然会把话带给上游卖家(生产者)和下游买家(消费者)。

## How

NameSrv是怎么保存Broker数据的？

NameSrv是怎么同步Broker数据给生产者和消费者的？

带着疑问，我们来一探究竟。

### NameSrv的启动过程

**核心代码 --- org.apache.rocketmq.namesrv.NamesrvStartup**

1. main方法

