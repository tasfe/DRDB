###工具与服务
该vitess工具和服务器旨在帮助你，如果启动时DB还很小，但它可以帮助你把它扩展成一个完整的数据库舰队。
在早期阶段，连接池，rowcache等vttablet模块能帮助你提升现有硬件的利用效率。
当你进行水平扩展时，这些自动化工具也会让你得心应手。

###vtctl
vtctl是进行管理操作的主要工具。它可以被用来跟踪数据分片，复制拓扑图与DB类目的信息收集。
它也可以用来完成自动故障切换，数据的重新分片(reshard)等功能；
当vtctl执行DBA操作时，其实就是更新一些lockserver(zookeeper)的内部状态数据；
在vitess服务框架内的的其余部分会观察到这些变化，并做出相应的反应。
例如，如果一个master db出现故障后切换到一个新的master db,，vitess其他服务进程会
观察到变化并将新的写操作重定向到那台新master;

###vttablet
vttablet的主要功能是代理MySQL实例。
它的作用是尽力让查询操作的吞吐量最大化；
以及为了保护MySQL进行SQL限制与治理。
每个MySQL实例都有一个vttablet代理进程。
vttablet还能够执行从vtctl接管的必要管理操作。
它还提供了用于数据流服务，比如复制数据的过滤和数据导出。

####vtocc
vtocc是vttablet以前的版本。它处理查询管理操作(同vttablet),
但现在已经不是大系统的一部分,它是一个不需要拓扑服务的独立程序,
它被用在单元测试中,唯一作用就是查询服务(带连接池，查询去重等）。
请注意，我们最终可能会产生一个不带拓扑服务的vttablet版本以替换vtocc。

###vtgate
vtgate目标是提供统一DB服务操作界面，就是说所有业务的DB操作都会连接到
vtgate上而不是DBs，客户端向vtgate发送SQL, vtgate接收到SQL后会分析，重写，并重新路由这些
SQL到不同的vttablets实例上，最后汇总各个vttablets的返回结果给客户端；

###vtctld
vtctld是一个HTTP服务器程序，提供从浏览器查看存储在lockserver里的数据信息的服务；
这对于trouble-shooting很有帮助，能从更高层面观察所有DB服务器状态；

###vtworker
vtworker是管理长时间运行进程的工具； 它是插件式架构，提供一些方便操作tablets的
工具库，目前已经开发的功能有：
—— 水平扩展differ工作：检验数据水平分片过程中的数据一致性（水平扩展或水平合并）
—— 垂直扩展differ工作：检验数据垂直拆分过程中的数据一致性（垂直拆分或合并）
也很容易为表内数据完整性添加其他校验程序（比如校验类似外键的关系），
并可以跨分片进行数据检验（比如一个分片上的索引含有其他分片上的数据）

###vtprimecache
vtprimecache是用来加速复制功能的mysql缓冲填充器，如果某个mysql复制线程滞后了，
vtprimacache就会被激活然后开始读取relay日志，并开几个线程或连接去取binlog event对应的
更改操作,并填充相应的row记录到mysql buffer cache.
比如从现在开始算起2，3秒后要执行一条更新SQL:
update table X where id=2;
它现在就会执行select from table x WHERE id=2 填充到缓冲中，
实际生产环境中这可以加速复制线程30%到40%左右；

###Other support tools
*mysqlctl*: manage MySQL instances.
*zkctl*: manage ZooKeeper instances.
*zk*: command line ZooKeeper client and explorer.

