---
title: 查询计划简要分析
date: 2020-03-20
categories:
  - Mongo
tags:
  - Mongo
---

#### 查看sql执行计划
```sql
在查询语句后添加 explain 即可
eg. db.collection.find().explain()
```

#### 返回信息解读
```
queryPlanner (查询计划): 查询优化选择的计划细节和被拒绝的计划
    namespace 运行查询的指定命名空间
    indexFilterSet Boolean值、表示mongodb在查询中是否使用索引过滤
    winningPlan 由查询优化选择的计划文档
         stage: 表示查询阶段的字符串
         inputStage: 表示子查询的文档
         inputStages: 表示子过程的文档数组
executionStats(执行状态): 被选中执行计划和被拒绝执行计划的详细说明
     queryPlanner.nReturned－匹配查询条件的文档数
     queryPlanner.executionTimeMillis－计划选择和查询执行所需的总时间（毫秒数）
     queryPlanner.totalKeysExamined－扫描的索引总数
     queryPlanner.totalDocsExamined－扫描的文档总数
     queryPlanner.executionStages－显示执行成功细节的查询阶段树
          executionStages.works－指定查询执行阶段执行的“工作单元”的数量
          executionStages.advanced－返回的中间结果数
          executionStages.needTime－未将中间结果推进到其父级的工作周期数
          executionStages.needYield－存储层要求查询系统产生的锁的次数
          executionStages.isEOF－指定执行阶段是否已到达流结束
     queryPlanner.allPlansExecution－包含在计划选择阶段期间捕获的部分执行信息，包括选择计划和拒绝计划
 serverInfo,（服务器信息）：MongoDB实例的相关信息：
     winningPlan－使用的执行计划
          shards－包括每个访问片的queryPlanner和serverInfo的文档数组
```
