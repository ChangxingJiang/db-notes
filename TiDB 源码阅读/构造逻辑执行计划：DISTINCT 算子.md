# 构造逻辑执行计划：DISTINCT 算子

在 SQL 查询中，`DISTINCT` 关键字用于去除结果集中的重复行。TiDB 在构建逻辑执行计划时，通过 `Planbuilder.buildDistinct` 函数将 `DISTINCT` 转换为一个特殊的聚合（Aggregation）算子来实现去重，具体地：

- 将 `DISTINCT` 子句中的列作为 `GROUP BY` 的分组项
- 对所有列应用 `first_row` 聚合函数，确保每个分组只保留一行数据

`buildDistinct` 函数位于 `pkg/planner/core/logical_plan_builder.go` 文件中，函数签名如下：

```go
func (b *PlanBuilder) buildDistinct(child base.LogicalPlan, length int) (*logicalop.LogicalAggregation, error) 
```

输出算子结构：

- `LogicalAggregation > child`

即在输入的逻辑算子之上，添加一层 `LogicalAggregation` 算子。

**步骤 1**：设置优化标志，具体包括：

- `FlagBuildKeyInfo`：表示需要构建键信息【TODO：增加优化机制的说明】
- `FlagPushDownAgg`：表示需要尝试下推聚合算子【TODO：增加优化机制的说明】

```go
b.optFlag = b.optFlag | rule.FlagBuildKeyInfo
b.optFlag = b.optFlag | rule.FlagPushDownAgg
```

**步骤 2**：如果当前正在构建 CTE，则标记该 CTE 包含递归禁止的操作符（`DISTINCT` 在递归 CTE 中是被禁止的）。

```go
if b.buildingCTE {
    b.outerCTEs[len(b.outerCTEs)-1].containRecursiveForbiddenOperator = true
}
```

**步骤 3**：创建逻辑执行计划的 [`LogicalAggregation` 算子](https://github.com/ChangxingJiang/db-notes/blob/main/TiDB%20%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB/%E9%80%BB%E8%BE%91%E6%89%A7%E8%A1%8C%E8%AE%A1%E5%88%92%E8%8A%82%E7%82%B9.md#logicalaggregation%E8%81%9A%E5%90%88%E6%93%8D%E4%BD%9C%E9%80%BB%E8%BE%91%E8%AE%A1%E5%88%92)，并初始化如下字段：

- `AggFuncs`：初始化为空的聚合函数切片，预分配容量为子计划的列数
- `GroupByItems`：将子计划 Schema 的前 `length` 列转换为表达式作为分组项

```go
plan4Agg := logicalop.LogicalAggregation{
    AggFuncs:     make([]*aggregation.AggFuncDesc, 0, child.Schema().Len()),
    GroupByItems: expression.Column2Exprs(child.Schema().Clone().Columns[:length]),
}.Init(b.ctx, child.QueryBlockOffset())
```

**步骤 4**：如果存在表提示信息，则将提示中的聚合类型提示和聚合下推偏好添加到逻辑算子中。

```go
if hintinfo := b.TableHints(); hintinfo != nil {
    plan4Agg.PreferAggType = hintinfo.PreferAggType
    plan4Agg.PreferAggToCop = hintinfo.PreferAggToCop
}
```

**步骤 5**：为每一列创建聚合函数创建 `AggFuncFirstRow` 聚合函数，并添加到逻辑算子中。

`AggFuncFirstRow` 函数会在分组内取第一个值，这样与 `DISTINCT` 的语义一致，即保留每个分组的一条记录。

```go
for _, col := range child.Schema().Columns {
    aggDesc, err := aggregation.NewAggFuncDesc(b.ctx.GetExprCtx(), ast.AggFuncFirstRow, []expression.Expression{col}, false)
    if err != nil {
        return nil, err
    }
    plan4Agg.AggFuncs = append(plan4Agg.AggFuncs, aggDesc)
}
```

**步骤 6**：将原始逻辑算子设置为聚合算子的子节点

```go
plan4Agg.SetChildren(child)
```

**步骤 7**：令聚合算子继承原始逻辑算子的 Schema 和输出字段

```go
plan4Agg.SetSchema(child.Schema().Clone())
plan4Agg.SetOutputNames(child.OutputNames())
```

**步骤 8**：调整聚合算子输出列的类型

由于 `DISTINCT` 被重写为 `first_row` 聚合函数，需要确保输出列的类型与 `first_row` 函数的返回类型一致。

```go
for i, col := range plan4Agg.Schema().Columns {
    col.RetType = plan4Agg.AggFuncs[i].RetTp
}
```

【TODO】TiDB 还有专门的优化规则处理 `DISTINCT` 场景：

1. 当 `DISTINCT` 的列包含唯一键时，可以消除 `DISTINCT` 操作（见 `aggregationEliminateChecker.tryToEliminateDistinct`）
2. 对于数据倾斜场景，有特殊的重写规则（见 `rule_aggregation_skew_rewrite.go`）
