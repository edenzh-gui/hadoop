# Hadoop 生态基础知识总结

## 一、 大数据与 Hadoop 简介
大数据通常具备 4V 特征：Volume（数据量大）、Velocity（处理速度快）、Variety（数据类型繁多）、Value（价值密度低）。
Hadoop 是 Apache 基金会旗下的一个开源分布式计算平台，主要为了解决两个核心问题：
1. **海量数据的存储**（HDFS）
2. **海量数据的计算**（MapReduce / YARN）

---

## 二、 核心组件 1：HDFS (分布式文件系统)
HDFS (Hadoop Distributed File System) 采用 Master/Slave（主从）架构。

```mermaid
graph TD
    Client[Client / 客户端] -->|1. 读写请求| NN(NameNode: 元数据大脑)
    NN -->|2. 管理与监控心跳| DN1(DataNode 1)
    NN -->|2. 管理与监控心跳| DN2(DataNode 2)
    NN -->|2. 管理与监控心跳| DN3(DataNode 3)
    Client <-->|3. 实际数据传输| DN1
    Client <-->|3. 实际数据传输| DN2
    Client <-->|3. 实际数据传输| DN3
    SNN(Secondary NameNode) -.->|定期合并 FsImage| NN
```

### 1. 核心设计思想
- **分块存储 (Block)**：大文件会被切割成多个 Block（Hadoop 3.x 默认 128MB），分散存储在多台服务器上。
- **多副本机制**：为了防止数据丢失，每个 Block 默认有 3 个副本，存放在不同的机器上。

### 2. 核心角色 (守护进程)
- **NameNode (NN - 大脑/主节点)**
  - 存储文件的元数据（文件名、目录结构、文件属性等）。
  - 记录每个文件块所在的 DataNode 节点信息。
  - **注意**：NN 的元数据存在内存中，如果单节点的 NN 挂掉，整个集群将无法访问。
- **DataNode (DN - 躯干/工作节点)**
  - 真正存储文件 Block 数据。
  - 定期向 NN 发送心跳和自己的块报告，证明自己“还活着”。
- **Secondary NameNode (2NN -秘书)**
  - 并不是 NN 的热备，而是用来辅助 NN 合并元数据日志（FsImage 和 Edits），减轻 NN 启动时的压力。

### 3. 高可用架构 (HA - High Availability) 与 JournalNode
在企业生产环境中，单台 NameNode 存在**单点故障风险**（如果 NN 宕机，整个集群将瘫痪）。为了解决这个问题，Hadoop 引入了 HA（高可用）机制。这也就是你提到的 **JournalNode** 大显身手的地方。

**HA 架构下的核心变化：**
- **取消 SecondaryNameNode**：HA 架构下不再需要 2NN，元数据合并工作直接由 Standby NameNode 完成。
- **双 NameNode 机制**：集群中会有两台 NameNode，一台是 **Active**（活跃状态，对外提供服务），另一台是 **Standby**（待命热备状态，时刻同步数据）。
- **JournalNode (JN - 日志节点)**：
  - **核心作用**：它是连接 Active NN 和 Standby NN 的数据共享桥梁。
  - **原理机制**：Active NN 会把每一次的元数据变更日志（Edits）写入到一组 JournalNode 集群中；Standby NN 则时刻盯着 JournalNode，一旦有新日志，立刻读取并在自己内存中重放。这样就保证了 Active 和 Standby 的数据时刻保持一致。
  - **部署特性**：JournalNode 通常部署为奇数个（如 3、5 个），只要有一半以上的节点存活，日志系统就能正常工作。
- **ZooKeeper (ZK) 与 ZKFC**：ZooKeeper 负责监控这两台 NN。如果 Active NN 突然宕机，ZooKeeper 会迅速反应，自动将 Standby NN 提升为新的 Active NN，实现无缝切换（自动故障转移）。

```mermaid
graph TD
    Client[Client / 客户端] -->|读写请求| ActiveNN(Active NameNode: 提供服务)
    ActiveNN -->|1. 写入变更日志| JN((JournalNodes 集群))
    StandbyNN(Standby NameNode: 随时待命) -->|2. 实时同步日志| JN
    ZK(ZooKeeper 集群) -.->|3. 监控心跳与自动选举切换| ActiveNN
    ZK -.->|3. 监控心跳与自动选举切换| StandbyNN
    ActiveNN -->|管理| DN(DataNodes)
    StandbyNN -.->|接收块报告| DN
```

---

## 三、 核心组件 2：YARN (分布式资源调度框架)
YARN (Yet Another Resource Negotiator) 就像是整个集群的“操作系统”，负责统一管理所有的 CPU 和内存资源。

```mermaid
graph TD
    Client[Client / 客户端] -->|1. 提交任务| RM(ResourceManager: 资源总管)
    RM -->|2. 分配资源并指令| NM1(NodeManager: 节点1)
    RM -->|2. 分配资源并指令| NM2(NodeManager: 节点2)
    NM1 -->|3. 启动任务| AM(ApplicationMaster: 任务负责人)
    AM -.->|4. 向 RM 申请资源| RM
    AM -->|5. 指挥执行| Task1(Container: 运行任务)
    NM2 -->|5. 启动任务| Task2(Container: 运行任务)
    AM -.->|6. 监控任务进度| Task1
    AM -.->|6. 监控任务进度| Task2
```

### 1. 核心角色
- **ResourceManager (RM - 资源总管/主节点)**
  - 掌握整个集群的计算资源。
  - 接收用户的计算任务，分配最初的资源。
- **NodeManager (NM - 单机管家/工作节点)**
  - 运行在每个 DataNode 机器上，管理这台机器上的 CPU 和内存。
  - 负责启动和监控容器（Container，资源隔离的基本单位）。
- **ApplicationMaster (AM - 任务负责人)**
  - 每个提交的计算任务都会产生一个专属的 AM。
  - 负责向 RM 申请资源，并在拿到资源后，指挥 NM 启动任务计算逻辑，并监控任务的执行状态。

---

## 四、 核心组件 3：从 MapReduce 到 Spark (计算引擎演进)

### 1. 传统的 MapReduce
MapReduce 是 Hadoop 早期绑定的计算引擎，核心思想是**“分而治之”**。
- **Map 阶段**：将复杂庞大的数据拆分，由多台机器并行处理。
- **Reduce 阶段**：将 Map 的中间结果汇总得出最终结论。
> *局限性：每次 Map 或 Reduce 处理完，中间数据都必须写回磁盘。频繁的磁盘 I/O 导致它极其缓慢，目前在实际企业开发中已被大量淘汰。*

### 2. 新一代王者：Spark
为了解决 MapReduce 的速度瓶颈，**Spark** 横空出世，目前已是企业级批处理计算的绝对主流。
- **核心优势（内存计算）**：Spark 引入了 RDD（弹性分布式数据集）的概念，将中间结果尽量缓存在**内存**中。它的速度通常比 MapReduce 快 10 到 100 倍！
- **与 Hadoop 的绝佳搭配 (Spark on YARN)**：Spark 本身只是一个“计算引擎”，虽然自带调度器，但在企业里标准做法是将它挂载在 Hadoop 之上，各司其职：
  - **存储底座 ➡ HDFS**
  - **资源管家 ➡ YARN**
  - **运算大脑 ➡ Spark**

```mermaid
graph TD
    User(开发者提交任务) -->|spark-submit| RM(YARN ResourceManager)
    RM -->|分配容器| NM(Worker节点 NodeManager)
    NM -->|启动执行器| Executor(Spark Executor: 内存计算)
    Executor <-->|读取/保存数据| HDFS(HDFS 底层存储)
```

---

## 五、 Hadoop 扩展生态图谱
除了上述核心架构，Hadoop 还有庞大的生态支撑实际业务：

1. **Hive (数据仓库)**：将 SQL 语句自动翻译为 MapReduce/Spark 任务。不会写 Java 也能做大数据分析，离线数仓建设必备！
2. **Flink (实时计算引擎)**：如果说 Spark 是批处理（离线计算）的王者，那么 Flink 就是流处理（实时计算）的霸主，适用于双十一实时大屏、风控拦截等。
3. **HBase (NoSQL 数据库)**：建立在 HDFS 上的列式数据库，支持对 PB 级数据的毫秒级随机读写。
4. **Flume / Sqoop / Kafka (数据通道)**：
   - Flume：采集服务器实时日志数据。
   - Sqoop：在 Hadoop 和关系型数据库（如 MySQL）之间倒腾数据。
   - Kafka：高性能的消息队列，用于缓冲海量数据，削峰填谷。
