# Hadoop + Spark + ZooKeeper 常见命令维护手册

这份手册整理了在日常开发、运维和排错时，最常使用的大数据组件命令。由于咱们配置了全局环境变量（`/etc/profile`），大部分 Hadoop 命令可以直接在终端任意位置执行。

---

## 一、 ZooKeeper 维护命令

ZooKeeper 是整个集群的基石，必须最早启动、最晚关闭。
*(注意：ZK 的命令需要进入到 `/opt/module/apache-zookeeper-3.7.2-bin` 目录下执行，或用绝对路径)*

### 1. 启停与状态查看
在 `spark01`、`spark02`、`spark03` 上分别执行：
```bash
# 启动 ZooKeeper 节点
bin/zkServer.sh start

# 停止 ZooKeeper 节点
bin/zkServer.sh stop

# 查看节点状态（会显示当前是 Leader 还是 Follower）
bin/zkServer.sh status

# 重启节点
bin/zkServer.sh restart
```

### 2. ZK 客户端命令 (排错用)
用于连接进 ZooKeeper 内部查看节点存活信息和 HA 锁。
```bash
# 进入客户端交互界面
bin/zkCli.sh

# 进去后可以查看当前 ZK 里存了哪些数据（例如查看 Hadoop HA 注册的节点）
[zk: localhost:2181(CONNECTED) 0] ls /
[zk: localhost:2181(CONNECTED) 1] ls /hadoop-ha
```

---

## 二、 Hadoop (HDFS) 维护命令

HDFS 的命令是最常用的，主要分为集群管理命令和文件操作命令。

### 1. 集群启停 (通常只需在 spark01 执行)
```bash
# 一键启动 HDFS (NameNode, DataNode, JournalNode, ZKFC)
start-dfs.sh

# 一键停止 HDFS
stop-dfs.sh

# 单独启停某个守护进程 (当某个节点意外挂掉时使用)
hdfs --daemon start namenode
hdfs --daemon stop datanode
```

### 2. HA 高可用管理命令
当发生脑裂或者你想强制切换 Active/Standby 状态时使用。
```bash
# 查看两个 NameNode 的当前状态 (谁是 active，谁是 standby)
hdfs haadmin -getServiceState nn1
hdfs haadmin -getServiceState nn2

# 强制进行主备切换 (比如把 nn1 切成 active)
hdfs haadmin -transitionToActive nn1

# 强制把 nn1 降级为 standby
hdfs haadmin -transitionToStandby nn1
```

### 3. 安全模式 (Safe Mode) 操作
集群刚启动时，HDFS 会短暂进入“只读”的安全模式。如果磁盘损坏或块丢失过多，会卡在安全模式出不来。
```bash
# 查看安全模式状态
hdfs dfsadmin -safemode get

# 强制离开安全模式 (危险操作，仅限确认数据无大碍时使用)
hdfs dfsadmin -safemode leave

# 强制进入安全模式 (运维维护或迁移数据时使用)
hdfs dfsadmin -safemode enter
```

### 4. 日常文件与目录操作 (完全类似 Linux Shell)
```bash
# 查看根目录下的文件
hdfs dfs -ls /

# 在 HDFS 上创建文件夹
hdfs dfs -mkdir -p /user/test/data

# 把本地的 file.txt 上传到 HDFS 的 /user/test/ 目录下
hdfs dfs -put ./file.txt /user/test/

# 把 HDFS 上的文件下载到本地当前目录
hdfs dfs -get /user/test/file.txt ./

# 查看 HDFS 上的文件内容
hdfs dfs -cat /user/test/file.txt

# 删除 HDFS 上的文件或目录 (加上 -skipTrash 彻底删除不进回收站)
hdfs dfs -rm -r /user/test/data
```

---

## 三、 Hadoop (YARN) 维护命令

YARN 是资源调度大管家，主要用于查看当前集群跑了什么任务、杀了什么卡死的任务。

### 1. 集群启停 (通常只需在 spark01 执行)
```bash
# 一键启动 YARN (ResourceManager, NodeManager)
start-yarn.sh

# 一键停止 YARN
stop-yarn.sh
```

### 2. 任务状态与管理 (Application)
```bash
# 列出当前正在运行的所有计算任务
yarn application -list

# 列出所有状态的任务 (包含已经运行完或失败的)
yarn application -list -appStates ALL

# 强行杀死某个卡住或死循环的任务 (非常有用的命令！)
yarn application -kill application_1623456789_0001

# 查看某个已结束任务的运行日志 (排查任务为什么挂掉)
yarn logs -applicationId application_1623456789_0001
```

### 3. 查看资源节点池 (Node)
```bash
# 查看所有 NodeManager 节点的状态 (存活节点数、分配情况)
yarn node -list -all
```

---

## 四、 Spark 运行与维护命令

Spark 没有像 HDFS 那样复杂的后台守护进程，它的命令主要是用来**提交任务**和**交互式开发**的。

### 1. 提交任务 (Spark-Submit)
企业中最常见的操作，将打好的 Jar 包丢到 YARN 集群上跑。
```bash
spark-submit \
  --class com.yourcompany.MainClass \
  --master yarn \
  --deploy-mode cluster \
  --driver-memory 2g \
  --executor-memory 4g \
  --executor-cores 2 \
  --num-executors 3 \
  /path/to/your/compiled-spark-job.jar \
  arg1 arg2
```
* **master yarn**：告诉 Spark 把任务交给 YARN 去调度。
* **deploy-mode cluster**：把任务的主控程序 (Driver) 也放到集群节点里去运行（生产环境标配）；如果设为 `client`，主控程序会跑在你敲命令的这台机器上，一旦你终端断开，任务就死了。

### 2. 交互式 Shell 开发
主要用来做数据探查、快速验证 API。
```bash
# 启动基于 Scala 的交互式终端
spark-shell --master yarn

# 启动基于 Python 的交互式终端 (需要各节点预装 Python)
pyspark --master yarn
```

### 3. 查看历史任务页面 (History Server - 选配)
如果你的 Spark 任务跑完就消失了，你想看已经跑完任务的耗时和执行图，需要启动 History Server（需要在 `spark-defaults.conf` 中配置日志存放路径）：
```bash
/opt/module/spark-3.5.1-bin-hadoop3/sbin/start-history-server.sh
```
启动后可以访问 `http://spark01:18080` 进行查看。

---

## 五、 终极排错大法：看日志！

当你敲命令没反应，或者发现某个进程起来马上又死了，**千万不要瞎猜，去看日志！**
这是大数据运维的核心灵魂。

* **ZooKeeper 日志**：`/opt/module/apache-zookeeper-3.7.2-bin/logs/zookeeper-*.out`
* **Hadoop HDFS/YARN 日志**：`/opt/module/hadoop-3.3.6/logs/` 目录下。
  * `hadoop-root-namenode-sparkXX.log` (找 NameNode 问题)
  * `hadoop-root-datanode-sparkXX.log` (找 DataNode 问题)
  * `hadoop-root-resourcemanager-sparkXX.log` (找 YARN 问题)

使用 `tail -n 100 日志路径` 或者 `less 日志路径` 来查看最后的 ERROR 或 Exception 堆栈。
