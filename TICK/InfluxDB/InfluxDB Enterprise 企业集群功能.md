### 一. 权益
需要有效的Lisence才能启动`influxd-meta`或`influxd`。Lisence限制了可以添加到集群的data node数量，以及data node可以使用的CPU核心数。
如果没有有效的Lisence，此过程将终止。

### 二. 查询管理

查询管理在整个集群范围内运行。`SHOW QUERIES`和`KILL QUERY <ID>`在集群中所有的data node上运行。`SHOW QUERIES`会显示整个集群所有正在运行的查询。`KILL QUERY`可以终止集群中任何正在运行的查询。

### 三. 订阅内容

Kapacitor使用的订阅在集群中工作。对于任何节点的写入将通过所有受支持的订阅协议转发给订阅者。

### 四. 连续查询

#### 集群上的配置和操作注意事项
了解如何配置InfluxDB Enterprise以及这些配置如何影响连续查询(CQ)引擎非常重要。
数据节点配置

- data node配置

`[continuous queries]` [run-interval](https://docs.influxdata.com/enterprise_influxdb/v1.8/administration/config-data-nodes#run-interval-1s) - InfluxDB检查是否需要运行CQ的间隔时间。需要将此选项设置为CQ运行的最低间隔时间。例如：如果最频繁的CQ每分钟运行一次，需要将运行间隔设置为1m。
- meta node配置

`[meta]` [lease-duration](https://docs.influxdata.com/enterprise_influxdb/v1.8/administration/config-meta-nodes#lease-duration-1m0s) - data node从meta node获取租约的默认持续时间。满足租约期限后，租约会自动过期。租约持续时间确保在给定时间只有一个data node正在运行某一项。例如：连续查询使用租约，以便所有data node不会同一时间按运行相同的CQ
- CQ的执行时间
 
CQ被顺序执行。需要根据工作量合理分配。上述配置参数可能会对观察到的CQ行为产生影响。

CQ服务在每个节点上运行，但是在任何时候都只授予单个节点执行CQ的独立访问权限。但是，每次`run-interval`检查时(并假设节点当前未执行CQ)，节点都会尝试获取CQ租约。默认情况下，`run-interval`为一秒检查一次。因此，data node会积极检查是否可以获取租约。在所有CQ执行时间少于`lease-duration`(默认为1m)的集群上，很有可能第一个获得租约的数据节点在`run-interval`检查时仍将保留该租约。其他节点会被拒绝获取租约，并且当用于该租约的节点再次请求租约时，将续订该租约，并将到期时间扩展到`lease-duration`。因此，在大多数情况下，我们观察到的单data node 获得了CQ租约并保持不变。它有效地成为了CQ的执行者，直到被回收为止。

现在考虑以下情况，CQ的执行时间比`lease-duration`。因此，在租约到期时，大约1秒后另一个数data node请求被授予了租约。租约的原始持有节点正在忙于依次执行最初处理的CQ列表，现在持有租约的data node从列表的顶部开始执行CQ

基于这种情况，CQ似乎是并行执行的，因此多个data node本质上是依次滚动注册CQ，并且租约从一个节点滚动到另一个节点。这个时候可能意味着在某个时刻所有节点都在尝试执行相同的复杂CQ(并且可能会争抢资源，因为他们会覆盖每个节点上该查询所生成的点。最后可能会有一些阶段性的偏移)

为避免此状态。可取的方法是减少集群上的总体负载，因此应将租约期限设置为大于所有CQ的总执行时间的值。

根绝配置CQ的当前执行方式，解决并行性的方法是对尝试运行更复杂的CQ时使用Kapacitor。请参考[Kapacitor as a Continuous Query engine](https://docs.influxdata.com/kapacitor/v1.5/guides/continuous_queries/)。

### 五. PProf端点
meta data 对外公开`/debug/pprof`可以进行概要分析和故障排除


### 六. 分片操作
复制分片
复制分片状态
停止分片
删除分片
截断分片

以上功能通过meta服务上的API和`influxd-ctl`子命令提供

### 七. OSS转换
支持将OSS单服务器导入为第一个数据节点
有关分步说明，请参阅OSS到集群的迁移

### 八. 查询路由
查询引擎将跳过查询所需分片的故障节点。如果另一个节点上有副本，它将在该节点上重试。
### 九. 备份还原
InfluxDB Enterprise集群从版本0.7.1开始支持备份和还原功能。有关更多信息，请查看备份和还原

