# MongoDB监控实战

## 一、常用的监控工具及手段

>• MongoDB Ops Manager 
>• Percona
>• 通用监控平台
>• 程序脚本

## 二、如何获取监控数据

>监控信息的来源： 
>• db.serverStatus()（主要）
>• db.isMaster()（次要）
>• mongostats 命令行工具（只有部分信息） 
>说明：db.serverStatus() 包含的监控信息是从上次开机到现在为止的累计数据.

## 三、serverStatus() Output

```bash
• host
• version
• process
• pid
• uptime
• uptimeMillis
• uptimeEstimate
• localTime
• asserts
• connections
• electionMetrics
• extra_info
• flowControl
• freeMonitoring
• flowControl
• freeMonitoring
• globalLock
• locks
• network
• opLatencies
• opReadConcernCounters
• opcounters
• opcountersRepl
• oplogTruncation
• repl
• storageEngine
• tcmalloc
• trafficRecording
• transactions
• transportSecurity
• wiredTiger
```

## 四、serverStatus() 主要信息

```bash
• connections: 关于连接数的信息；
• locks: 关于 MongoDB 使用的锁情况；
• network: 网络使用情况统计；
• opcounters: CRUD 的执行次数统计；
• repl: 复制集配置信息；
• wiredTiger: 包含大量 WirdTiger 执行情况的信息： 
	block-manager: WT 数据块的读写情况；
	session: session 使用数量； 
	concurrentTransactions: Ticket 使用情况；
• mem: 内存使用情况；
• metrics: 一系列性能指标统计信息；
```

## 五、监控报警的考量

>• 具备一定的容错机制以减少误报的发生；
>• 总结应用各指标峰值；
>• 适时调整报警阈值；
>• 留出足够的处理时间；

## 六、建议监控指标说明

| 指标                          | 功能                                                         | 采集方法                                                     |
| ----------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| opcounters（操作计数器）      | 查询、更新、插入、删除、getmore 和其他命令的的数量。         | db.serverStatus().opcounters                                 |
| tickets（令牌）               | 对 WiredTiger 存储引擎的读/写令牌数量。令牌数量表示了可以进入存储引擎的并发操作数量。 | db.serverStatus().wiredTiger.concurrentTransactions          |
| replication lag（复制延迟）   | 这个指标代表了写操作到达从结点所需要的最小时间。过高的 replication lag 会减小从结点的价值并且不利于配置了写关 注 w>1 的那些操作。 | db.adminCommand({'replSetGetStatus': 1})                     |
| oplog window（复制时间窗）    | 这个指标代表oplog可以容纳多长时间的写操作。它表示了一个从结点可以离线多长时间仍能够追上主节点。通常建议该值应大于24小时为佳。 | db.oplog.rs.find().sort({$natural: -1}).limit(1).next().ts -db.oplog.rs.find().sort({$natural: 1}).limit(1).next().ts |
| connections（连接数）         | 连接数应作为监控指标的一部分，因为每个连接都将消耗资源。应该计算低峰/正常/高 峰时间的连接数，并制定合理的报警阈值范围。 | db.serverStatus().connections                                |
| Query targeting（查询专注度） | 索引键/文档扫描数量比返回的文档数量，按秒平均。如果该值比较高表示查询系需要进行很多低效的扫描来满足查询。这个情况通常代表了索引不当或缺少索引来支持查询。 | var status = db.serverStatus()<br/>status.metrics.queryExecutor.scanned / status.metrics.document.returned<br/>status.metrics.queryExecutor.scannedObjects / status.metrics.document.returned |
| Scan and Order（扫描和排序）  | 每秒内内存排序操作所占的平均比例。内存排序可能会十分昂贵，因为它们通常要求缓冲大量数据。如果有适当索引的情况下，内存排序是可以避免的。 | var status = db.serverStatus()<br/>status.metrics.operation.scanAndOrder / status.opcounters.query |
| 复制集节点状态                | 每个节点的运行状态。如果节点状态不是PRIMARY、SECONDARY、ARBITER 中的一个，或无法执行上述命令则报警 | db.runCommand("isMaster")                                    |
| dataSize（数据大小）          | 整个实例数据总量（压缩前）                                   | 每个 DB 执行db.stats()；                                     |
| StorageSize（磁盘空间 大小）  | 已使用的磁盘空间占总空间的百分比。                           | 每个 DB 执行db.stats()；                                     |
|                               |                                                              |                                                              |