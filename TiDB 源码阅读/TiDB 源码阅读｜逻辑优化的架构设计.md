# 逻辑优化的入口逻辑

本文将介绍 TiDB 中逻辑优化的入口逻辑，包括入口函数、优化规则的统一结构和逻辑优化规则列表。

## 入口函数：`logicalOptimize` 函数

`logicalOptimize` 函数是 TiDB 中逻辑优化的入口函数，位于 `pkg/planner/core/optimizer.go` 文件中。该函数负责应用一系列逻辑优化规则，将初始逻辑计划转换为优化后的逻辑计划。

### 函数签名

```go
func logicalOptimize(ctx context.Context, flag uint64, logic base.LogicalPlan) (base.LogicalPlan, error)
```

### 参数说明

- `ctx context.Context`：上下文信息，用于传递控制信息和跟踪执行过程
- `flag uint64`：优化标志位，控制启用哪些优化规则
- `logic base.LogicalPlan`：待优化的逻辑计划

### 返回值

- `base.LogicalPlan`：优化后的逻辑计划
- `error`：错误信息，如果优化过程中出现错误

### 执行流程

1. 函数首先创建优化跟踪操作对象 `logicalOptimizeOp`
2. 然后按顺序依次应用注册在 `optRuleList` 中的逻辑优化规则
3. 每个规则应用后，会检查计划是否被修改，并更新优化跟踪信息
4. 如果某个规则应用过程中出现错误，则立即终止优化过程并返回错误

**步骤 1**：创建优化跟踪操作对象

```go
logicalOptimizeOp := &optimizetrace.LogicalOptimizeOp{}
```

**步骤 2**：应用优化规则

```go
for i, rule := range optRuleList {
    // 检查规则是否启用
    if flag&rule.Mask != rule.Mask {
        continue
    }
    
    // 创建规则跟踪对象
    ruleOp := optimizetrace.NewRuleOptimizeOp(rule.Name())
    
    // 应用优化规则
    newLogicalPlan, changed, err := rule.Optimize(ctx, logic, ruleOp)
    if err != nil {
        return nil, err
    }
    
    // 更新逻辑计划
    logic = newLogicalPlan
    
    // 更新优化跟踪信息
    if changed {
        logicalOptimizeOp.AppendRuleOptimize(ruleOp)
    }
}
```

**步骤 3**：返回优化后的逻辑计划

```go
return logic, nil
```

## 优化规则的统一结构

TiDB 中所有的逻辑优化规则都实现了 `base.LogicalOptRule` 接口，该接口定义在 `pkg/planner/core/base/rule_base.go` 文件中。

### 接口定义

```go
type LogicalOptRule interface {
    // Optimize 方法应用优化规则并返回优化后的计划
    Optimize(context.Context, LogicalPlan, *optimizetrace.LogicalOptimizeOp) (LogicalPlan, bool, error)
    
    // Name 方法返回规则名称
    Name() string
}
```

### 方法说明

**`Optimize` 方法**：

参数：
- `context.Context`：上下文信息
- `LogicalPlan`：待优化的逻辑计划
- `*optimizetrace.LogicalOptimizeOp`：优化跟踪操作对象
  
返回值：
- `LogicalPlan`：优化后的逻辑计划
- `bool`：表示计划是否被修改，如果计划被修改返回 `true`，否则返回 `false`
- `error`：优化过程中发生的错误

**`Name` 方法**：返回优化规则的名称，用于日志和跟踪。

### 规则注册

所有优化规则都在 `optRuleList` 变量中注册，定义在 `pkg/planner/core/optimizer.go` 文件中。每个规则都有一个 `Mask` 字段，用于控制该规则是否启用。

## 逻辑优化规则

TiDB 的逻辑优化规则主要在 `pkg/planner/core/rule_*.go` 文件中实现。以下是主要的逻辑优化规则：

| Go 类名 | 规则类型 | 规则名称 | 规则英文名 | 简介 |
|---------|---------|---------|-----------|------|
| `PPDSolver` | 谓词优化类 | 谓词下推 | Predicate Push Down | 将过滤条件尽可能下推到数据源，减少数据扫描量 |
| `PredicateSimplification` | 谓词优化类 | 谓词简化 | Predicate Simplification | 简化复杂过滤条件，合并冗余表达式 |
| `JoinReOrderSolver` | 连接优化类 | 连接重排序 | Join Reorder | 调整连接顺序以减少中间结果集大小 |
| `OuterJoinEliminator` | 连接优化类 | 连接消除 | Join Elimination | 消除不必要的连接操作 |
| `ConvertOuterToInnerJoin` | 连接优化类 | 外连接转内连接 | Convert Outer to Inner Join | 将外连接转换为内连接以提高效率 |
| `SemiJoinRewriter` | 连接优化类 | 半连接重写 | Semi Join Rewrite | 优化 EXISTS、IN 等子查询的执行方式 |
| `AggregationPushDownSolver` | 聚合和投影类 | 聚合下推 | Aggregation Push Down | 将聚合操作下推到数据源，减少计算量 |
| `AggregationEliminator` | 聚合和投影类 | 聚合消除 | Aggregation Elimination | 消除不必要的聚合操作 |
| `SkewDistinctAggRewriter` | 聚合和投影类 | 聚合倾斜重写 | Skew Distinct Aggregation Rewrite | 处理数据倾斜情况下的聚合操作 |
| `ProjectionEliminator` | 聚合和投影类 | 投影消除 | Projection Elimination | 消除不必要的投影操作 |
| `DecorrelateSolver` | 子查询优化类 | 去相关优化 | Decorrelation | 将相关子查询转为非相关子查询 |
| `ColumnPruner` | 列裁剪和常量传播类 | 列裁剪 | Column Pruning | 移除计划中不需要的列 |
| `GcSubstituter` | 列裁剪和常量传播类 | 常量替换 | Constant Propagation | 使用常量值替换表达式 |
| `ConstantPropagationSolver` | 列裁剪和常量传播类 | 常量传播 | Constant Propagation Solver | 传播常量值，简化表达式 |
| `PushDownTopNOptimizer` | TopN 和窗口函数优化类 | TopN 下推 | TopN Push Down | 将 LIMIT 操作下推到数据源 |
| `DeriveTopNFromWindow` | TopN 和窗口函数优化类 | 从窗口函数派生 TopN | Derive TopN from Window | 将窗口函数转换为 TopN 操作 |
| `BuildKeySolver` | 构建键信息类 | 构建键信息 | Build Key Info | 收集唯一键信息到 Schema 中 |
| `ResultReorder` | 结果处理类 | 结果重排序 | Result Reorder | 重排序查询结果列 |
| `MaxMinEliminator` | 聚合函数优化类 | 最大/最小值消除 | Max/Min Eliminate | 优化 MAX/MIN 函数的执行 |
| `PartitionProcessor` | 分区优化类 | 分区处理 | Partition Processor | 分区裁剪和处理 |
| `CollectPredicateColumnsPoint` | 统计信息收集类 | 收集谓词列 | Collect Predicate Columns | 收集谓词中的列用于统计信息 |
| `SyncWaitStatsLoadPoint` | 统计信息加载类 | 同步等待统计信息 | Sync Wait Stats Load | 同步等待统计信息加载完成 |
| `PushDownSequenceSolver` | 序列优化类 | 序列值下推 | Push Down Sequence | 处理序列值下推 |
| `EliminateUnionAllDualItem` | 集合操作优化类 | 消除 UnionAll 空表 | Eliminate UnionAll Dual Item | 消除 UNION ALL 中的空表操作 |
| `ResolveExpand` | 扩展解析类 | 解析扩展 | Resolve Expand | 解析和处理计划中的扩展操作 | 
