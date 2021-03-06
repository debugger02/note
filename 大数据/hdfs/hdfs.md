## HDFS 设计理念与目标

**假定硬件损坏是常态**

HDFS 实例可能会由成百上千的服务器组成，每个机器存储文件系统的部分数据。HDFS 总是会有部分组件处在不可用的状态，因此错误检测与迅速的自动恢复是 HDFS 的核心架构目标。

**流式数据访问**

HDFS 被设计为应对批量处理的场景，而不是用户的交互式使用。HDFS 更看重高吞吐而不是低延迟的数据获取。POSIX强加了许多针对HDF的应用程序不需要的硬需求。POSIX语义在一些关键领域被用来提高数据吞吐量。

**针对大数据集**

在HDFS中，一个典型的文件大小是千兆到兆字节。因此，HDFS被调整为支持大文件。 它应该提供高聚合数据带宽，并可扩展到单个集群中的数百个节点。它应该在一个实例中支持数千万个文件。

**简单一致性模型**

HDFS 应用需要对文件的获取是 write-once-read-many 的场景. 文件一旦创建、写入和关闭，除了追加和截断之外，不需要更改。支持将内容追加到文件末尾，但不能在任意点进行更新。 这种假设简化了数据一致性问题，并支持高吞吐量数据访问。MapReduce应用程序或web爬虫应用程序非常适合这个模型。

**移动计算比移动数据便宜**

如果一个应用程序请求的计算是在它所操作的数据附近执行的，那么它的效率要高得多。当数据集的大小很大时尤其如此。这将最小化网络拥塞并增加系统的总吞吐量。假设通常最好将计算迁移到数据所在的位置，而不是将数据移动到应用程序运行的位置。HDFS为应用程序提供接口，使其能够更靠近数据所在的位置。

**异构软硬件平台间的可移植性**

HDFS被设计成易于从一个平台移植到另一个平台。这有助于HDFS作为一个大型应用程序选择平台的广泛采用。

## HDFS 架构

HDFS 是 master/slave 架构. HDFS 集群由 一个 NameNode和多个 DataNodes 组成 ，HDFS 暴露一个文件系统的命名空间，允许用户将数据存储为文件。在内部一个文件被切分为多个 blocks，这些 blocks 存储在多个 datanode 节点，NameNode 执行文件系统命名空间操作，如打开、关闭和重命名文件和目录。它还确定块到数据节点的映射。数据节点负责为来自文件系统客户端的读写请求提供服务。DataNodes还根据NameNode的指令执行块创建、删除和复制。

<div align="center">
    <img src="../zzzimg/hadoop/hdfsarchitecture.png" width="50%" />
</div>

**Namenode**

NameNode 维护了文件系统的命名空间，文件系统命名空间与属性的任何改变都被记录在 NameNode，一个应用可以指定文件在HDFS中的副本个数，一个文件的副本数量被称为这个文件的副本因子，这些信息被存储在 NameNode.

NameNode 使用了被称作 `EditLog` 的事务日志来持久化发生在元数据的每个改变，例如，在 HDFS 中创建一个新文件会导致 NameNode 在 EditLog 中插入一条记录来指示这一点。类似地，更改文件的复制因子会导致在EditLog中插入新记录。NameNode使用其本地主机OS文件系统中的文件来存储EditLog。整个文件系统命名空间（包括块到文件的映射和文件系统属性）存储在一个名为 `FsImage` 的文件中。FsImage也作为文件存储在NameNode的本地文件系统中。

**Datanode**

DataNode 在本地文件系统中将 HDFS 数据存储为文件。DataNode不知道HDFS文件，它将每个HDFS数据块存储在本地文件系统中的单独文件中。DataNode不会在同一目录中创建所有文件。相反，它使用启发式方法来确定每个目录的最佳文件数，并相应地创建子目录。在同一目录中创建所有本地文件不是最佳选择，因为本地文件系统可能无法有效地支持单个目录中的大量文件。当DataNode启动时，它会扫描其本地文件系统，生成与每个本地文件对应的所有HDFS数据块的列表，并将此报告发送到NameNode。该报告称为Blockreport。

**副本选取**

为了最小化全局带宽消耗和读取延迟，HDFS尝试满足来自最靠近读卡器的副本的读取请求。如果与读卡器节点在同一机架上存在副本，则首选该副本以满足读取请求。如果HDFS集群跨越多个数据中心，则首选驻留在本地数据中心的副本，而不是任何远程副本。

## HDFS HA

### Quorum Journal Manager

HDFS高可用性特性通过提供在同一集群中运行两个（从3.0.0开始超过两个）冗余NameNodes的选项来解决单点故障问题，该节点采用主动/被动配置，并具有热备用。这允许在计算机崩溃的情况下快速故障转移到新的NameNode，或者为了计划的维护而由管理员启动的正常故障转移。

在典型的HA集群中，两台或多台独立的计算机被配置为NameNodes。在任何时间点，都有一个NameNodes处于 `Active` 状态，其他的处于 `standby` 状态。活动NameNode负责集群中的所有客户机操作，而备用节点只是充当worker，维护足够的状态，以便在必要时提供快速故障转移。

为了使 Standby 节点能够维持对 active 节点状态的同步，这两个节点都需要与一组被称作 “JournalNodes” (JNs) 的独立后台进程进行通信。Active 节点对命名空间的任何改动，将会在大多数的 JNs 中进行持久地记录这些修改。standby 节点能够从JNs读取编辑，并不断监视它们对编辑日志的更改, 当 standby 节点看到这些编辑时，它将它们应用到自己的名称空间。在发生故障转移的情况下，备用服务器将确保在将自身提升到活动状态之前已从 JournalNodes 读取所有编辑。这可确保在发生故障转移之前完全同步命名空间状态。

**JournalNode machines** 

运行 JournalNodes 的机器，JournalNode 后台进程相对轻量，所以这些后台进程可与其他hadoop后台进程部署在一起。但是必须至少在 3 个且是奇数节点运行 JournalNode , 因为 edit log modifications 必须写入大多数 JNs. 这将使系统能够容忍单个机器的故障。 当运行 N JournalNodes, 这个系统可以容忍最多 (N - 1) / 2 个节点的失败，并正常运行。

`注意`，在HA集群中，备用NameNodes还执行命名空间状态的检查点，因此无需在HA集群中运行辅助 NameNode、检查点node或BackupNode。事实上，这样做是错误的。这还允许正在将非启用HA的HDFS集群重新配置为启用HA的人重用以前专用于 Secondary NameNode 的硬件。

**部署**

与联邦配置类似，HA配置是向后兼容的，允许现有的单个NameNode配置在没有更改的情况下工作。新配置的设计使得集群中的所有节点可能具有相同的配置，而无需根据节点的类型将不同的配置文件部署到不同的计算机。

就像 HDFS Federation, HA clusters 重用 nameservice ID 去识别 HDFS instance. 另外，一个新的抽象被称作 NameNode ID 随着 HA 被添加，用于在集群中去识别不同的 NameNode. 为了支持所有NameNodes的单个配置文件，相关的配置参数以nameservice ID和NameNode ID作为后缀。

具体在 hdfs-site.xml 中的配置项如下：
- dfs.nameservices： 这个新命名服务的逻辑名称
- dfs.ha.namenodes.[nameservice ID]：在命名服务中每个 NameNode 的唯一标识
- dfs.namenode.rpc-address.[nameservice ID].[name node ID]：要侦听的每个NameNode的完全限定的RPC地址
- dfs.namenode.http-address.[nameservice ID].[name node ID] - 要侦听的每个NameNode的完全限定的http地址
- dfs.namenode.shared.edits.dir：NameNodes 将要写入读取 edits 的一组 JNs 的标识地址 
- dfs.client.failover.proxy.provider.[nameservice ID]: HDFS clients 用于联系 Active NameNode 的 java 类
- dfs.ha.fencing.methods：用于在 active namenode 发生 failover 时获取 active NameNode 数据的脚本或者 java 类列表，自带的两种方式有 shell and sshfence
  
**Automatic Failover**

> 以上的章节描述了如何配置人工 failover，人工模式下这个系统不会自动触发一个从 active 到 standby namenode 的 failover 过程,即使 active node 已经失败。
> Automatic failover 添加两个新的组件到 HDFS 部署: ZooKeeper quorum 与 ZKFailoverController process (abbreviated as ZKFC).

自动HDFS故障转移的实现依赖于ZooKeeper完成以下任务：
- 失败检测
- Active NameNode 选举
- 健康监控
- ZooKeeper session 管理
- ZooKeeper-based 选举








