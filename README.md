# Hadoop Learning & Deployment Guide

本项目包含了 Hadoop 生态的基础核心概念学习笔记，以及在真实服务器环境下搭建完全分布式 Hadoop 集群的标准操作流程（SOP）。

## 目录索引 (Contents)

1. [Hadoop 生态基础知识总结](Hadoop_Basic_Knowledge.md)
   - 大数据与 Hadoop 简介
   - HDFS (分布式文件系统)
   - YARN (资源调度框架)
   - MapReduce (计算框架)
   - 常见周边生态 (Hive, Spark, HBase, Zookeeper 等)

2. [Hadoop 完全分布式集群部署 SOP (3节点)](Hadoop_Cluster_SOP.md)
   - 集群节点规划 (1 Master, 2 Workers)
   - 部署前置准备 (免密登录、防火墙、JDK 配置)
   - 核心组件安装与配置
   - 集群分发与初始化
   - 启动与 Web UI 验证
