# Hadoop HA 集群手动初始化步骤

> ⚠️ **注意**：
> 当你使用 Ansible 跑完 `ansible-playbook -i inventory/hosts.ini site.yml` 后，所有节点的软件、环境变量和配置文件都已经完美就绪！
> 
> 但对于一个全新的 Hadoop HA 集群，**千万不能直接 start-all**。你必须严格按照以下顺序进行一次“开荒初始化”。初始化只需在集群生命周期内执行一次。

---

### 第零步：使环境变量生效 (非常重要)
在你在某台机器上敲击 `hdfs` 这种命令之前，由于这可能是你之前的旧 SSH 会话，Ansible 刚刚写入的环境变量还未加载。
**在所有你需要敲命令的机器上，先执行一次：**
```bash
source /etc/profile
```

### 第一步：启动全体 ZooKeeper 集群
在所有 ZK 节点 (`spark01`, `spark02`, `spark03`) 上执行：
```bash
/opt/module/apache-zookeeper-3.7.2-bin/bin/zkServer.sh start
```

### 第二步：启动全体 JournalNode
为了让 NameNode 格式化时能把元数据同步给 JournalNode，必须先启动 JN。
在 **所有机器**（或者你在 inventory 里配置的 `journalnodes`，默认是 `spark04`, `spark05`, `spark06`）上执行：
```bash
hdfs --daemon start journalnode
```

### 第三步：格式化第一个 NameNode
在**主 NameNode** (`spark01`) 上执行格式化：
```bash
hdfs namenode -format
```
格式化成功后，立刻启动它：
```bash
hdfs --daemon start namenode
```

### 第四步：备用 NameNode 同步元数据
在**备用 NameNode** (`spark02`) 上执行同步命令（它会去向 spark01 讨要元数据）：
```bash
hdfs namenode -bootstrapStandby
```
同步完成后，也可以把它启动起来：
```bash
hdfs --daemon start namenode
```

### 第五步：格式化 ZKFC (ZooKeeper Failover Controller)
在 `spark01` 上执行一次即可（这会在 ZooKeeper 里创建 Hadoop HA 的专属锁节点）：
```bash
hdfs zkfc -formatZK
```

---

### 第六步：大功告成，一键启动剩余组件！
在 `spark01` 上执行最后的一键拉起脚本，它会帮我们把刚才没启动的 DataNode、ZKFC 以及 YARN 的所有组件全拉起来：
```bash
start-dfs.sh
start-yarn.sh
```

*(如果在执行 `start-dfs.sh` 时提示 `namenode is running... Stop it first`，请忽略，那是脚本在告诉你第一台和第二台 NN 已经在前几步启动了，它会继续去拉起其他没启动的服务。)*

### 验证集群
打开浏览器访问：
- `http://spark01:9870`
- `http://spark02:9870`
- `http://spark01:8088`
