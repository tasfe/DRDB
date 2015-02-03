#Motivation and vision
```
###RDB/NOSQL/VITESS
Relational databases (like MySQL) were initially built and optimized for non-web OLTP systems. However, they have still managed to fulfill most of the needs of large scale web applications. Many of their legacy requirements have held them back from being able to optimally meet the needs of today’s applications. This has led to the development of alternate storage solutions (NoSQL) which in some respects have thrown the baby out with the bathwater.

###SCALE MYSQL OUT
With Vitess, we take a different approach. We think that databases like mysql have what it takes to provide an efficient data storage layer. What is missing is the ability to easily scale out and then coordinate many instances of a single logical schema. The way we plan to achieve this is by providing a subset SQL front end with limited guarantees and a loosely coupled distributed system to automate complex management scenarios.

###CONNECTION POOL
Sessionless connections: A mysql connection carries a lot of context. This is usually unnecessary for web apps. Most Vitess connections are sessionless, which makes them lightweight, and this allows us to pool a very small number of connections to mysql.

###事务的原子性
ACID (Atomicity): This feature is an overkill for most web apps. Instead of honoring system-wide atomicity, Vitess gives you atomic guarantees for a given entity addressed by a “key” (like a user id). This allows us to transparently scale the database by splitting on key ranges.

###事务的一致性
ACID (Consistency): Vitess relaxes this rule towards data being eventually consistent. This allows us to use replication to distribute read traffic in situations where the app does not need up-to-date data. You can always request data to be read from the master db if you explicitly need up-to-date data.

###ROW CACHE
Buffer cache vs. row cache: The mysql buffer cache is optimized for range scans over indices and tables, particularly when data is densely packed. Unfortunately, it’s not good for random access tables. Vitess will allow you to designate certain tables as random access. For such cases, it will maintain row based caches and keep them consistent by fielding all DMLs that could potentially affect them.
```

#Vitess feature set
```
###Sharding

每个分片数据库里的所有表都需要含有一个列，这个列是分片键shard key,Vitess将会依据这些分片键的值决定到哪一个分片数据库去查找或修改数据；

所有以分片键作为索引的表合在一起就是一个keyspace，基本上也就是一个逻辑DB，它把所有分片数据库上存储的数据逻辑合在一处，对外而言一个keyspace就是代表要进行数据分片的业务DB;

我们基于范围顺序进行分片，这个策略的优势是便于简单的内存查询，缺陷是会导致热点区，这种情况我们建议业务先进行哈希再对哈希值进行分片；

这样也暗示了所有的DML操作都需要指定分片键的键值，这样暗示了分片键基本上就是作为主键在使用；

分片键的数据类型应该是自然数值类型或varbinary,不支持varchar作为分片键，因为字符集不支持range分区

如果一个分片数据过热，vitess可以再次对其进行分片，一分为二，这是通过过滤复制来做到的。vitess也能将不同分片进行合并；

###Replication
You can specify a replication factor for a keyspace. This will make Vitess create that many replicas for each master database.

A shard map is a single list of databases that spans the entire keyspace. For example, a master shard map will contain all the master databases for the keyspace.

There will be a server (wrangler) that can serve these shard maps to the application based on what it requires. For example, the application can request a “random” replica shard map for reading data, or it could request the master shard map if it wishes to write data, or needs to read up-to-date info.

Vitess will support multiple data centers. It assumes that there is only one master data center, where all the master databases reside. Each data center will have a 'wrangler' server that will serve information about the local shard map in that data center.
Vitess is capable of electing a new master for maintenance (or due to failure). It will also provide a scheme to correspondingly reparent all downstream replicas automatically.
Schema rollout
Many DDLs cannot be performed on high traffic live systems due to their locking requirements. Vitess will allow you to coordinate such rollout with un-noticeable downtime by deploying the DDL to offline replicas and a reparenting process.

Tablet server
The tablet server and vtocc share most of the same code. The tablet server performs everything that vtocc does, but is also aware that it’s part of a multi-sharded data store. It provides additional shard-aware functionality that supports the above described features like splits and schema deployments.

Why did we choose go?
Go is miles ahead of C++ and Java in terms of expressibility and close in terms of performance. It is also relatively simple and has a straightforward interaction with Linux system calls.

The main drawback is also its strength - the garbage collector. vtocc has made spot optimizations to minimize most of the adverse effects of Go’s stop-the-world gc. At this point, we are trading some amount of performance for greater creativity and efficiency at lower layers. unless you’re trying to max out on qps for your servers, you should see acceptable performance from vtocc. Also, go’s garbage collector is being improved. So, this should only get better over time. Go’s existing mark-and-sweep garbage collector is sub-optimal for systems that use large amounts of static memory (like caches). In the case of vtocc, this would be the row cache. To alleviate this, we intend to use memcache for the time being. If the gc ends up addressing this, it should be fairly trivial to switch to an in-memory row cache. Note that the row cache functionality is not fully ready yet.

```


## Concepts
We need to introduce some common terminologies that are used in Vitess:
### Keyspace
A keyspace is a logical database.
In its simplest form, it directly maps to a MySQL database name.
When you read data from a keyspace, it is as if you read from a MySQL database.
Vitess could fetch that data from a master or a replica depending
on the consistency requirements of the read.

When a database gets [sharded](http://en.wikipedia.org/wiki/Shard_(database_architecture)),
a keyspace maps to multiple MySQL databases,
and the necessary data is fetched from one of the shards.
Reading from a keyspace gives you the impression that the data is read from
a single MySQL database.

### Shard

A division within a Keyspace. All the instances inside a Shard have the same data (or should have the same data,
modulo some replication lag).

A Keyspace usually has one shard when not using any sharding (we name it '0' by convention). When sharded, a Keyspace will have N shards (usually, N is a power of 2) with non-overlapping data.

We support [dynamic resharding](Resharding.md), when one shard is split into 2 shards for instance. In this case, the data in the
source shard is duplicated into the 2 destination shards, but only during the transition. Afterwards, the source shard is
deleted.

A shard usually contains one MySQL master, and many MySQL slaves. The slaves are used to serve read-only traffic (with
eventual consistency guarantees), run data analysis tools that take a long time to run, or perform administrative tasks (backups, restore, diffs, ...)

### Tablet

A tablet is a single server that runs:
- a MySQL instance
- a vttablet instance
- a local row cache instance
- an other per-db process that is necessary for operational purposes

It can be idle (not assigned to any keyspace), or assigned to a keyspace/shard. If it becomes unhealthy, it is usually changed to scrap.

It has a type. The commonly used types are:
- master: for the mysql master, RW database.
- replica: for a mysql slave that serves read-only traffic, with guaranteed low replication latency.
- rdonly: for a mysql slave that serves read-only traffic for backend processing jobs (like map-reduce type jobs). It has no real guaranteed replication latency.
- spare: for a mysql slave not use at the moment (hot spare).
- experimental, schema, lag, backup, restore, checker, ... : various types for specific purposes.

Only master, replica and rdonly are advertised in the Serving Graph.

### Keyspace id
A keyspace id (keyspace_id) is a column that is used to identify a primary entity
of a keyspace, like user, video, order, etc.
In order to shard a database, all tables in a keyspace need to
contain a keyspace id column.
Vitess sharding ensures that all rows that have a common keyspace id are
always together.

It's recommended, but not necessary, that the keyspace id be the leading primary
key column of all tables in a keyspace.

If you do not intend to shard a database, you do not have to
designate a keyspace_id.
However, you'll be required to designate a keyspace_id
if you decide to shard a currently unsharded database.

A keyspace_id can be an unsigned number or a binary character column (unsigned bigint
or varbinary in mysql tables). Other data types are not allowed because of ambiguous
equality or inequality rules.

TODO: The keyspace id rules need to be solidified once VTGate features are finalized.

### Shard graph
The shard graph defines how a keyspace has been sharded. It's basically a per-keyspace
list of non-intersecting ranges that cover all possible values a keyspace id can cover.
In other words, any given keypsace id is guaranteed to map to one and only one
shard of the shard graph.

We are going with range based sharding.
The main advantage of this scheme is that the shard map is a simple in-memory lookup.
The downside of this scheme is that it creates hot-spots for sequentially increasing keys.
In such cases, we recommend that the application hash the keys so they
distribute more randomly.

For instance, an application may use an incrementing UserId as a primary key for user records,
and a hashed version of that UserId as a keyspace_id. All data related to one user will be on
the same shard, as all rows will share that keyspace_id.

### Replication graph
The [Replication Graph](ReplicationGraph.md) represents the relationships between the master
databases and their respective replicas.
This data is particularly useful during a master failover.
Once a new master has been designated, all existing replicas have to
repointed to the new master so that replication can resume.

### Serving graph
The [Serving Graph](ServingGraph.md) is derived from the shard and replication graph.
It represens the list of active servers that are available to serve
queries.
VTGate (or smart clients) query the serving graph to find out which servers
they are allowed to send queries to.

### Topology Server
The Topology Server is the backend service used to store the Topology data, and provide a locking service. The implementation we use in the tree is based on Zookeeper. Each Zookeeper process is run on a single server, but may share that server with other processes.

There is a global instance of that service. It contains data that doesn't change often, and references other local instances. It may be replicated locally in each Data Center as read-only copies. (a Zookeeper instance with two master instances per cell and one or two replicas per cell is a good configuration).

There is one local instance of that service per Cell (Data Center). The goal is to transparently support a Cell going down. When that happens, we assume the client traffic is drained out of that Cell, and the system can survive
using the remaining Cells. (a Zookeeper instance running on 3 or 5 hosts locally is a good configuration).

The data is partitioned as follows:
- Keyspaces: global instance
- Shards: global instance
- Tablets: local instances
- Serving Graph: local instances
- Replication Graph: the master alias is in the global instance, the master-slave map is in the local cells.

Clients usually just read the local Serving Graph, therefore they only need the local instance to be up. Also, we provide a caching layer for Zookeeper, to survive local Zookeeper failures and scale read-only access dramatically.

### Cell (Data Center)

A Cell is a group of servers and network infrastructure collocated in an area. It is usually a full Data Center, or a subset of a full Data Center.

A Cell has an associated Topology Server, hosted in that Cell. Most information about the tablets in a cell is hosted in that cell's Topology Server. That way a Cell can be taken down and rebuilt as a unit, for instance.

We try to limit cross-cell traffic (both for data and metadata), and gracefully handle cell-level failures (like a Cell being cut off the network). Having the ability to route client traffic to Cells individually is a great feature to have
(but not provided by the Vitess software).



#Motivation和愿景
```
### RDB/ NOSQL/ VITESS
关系数据库（如MySQL）进行初步建成和非Web OLTP系统进行了优化。然而，他们仍然设法满足大多数的大型网络应用的需求。他们的许多传统的需求已经从能够以最佳方式满足当今应用的需求举行他们回来。这导致了备用的存储解决方案（NoSQL的），其在某些方面已经抛出婴儿连同洗澡水的发展。

### SCALE MYSQL OUT
随着Vitess，我们采取不同的方法。我们认为像MySQL数据库有什么需要，以提供高效的数据存储层。现在缺少的是能够轻松地向外扩展，然后协调一个单一的逻辑架构的多个实例的能力。我们计划实现这一目标的方式是通过提供一个子集的SQL前端有限担保和松耦合分布式系统来自动完成复杂的管理方案。

###连接池
无会话连接：一个MySQL连接进行了很多方面的。这通常是没有必要的网络应用程序。大多数Vitess连接是无会话，这使得它们重量轻，这使我们能够集中在极少数的连接到MySQL的。

###事务的原子性
ACID（原子）：此功能是一个矫枉过正的大多数Web应用程序。没有履行全系统的原子性，Vitess给你的原子保证对于一个给定的实体通过“钥匙”（比如用户ID）解决。这使我们能够通过拆分重点范围透明规模的数据库。

###事务的一致性
ACID（一致性）：Vitess放松对数据是一致的，最终这条规则。这使我们能够使用复制到分发情况读取交通那里的应用程序并不需要先进的最新数据。您可以随时要求从主数据库中读取数据，如果你明确需要多达最新数据。

### ROW CACHE
缓冲区高速缓存与行缓存：MySQL的缓冲区高速缓存的范围扫描超过指标和表，特别是当数据被密密麻麻的优化。不幸的是，这不是很好的随机访问表。 Vitess将允许你指定某些表作为随机访问。对于这样的情况下，将保持基于行的缓存，让他们通过派出了可能会影响他们的所有DMLS一致。
```

#Vitess功能集
```
###分片
每个分片数据库里的所有表都需要含有一个列，这个列是分片键shard key,Vitess将会依据这些分片键的值决定到哪一个分片数据库去查找或修改数据；

所有以分片键作为索引的表合在一起就是一个keyspace，基本上也就是一个逻辑DB，它把所有分片数据库上存储的数据逻辑合在一处，对外而言一个keyspace就是代表要进行数据分片的业务DB;

我们基于范围顺序进行分片，这个策略的优势是便于简单的内存查询，缺陷是会导致热点区，这种情况我们建议业务先进行哈希再对哈希值进行分片；

这样也暗示了所有的DML操作都需要指定分片键的键值，这样暗示了分片键基本上就是作为主键在使用；

分片键的数据类型应该是自然数值类型或varbinary,不支持varchar作为分片键，因为字符集不支持range分区

如果一个分片数据过热，vitess可以再次对其进行分片，一分为二，这是通过过滤复制来做到的。vitess也能将不同分片进行合并；


###复制
您可以为密钥空间指定复制因子。这将使Vitess创建许多副本的每个主数据库。
一个碎片地图数据库，涵盖了整个密钥空间的单一列表。例如，主碎片地图将包含所有的密钥空间的主数据库。
会有一个服务器（牧马），它可以服务于这些碎片映射到基于它所需要的应用程序。例如，应用程序可以请求一个“随机”副本碎片的地图为读取数据，或者如果它希望写入数据，或需要读取向上的最新信息也可以要求在主碎片地图。
Vitess将支持多个数据中心。它假定只有一个主数据中心，在那里所有的主数据库驻留。每个数据中心都会有，这将有助于了解当地的地图碎片在该数据中心信息的“牧马人”的服务器。
Vitess能够选出一个新的主维修的（或因故障）。它还将提供一项计划，以相应reparent自动所有下游副本。
架构部署
许多的DDL不能在高流量带电系统进行，由于它们的锁定要求。 Vitess将允许您通过部署DDL脱机副本和一个重排根目录的过程来协调这样的部署与联合国明显的停机时间。

平板电脑服务器
平板电脑服务器和vtocc份额大部分相同的代码。平板电脑服务器执行一切vtocc做，但也意识到，这是一个多分片数据存储的一部分。它提供了支持上述类似的分裂和架构的部署功能的其他碎片感知功能。

为什么我们选择去了？
围棋是未来C ++和Java英里的表达性方面，密切在性能方面。它也比较简单，并具有使用Linux系统调用的直接相互作用。

的主要缺点是也其强度 - 垃圾收集器。 vtocc作出点的优化，以尽量减少大部分的Go的停止的世界GC的不利影响。在这一点上，我们是贸易的更大的创造力和效率性能的一些量在下层。除非你想最大出来QPS为您的服务器，你应该看到vtocc可接受的性能。此外，走的垃圾收集器正在改善。所以，这应该只得到随着时间的推移更好。转到现有的标记 - 清除垃圾收集器是次优的使用大量的静态存储器（如高速缓存）系统。在vtocc的情况下，这将是该行的高速缓存。为了缓解这个问题，我们打算使用的memcache暂时。如果GC结束处理这一点，应该是相当微不足道切换到内存中的行高速缓存。需要注意的是该行的缓存功能没有完全准备好。




##概念
我们需要引进那些在Vitess中常用的一些术语：
###密钥空间
一个密钥空间是一个逻辑数据库。
最简单的形式，它直接映射到一个MySQL数据库名。
当你从一个密钥空间中读取数据，这是因为如果你从一个MySQL数据库读取。
Vitess可以从主获取的数据或根据副本
在读出的一致性要求。

当一个数据库中获取[分片]（http://en.wikipedia.org/wiki/Shard_（database_architecture）），
一个密钥空间映射到多个MySQL数据库，
和必要的数据从碎片中的一个取出。
从密钥空间读给你的印象是从中读取数据
一个MySQL数据库。

###碎片

一个密钥空间内的分工。所有一个碎片内的实例具有相同的数据（或应当具有相同的数据，
模某些复制滞后）。

密钥大小通常在不使用任何分片（我们将其命名为'0'约定）1碎片。当分片，一个密钥空间将具有N个碎片（通常，N是2的幂）与非重叠的数据。

我们支持[动态resharding]（Resharding.md），当一个碎片被分成2碎片的实例。在这种情况下，在该数据
源碎片被复制到2目的地碎片，但只有在过渡。随后，源分片
删除。

一个碎片通常包含一个MySQL主，很多MySQL的奴隶。奴隶被用来为只读流量（带
最终一致性保证），运行数据分析工具，需要很长的时间来运行，或执行管理任务（备份，恢复，差异，...）

###平板

片剂是运行在一台服务器：
- 一个MySQL实例
- 一个vttablet实例
- 一个本地行缓存实例
- 一个其他每分贝过程所必需的操作的目的

它可以是空闲（未分配给任何密钥空间），或分配给一个密钥空间/碎片。如果它变得不健康时，通常改变为报废。

它有一个类型。常用的类型是：
- 主：对于MySQL主，RW数据库。
- 副本：对于MySQL从供应只读的交通，有保障的低复制延迟。
- RDONLY：对于MySQL从供应只读流量后端处理作业（如地图，减少类型的工作）。它有没有真正的保证复制延迟。
- 备用：对于MySQL从不会在此刻（热备用）使用。
- 实验，模式，滞后，备份，恢复，检查，...：各类用于特定目的。

只有主，副本和RDONLY通告在服务图。

###密钥空间ID
甲密钥空间ID（keyspace_id）是用于识别一个主实体的列
一个密钥空间，喜欢的用户，视频，秩序，等等。
以分片的数据库，在密钥空间中的所有表需要
包含密钥空间id列。
Vitess拆分可确保有一个共同的密钥空间的id的所有行均
永远在一起。

它的建议，但不是必要，该密钥空间ID是主要主
在密钥空间中所有表的键列。

如果你不打算分片的数据库，你不必
指定keyspace_id。
然而，你必须指定一个keyspace_id
如果你决定分片当前unsharded数据库。

一个keyspace_id可以是一个无符号数或二进制字符列（无符号BIGINT
或varbinary在MySQL表）。因为不明确的其他数据类型是不允许
平等或不平等的规则。

TODO：密钥空间ID规则需要被固化，一旦VTGate功能完成。

###碎片图
该碎片图定义了一个密钥空间已经分片。它基本上是每个密钥空间
的非相交范围涵盖所有可能的值的密钥空间ID可以覆盖列表。
换句话说，任何给定的keypsace ID是保证映射到一个且仅一个
碎片的碎片图形。

我们将与范围基于分片。
该方案的主要优点是，分片映射是一个简单的内存查找。
此方案的缺点是，它创造热点依次增加键。
在这种情况下，我们建议应用散列密钥，以便它们
分布更随意。

例如，一个应用程序可以使用一个递增用户ID作为主键为用户记录，
并且用户ID作为keyspace_id的哈希版本。涉及到一个用户的所有数据将是
同样的碎片，因为所有的行会共享keyspace_id。

###复制图
在[复制图]（ReplicationGraph.md）代表主之间的关系
数据库及其相应的副本。
这个数据是一个主故障转移期间特别有用。
一旦一个新的主已被指定，所有现有的副本都
重新瞄准到新的主，这样复制就可以恢复。

###服务图
在[服务图]（ServingGraph.md）从碎片和复制图形而得。
它represens可用来服务的活动服务器的列表
查询。
VTGate（或智能客户端）查询服务图，找出哪些服务器
他们被允许发送查询。

###拓扑服务器
拓扑服务器是用于存储拓扑数据，并提供一个锁定服务后端服务。我们在树使用的实现是基于动物园管理员。每个动物园管理员进程运行在单个服务器上，但也可以与其他进程共享服务器。

有该服务的全局实例。它包含的数据不经常更改，并参考其他地方的情况。它可以在本地在每个数据中心复制只读副本。 （每单元有两个主实例和每单元一个或两个副本的动物园管理员实例是一个不错的配置）。

还有的是，每个服务单元（数据中心）的一个本地实例。我们的目标是透明地支持一个细胞下降。当发生这种情况，我们假设流量排出该细胞的客户端，并且该系统可以生存
用剩余的细胞。 （3或5主机本地运行一个动物园管理员实例是一个不错的配置）。

该数据被划分为：
- Keyspaces：全局实例
- 碎片：全局实例
- 片剂：本地实例
- 服务图：本地实例
- 复制图：主别名在全局实例，主从地图是在当地细胞。

客户通常只是读取本地服务图表，所以他们只需要在本地实例待涨。此外，我们提供了一个缓存层的动物园管理员，生存当地动物园管理员的故障，并极大地扩展只读访问。

###单元（数据中心）

小区是一组服务器和网络基础设施并置的区域中的。它通常是一个完整的数据中心，或者一个完整的数据中心的一个子集。

一个单元有一个相关的拓扑服务器，托管在该单元格。大约在单元格中的药片大部分信息托管在该单元格的拓扑服务器。这样，一个小区可以取下来，并重建为单位，例如。

我们试图限制跨小区业务（包括数据和元数据），并妥善处理单元级的故障（如电池被切断网络）。有能力将客户端流量细胞分别是一个很大的特点有
（但不是由Vitess软件提供）。


#vitess迷思

##一。数据一致性模型
```
在vitess的官方介绍资料中提到vitess与关系型数据库，NoSQL的数据库的区别：
关系型数据库，比如MySQL的产品
优点有：
*完整支持事务
*支持索引
*支持连接操作
缺点有：
*不支持数据分片（即分布式数据库功能）
*完整的ACID（这也是缺点吗，存疑）
*强架构表结构导致易用性变差
vitess产品
优点：
*受限的事务支持
*支持索引
*支持连接操作
*支持数据分片
缺点有：
*数据一致性不是强一致性
*和关系型数据库一样，强表结构特征
NoSQL的产品比如HBase的产品
优点：
*数据分片
*非结构化数据
缺点
*最终一致性
*不支持事务
*不支持索引
*不支持连接

其他的都没有问题，vitess说自己是最终一致性这个如何解释？难道内部不是使用的主从复制，根据读操作的一致性要求，高要求走各个shards的master,低要求的读操作走slave？难道是双写，三写模式？这个需要确认。最终一致性是针对什么而言的呢?

查阅观念时发现如下解释，说明是可以保证OLTP事务的强一致性的。
一个密钥空间是一个逻辑数据库。最简单的形式，它直接映射到一个MySQL数据库名。当你从一个密钥空间中读取数据，这是因为如果你从一个MySQL数据库读取。 Vitess可以从主或取决于所读取的一致性要求的副本获取该数据。
原文档所说的最终一致性是指允许OLAP事务从slave上拉数据，当然因为主从复制是异步的，所以是最终一致性的。

#vitess迷思

##一. 数据一致性模型
```
在vitess的官方介绍资料中提到vitess与关系型数据库,NoSQL数据库的区别：
关系型数据库，比如MySQL产品
优点有：
* 完整支持事务
* 支持索引
* 支持连接操作
缺点有：
* 不支持数据分片（即分布式DB功能）
* 完整的ACID（这也是缺点吗，存疑）
* 强schema表结构导致易用性变差
vitess产品
优点：
* 受限的事务支持
* 支持索引
* 支持连接操作
* 支持数据分片
缺点有：
* 数据一致性不是强一致性
* 和关系型数据库一样，强表结构特征
NoSQL产品比如HBase产品
优点：
* 数据分片
* 非结构化数据
缺点
* 最终一致性
* 不支持事务
* 不支持索引
* 不支持连接

其他的都没有问题，vitess说自己是最终一致性这个如何解释？难道内部不是使用的主从复制，根据读操作的一致性要求，高要求走各个shards的master,低要求的读操作走slave？难道是双写，三写模式？这个需要确认。最终一致性是针对什么而言的呢?

查阅conceptions时发现如下解释，说明是可以保证OLTP事务的强一致性的。
A keyspace is a logical database. In its simplest form, it directly maps to a MySQL database name. When you read data from a keyspace, it is as if you read from a MySQL database. Vitess could fetch that data from a master or a replica depending on the consistency requirements of the read.
原文档所说的最终一致性是指允许OLAP事务从slave上拉数据，当然因为主从复制是异步的，所以是最终一致性的。
```





