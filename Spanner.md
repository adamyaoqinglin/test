---
title: Spanner
mathjax: true
---

[Google Spanner](http://gitlab.turbolinux.com.cn/shuail/newsql-study/blob/master/paper/Spanner-%20Google%E2%80%99s%20Globally-Distributed%20Database.pdf)
[中文版](http://gitlab.turbolinux.com.cn/shuail/newsql-study/blob/master/paper/Google%20Spanner%E4%B8%AD%E6%96%87%E7%89%88.pdf)

## 摘要

  Spanner可扩展，多版本，全球部署，同步复制数据库。支持全球部署和外部一致的分布式事务。论文描述了Spanner的架构，特性，不同设计策略下的原理，一个暴露了时钟不确定性的新时间API。

## 介绍

  在最高抽象层面，Spanner就是一个数据库，把数据分片存储在许多Paxos状态机上，这些机器位于遍布全球的数据中心。客户端会自动在副本之间进行失败恢复。随着数据的变化和服务器的变化，Spanner会自动把数据进行重新分片，有效应对负载变化和处理失败。
  Spanner演化成为一个具有时间属性的多版本数据库。Spanner特性：1.在数据的副本配置方面，应用可以在一个很细的力度上进行动态控制。应用可以详细规定，哪些数据中心包含哪些数据，数据距离用户有多远（控制用户读取数据的延迟），不同数据副本之间距离有多远（控制写操作的延迟），以及需要维护多少个副本（控制可用性和读操作性能）。数据也可以被动态和透明地在数据中心之间进行移动，从而平衡不同数据中心内资源的使用。2.Spanner有两个很难在一个分布式数据库上实现的特性：提供读写操作的外部一致性；在一个时间戳下面的跨数据库全球一致性读操作。

## 实现

  一个Spanner部署称为一个universe。Spanner被组织成多个zone的集合，zone是管理部署的基本单元。当新的数据中心加入或者老的数据中心被关闭时，zone合一被加入到一个运行的系统中或从中移除。zone也是物理隔离的单元，在一个数据中心中，可能有一个或者多个zone，例如，属于不同应用的数据可能必须被分区存储到同一个数据中心的不同服务器集合中。
![Figure 1: Spanner server organization](Spanner/org.png)
  一个zone包括一个zonemaster，和一百至几千个spanserver。zonemaster把数据分配给spanserver，spanserver把数据提供给客户端。客户端使用每个zone上面的location proxy来定位可以为自己提供数据的spanserver。universe master是一个控制台，显示了关于zone的各种状态信息，可以用于相互之间的调试。Placement driver会周期性地与spanserver进行交互，来发现那些需要被转移的数据，或者市委了满足新的副本约束条件，或者是为了进行负载均衡。

### Spanserver Software Stack

  每个spanserver负载管理100-1000个成为tablet的数据结构的实例。一个tablet就类似于BigTable中的tablet，实现了下面的映射：(key:string, timestamp:int64)->string
![Figure 2: Spanserver software stack](Spanner/sss.png)
  每个spanserver会在每个tablet上实现一个单个的Paxos状态机。当前的Spanner实现中，会对每个Paxos写操作进行两次记录：一次写入到tablet日之中，一次写入到Paxos日之中。在Paxos实现上采用了管道化的方式，可以在存在广域网延迟时改进Spanner的吞吐量，但是，Paxos会把写操作按照顺序的方式执行。
  Paxos 状态机是用来实现一系列被一致性复制的映射。每个副本的键值映射状态,都会被保存到相应的 tablet 中。写操作必须在领导者上初始化 Paxos 协议,读操作可以直接从底层的任何副本的 tablet 中访问状态信息,只要这个副本足够新。副本的集合被称为一个 Paxos group。
  对于每个是领导者的副本而言,每个 spanserver 会实现一个锁表来实现并发控制。这个锁表包含了两阶段锁机制的状态:它把键的值域映射到锁状态上面。对于每个扮演领导者角色的副本,每个 spanserver 也会实施一个事务管理器来支持分布式事务。这个事务管理器被用来实现一个 participant leader,该组内的其他副本则是作为participant slaves。如果一个事务只包含一个 Paxos 组(对于许多事务而言都是如此),它就可以绕过事务管理器,因为锁表和 Paxos 二者一起可以保证事务性。如果一个事务包含了多于一个 Paxos 组,那些组的领导者之间会彼此协调合作完成两阶段提交。其中一个参与者组,会被选为协调者,该组的 participant leader 被称为 coordinator leader,该组的 participant slaves被称为 coordinator slaves。每个事务管理器的状态,会被保存到底层的 Paxos 组。

### Directories and Placement

  在一系列键值映射的上层，Spanner实现支持一个被称为目录的桶抽象，也就是包含公共前缀的连续键的集合。对目录的支持可以允许应用通过合适的键来控制数据的位置。一个目录是数据放置的基本单元。属于一个目录的所有数据，都具有相同的副本配置。当数据在不同的Paxos组之间移动是，会一个目录一个目录地转移。Spanner可能会移动一个目录从而减轻一个 Paxos 组的负担,也可能会把那些被频繁地一起访问的目录都放置到同一个组中,或者会把一个目录转移到距离访问者更近的地方。当客户端操作正在进行时,也可以进行目录的转移。我们可以预期在几秒内转移 50MB 的目录。
![Figure 3: Directories are the unit of data movement between Paxos groups](Spanner/dir.png)
  一个目录变得太大时，Spanner会把它分片存储，每个分片可能会被保存到不同的Paxos组上。

### 数据模型

  Spanner 会把下面的数据特性集合暴露给应用:基于模式化的半关系表的数据模型,查询语言和通用事务。需要支持模式化的半关系表是由Megastore的普及来支持的
  应用的数据模型是架构在被目录桶装的键值映射层之上。一个应用会在一个universe中创建一个或者多个数据库。每个数据库可以包含无限数量的模式化的表。每个表都和关系数据库表类似,具备行、列和版本值。
  Spanner 的数据模型不是纯粹关系型的,它的行必须有名称。更准确地说,每个表都需要有包含一个或多个主键列的排序集合。这种需求,让 Spanner 看起来仍然有点像键值存储:主键形成了一个行的名称,每个表都定义了从主键列到非主键列的映射。当一个行存在时,必须要求已经给行的一些键定义了一些值(即使是 NULL)。采用这种结构是很有用的,因为这可以让应用通过选择键来控制数据的局部性。
![Figure 4: Example](Spanner/data.png)
  图 4 包含了一个 Spanner 模式的实例,它是以每个用户和每个相册为基础存储图片元数据。这个模式语言和 Megastore 的类似,同时增加了额外的要求,即每个 Spanner 数据库必须被客户端分割成一个或多个表的层次结构(hierarchy)。客户端应用会使用 INTERLEAVE IN 语句在数据库模式中声明这个层次结构。这个层次结构上面的表,是一个目录表。目录表中的每行都具有键 K,和子孙表中的所有以 K 开始(以字典顺序排序)的行一起,构成了一个目录。ON DELETE CASCADE 意味着,如果删除目录中的一个行,也会级联删除所有相关的子孙行。这个图也解释了这个实例数据库的交织层次(interleaved layout),例如 Albums(2,1)代表了来自 Albums 表的、对应于 user_id=2 和 album_id=1 的行。这种表的交织层次形成目录,是非常重要的,因为它允许客户端来描述存在于多个表之间的位置关系,这对于一个分片的分布式数据库的性能而言是很重要的。没有它的话,Spanner 就无法知道最重要的位置关系。

## TrueTime

aa $$ t_abs(e_2^star) $$
![Figure 5: TrueTime](Spanner/time.png)

### 时间戳管理

### 并发控制

#### Paxos领导者租约

#### 为读写事务分配时间戳

#### 在某个时间戳下的读操作

#### 为只读事务分配时间戳

### 细节

#### 读写事务

#### 只读事务

#### 模式变更事务

#### 优化

## 实验分析

### 微测试基准

### 可用性

### TrueTime

### F1

## 相关工作

## 未来工作

## 总结

  Spanner 对来自两个研究群体的概念进行了结合和扩充:一个是数据库研究群体,包括熟悉易用的半关系接口,事务和基于 SQL 的查询语言;另一个是系统研究群体,包括可扩展性,自动分区,容错,一致性复制,外部一致性和大范围分布。
