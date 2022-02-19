---

layout: post
title: "Redis Cluster Specification"
category: "system-design"
author: "kkzhang"
---

# 1-Main Properties and Rationales of the Design

## 1-1 Redis Cluster Goals

Redis Cluster 是 Redis 的分布式实现，按照优先级有以下目标：

- 高性能（high performance）
- 支持线性扩展（linear scalability）至 1000 个节点
- 没有代理（proxy）；集群节点间通过异步复制数据；不支持数据合并操作
- 可接受的写入安全：在网络分区情况下，系统尽可能保存访问到多数派分区的 Client 的写操作。不过，存在一个小的时间窗口，在此期间的写入操作可能会丢失（*failover 前的写入可能会在 failover 过程中丢失*）。如果 Client 的写入操作连接到少数派分区，则这个丢失时间窗口会更大。
    
    > 网络分区时，集群节点被划分成多数派分区（majority），少数派分区（minority）
    > 
- 可用性：大部分 master 节点可用，并且对少部分不可用的 master，每一个 master 至少有一个当前可用的 slave。通过使用 *replicas migration* 技术，当前没有slave的master会从当前拥有多个slave的master接受到一个新slave来确保可用性。