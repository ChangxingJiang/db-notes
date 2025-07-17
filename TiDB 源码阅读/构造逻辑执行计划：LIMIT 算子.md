# TiDB 源码阅读｜构造逻辑执行计划：LIMIT 算子

## `buildLimit` 函数处理逻辑

`buildLimit` 函数用于构造 `LIMIT` 查询的逻辑执行计划，函数签名如下：

```go
func (b *PlanBuilder) buildLimit(src base.LogicalPlan, limit *ast.Limit) (base.LogicalPlan, error)
```

输出算子结构：

- `LogicalTableDual`（`LIMIT` 子句的 `count + offset = 0`）
- `LogicalLimit > child`

**步骤 1**：设置优化标志和 CTE 状态检查

设置优化标志 `FlagPushDownTopN`，并标记 CTE 是否包含限制性操作符

```go
b.optFlag = b.optFlag | rule.FlagPushDownTopN
// flag it if cte contain limit
if b.buildingCTE {
    b.outerCTEs[len(b.outerCTEs)-1].containRecursiveForbiddenOperator = true
}
```

**步骤 2**：调用 `extractLimitCountOffset` 函数从 `LIMIT` 子句中提取出 count 和 offset 值

```go
var (
    offset, count uint64
    err           error
)
if count, offset, err = extractLimitCountOffset(b.ctx.GetExprCtx(), limit); err != nil {
    return nil, err
}
```

**步骤 3**：如果 `count + offset` 溢出了 64 位整数，则减小 `count` 的值避免异常。

```go
if count > math.MaxUint64-offset {
    count = math.MaxUint64 - offset
}
```

**步骤 4**：如果 `count + offset = 0`，则创建一个空的 [`LogicalTableDual` 逻辑计划算子](https://github.com/ChangxingJiang/db-notes/blob/main/TiDB%20%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB/%E9%80%BB%E8%BE%91%E6%89%A7%E8%A1%8C%E8%AE%A1%E5%88%92%E8%8A%82%E7%82%B9.md#logicaltabledual%E5%8F%8C%E9%87%8D%E8%A1%A8%E9%80%BB%E8%BE%91%E8%AE%A1%E5%88%92)，继承原节点的 Schema 和输出列名，并返回空节点。

```go
if offset+count == 0 {
    tableDual := logicalop.LogicalTableDual{RowCount: 0}.Init(b.ctx, b.getSelectOffset())
    tableDual.SetSchema(src.Schema())
    tableDual.SetOutputNames(src.OutputNames())
    return tableDual, nil
}
```

**步骤 5**：创建 [`LogicalLimit` 逻辑计划算子](https://github.com/ChangxingJiang/db-notes/blob/main/TiDB%20%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB/%E9%80%BB%E8%BE%91%E6%89%A7%E8%A1%8C%E8%AE%A1%E5%88%92%E8%8A%82%E7%82%B9.md#logicallimit%E9%99%90%E5%88%B6%E8%A1%8C%E6%95%B0%E9%80%BB%E8%BE%91%E8%AE%A1%E5%88%92)，并初始化 `count` 和 `offset` 属性。

```go
li := logicalop.LogicalLimit{
    Offset: offset,
    Count:  count,
}.Init(b.ctx, b.getSelectOffset())
```

**步骤 6**：检查并应用表级提示，更新 `LogicalLimit` 逻辑计划算子的 `PreferLimitToCop` 字段。

```go
if hint := b.TableHints(); hint != nil {
    li.PreferLimitToCop = hint.PreferLimitToCop
}
```

**步骤 7**：将输入的节点添加为 `LogicalLimit` 逻辑计划算子的子节点

```go
li.SetChildren(src)
```

**步骤 8**：返回 `LogicalLimit` 逻辑计划算子

```go
return li, nil
```
