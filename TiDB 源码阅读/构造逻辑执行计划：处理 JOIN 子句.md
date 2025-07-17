# TiDB 源码阅读｜构造逻辑执行计划：处理 JOIN 子句

## `buildJoin` 函数处理流程

在 TiDB 中，`PlanBuilder.buildJoin` 函数用于将 SQL 语句中的 JOIN 子句转换为逻辑执行计划中的 JOIN 节点，位于 `pkg/planner/core/logical_plan_builder.go` 文件中，函数签名如下：

```go
func (b *PlanBuilder) buildJoin(ctx context.Context, joinNode *ast.Join) (base.LogicalPlan, error)
```

输出算子结构：

- `buildResultSetNode()` 函数返回值（没有右表）
- `LogicalJoin > buildResultSetNode(), buildResultSetNode()`（非 `ON` 子句关联的内关联）
- `LogicalSelection > LogicalJoin > buildResultSetNode(), buildResultSetNode()`（`ON` 子句关联的内关联）

函数处理逻辑如下：

**步骤 1**：在某些语句中时若 AST 的关联节点右侧为空，则直接调用 [`PlanBuilder.buildResultSetNode` 函数](https://github.com/ChangxingJiang/db-notes/blob/main/TiDB%20%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB/%E6%9E%84%E9%80%A0%E9%80%BB%E8%BE%91%E6%89%A7%E8%A1%8C%E8%AE%A1%E5%88%92%EF%BC%9A%E5%A4%84%E7%90%86%20FROM%20%E5%AD%90%E5%8F%A5.md#buildresultsetnode-%E5%87%BD%E6%95%B0%E5%A4%84%E7%90%86%E6%B5%81%E7%A8%8B) 构造左侧的结果集节点。

```go
if joinNode.Right == nil {
    return b.buildResultSetNode(ctx, joinNode.Left, false)
}
```

**步骤 2**：设置优化标志

```go
b.optFlag = b.optFlag | rule.FlagPredicatePushDown
b.optFlag = b.optFlag | rule.FlagJoinReOrder
b.optFlag |= rule.FlagPredicateSimplification
b.optFlag |= rule.FlagConvertOuterToInnerJoin
b.optFlag |= rule.FlagEmptySelectionEliminator
```

**步骤 3**：调用 [`PlanBuilder.buildResultSetNode` 函数](https://github.com/ChangxingJiang/db-notes/blob/main/TiDB%20%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB/%E6%9E%84%E9%80%A0%E9%80%BB%E8%BE%91%E6%89%A7%E8%A1%8C%E8%AE%A1%E5%88%92%EF%BC%9A%E5%A4%84%E7%90%86%20FROM%20%E5%AD%90%E5%8F%A5.md#buildresultsetnode-%E5%87%BD%E6%95%B0%E5%A4%84%E7%90%86%E6%B5%81%E7%A8%8B)，为关联节点的左右表分别构造逻辑执行计划算子。

```go
leftPlan, err := b.buildResultSetNode(ctx, joinNode.Left, false)
if err != nil {
    return nil, err
}

rightPlan, err := b.buildResultSetNode(ctx, joinNode.Right, false)
if err != nil {
    return nil, err
}
```

**步骤 4**：检查关联是否违反 CTE 递归限制，递归 CTE 不允许出现在 `LEFT JOIN` 的右表。

```go
// The recursive part in CTE must not be on the right side of a LEFT JOIN.
if lc, ok := rightPlan.(*logicalop.LogicalCTETable); ok && joinNode.Tp == ast.LeftJoin {
    return nil, plannererrors.ErrCTERecursiveForbiddenJoinOrder.GenWithStackByArgs(lc.Name)
}
```

**步骤 5**：合并左表和右表的 `handleHelper`

```go
handleMap1 := b.handleHelper.popMap()
handleMap2 := b.handleHelper.popMap()
b.handleHelper.mergeAndPush(handleMap1, handleMap2)
```

**步骤 6**：初始化 [`LogicalJoin` 逻辑算子](https://github.com/ChangxingJiang/db-notes/blob/main/TiDB%20%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB/%E9%80%BB%E8%BE%91%E6%89%A7%E8%A1%8C%E8%AE%A1%E5%88%92%E8%8A%82%E7%82%B9.md#logicaljoin%E8%BF%9E%E6%8E%A5%E6%93%8D%E4%BD%9C%E9%80%BB%E8%BE%91%E8%AE%A1%E5%88%92)，具体执行如下步骤：

- 根据 AST 节点及上层语句状态，设置 `StraightJoin` 字段，标记是否为直接连接
- `SetChildren`：将左表和右表的逻辑算子添加为 `LogicalJoin` 算子的子节点
- `SetSchema`：合并左表和右表的 `Schema` 信息
- `SetOutputNames`：合并左表和右表的输出字段信息

创建并初始化 JOIN 逻辑计划节点，设置其子节点、Schema 和输出列名。

```go
joinPlan := logicalop.LogicalJoin{StraightJoin: joinNode.StraightJoin || b.inStraightJoin}.Init(b.ctx, b.getSelectOffset())
joinPlan.SetChildren(leftPlan, rightPlan)
joinPlan.SetSchema(expression.MergeSchema(leftPlan.Schema(), rightPlan.Schema()))
joinPlan.SetOutputNames(make([]*types.FieldName, leftPlan.Schema().Len()+rightPlan.Schema().Len()))
copy(joinPlan.OutputNames(), leftPlan.OutputNames())
copy(joinPlan.OutputNames()[leftPlan.Schema().Len():], rightPlan.OutputNames())
```

**步骤 7**：根据 AST 中的关联类型，为新逻辑算子设置关联类型

根据 SQL 中指定的 `JOIN` 类型设置逻辑计划中的 `JOIN` 类型，并处理相关的 `NULL` 标志。

```go
switch joinNode.Tp {
case ast.LeftJoin:
    // left outer join need to be checked elimination
    b.optFlag = b.optFlag | rule.FlagEliminateOuterJoin
    joinPlan.JoinType = logicalop.LeftOuterJoin
    util.ResetNotNullFlag(joinPlan.Schema(), leftPlan.Schema().Len(), joinPlan.Schema().Len())
case ast.RightJoin:
    // right outer join need to be checked elimination
    b.optFlag = b.optFlag | rule.FlagEliminateOuterJoin
    joinPlan.JoinType = logicalop.RightOuterJoin
    util.ResetNotNullFlag(joinPlan.Schema(), 0, leftPlan.Schema().Len())
default:
    joinPlan.JoinType = logicalop.InnerJoin
}
```

**步骤 8**：合并左表和右表的 `FullSchema` 和 `FullNames`，并添加到新逻辑算子的 `FullSchema` 中

```go
// Merge sub-plan's FullSchema into this join plan.
// Please read the comment of LogicalJoin.FullSchema for the details.
var (
    lFullSchema, rFullSchema *expression.Schema
    lFullNames, rFullNames   types.NameSlice
)
if left, ok := leftPlan.(*logicalop.LogicalJoin); ok && left.FullSchema != nil {
    lFullSchema = left.FullSchema
    lFullNames = left.FullNames
} else {
    lFullSchema = leftPlan.Schema()
    lFullNames = leftPlan.OutputNames()
}
if right, ok := rightPlan.(*logicalop.LogicalJoin); ok && right.FullSchema != nil {
    rFullSchema = right.FullSchema
    rFullNames = right.FullNames
} else {
    rFullSchema = rightPlan.Schema()
    rFullNames = rightPlan.OutputNames()
}
if joinNode.Tp == ast.RightJoin {
    // Make sure lFullSchema means outer full schema and rFullSchema means inner full schema.
    lFullSchema, rFullSchema = rFullSchema, lFullSchema
    lFullNames, rFullNames = rFullNames, lFullNames
}
joinPlan.FullSchema = expression.MergeSchema(lFullSchema, rFullSchema)
```

**步骤 9**：对于外关联，则重置内表的 `NULL` 标志。

```go
// Clear NotNull flag for the inner side schema if it's an outer join.
if joinNode.Tp == ast.LeftJoin || joinNode.Tp == ast.RightJoin {
    util.ResetNotNullFlag(joinPlan.FullSchema, lFullSchema.Len(), joinPlan.FullSchema.Len())
}
```

**步骤 10**：合并左表和右表完整的列名列表，添加到新逻辑节点的 `FullNames` 字段中。

```go
// Merge sub-plan's FullNames into this join plan, similar to the FullSchema logic above.
joinPlan.FullNames = make([]*types.FieldName, 0, len(lFullNames)+len(rFullNames))
for _, lName := range lFullNames {
    name := *lName
    joinPlan.FullNames = append(joinPlan.FullNames, &name)
}
for _, rName := range rFullNames {
    name := *rName
    joinPlan.FullNames = append(joinPlan.FullNames, &name)
}
```

**步骤 11**：如果用户通过提示指定了 `JOIN` 首选算法，则指定关联的首选

```go
// Set preferred join algorithm if some join hints is specified by user.
joinPlan.SetPreferredJoinTypeAndOrder(b.TableHints())
```

**步骤 12**：对于 `NATURAL JOIN`，调用 `PlanBuilder.buildNaturalJoin` 函数处理。【TODO：`buildNaturalJoin` 函数】

`NATURAL JOIN` 会自动使用两个表中同名的列作为连接条件，不包含 ON 或 USING 条件。

```go
// "NATURAL JOIN" doesn't have "ON" or "USING" conditions.
//
// The "NATURAL [LEFT] JOIN" of two tables is defined to be semantically
// equivalent to an "INNER JOIN" or a "LEFT JOIN" with a "USING" clause
// that names all columns that exist in both tables.
//
// See https://dev.mysql.com/doc/refman/5.7/en/join.html for more detail.
if joinNode.NaturalJoin {
    err = b.buildNaturalJoin(joinPlan, leftPlan, rightPlan, joinNode)
    if err != nil {
        return nil, err
    }
}
// ... 其他关联方式 ...
return joinPlan, nil
```

**步骤 13**：如果使用 `USING` 子句关联（`oinNode.Using != nil`），则调用 `PlanBuilder.buildUsingClause` 函数处理。【TODO：`buildUsingClause` 函数】

```go
else if joinNode.Using != nil {
    err = b.buildUsingClause(joinPlan, leftPlan, rightPlan, joinNode)
    if err != nil {
        return nil, err
    }
}
// ... 其他关联方式 ...
return joinPlan, nil
```

**步骤 14**：如果使用 `ON` 子句关联（`joinNode.On != nil`），则执行以下处理流程：

```go
else if joinNode.On != nil {
    // ... 具体处理步骤 ...
}
// ... 其他关联方式 ...
return joinPlan, nil
```

**步骤 14.1**：设置当前语句处理上下文为 `onClause`。

```go
b.curClause = onClause
```

**步骤 14.2**：调用 `PlanBuilder.rewrite` 函数重写 `ON` 子句表达式。【TODO：`rewrite` 函数】

```go
onExpr, newPlan, err := b.rewrite(ctx, joinNode.On.Expr, joinPlan, nil, false)
if err != nil {
    return nil, err
}
```

**步骤 14.3**：检查重写后的计划是否发生变化，当前不支持 `ON` 子句中包含子查询。

```go
if newPlan != joinPlan {
    return nil, errors.New("ON condition doesn't support subqueries yet")
}
```

**步骤 14.4**：将表达式拆分为合取范式（CNF）项。

```go
onCondition := expression.SplitCNFItems(onExpr)
```

**步骤 14.5**：对于内连接（INNER JOIN），此时 `ON` 子句实际上被视作 `WHERE` 子句处理，将 `ON` 条件转换为 `LogicalSelection` 节点，并令关联节点作为其子节点，以便应用可能的去相关优化。

```go
// Keep these expressions as a LogicalSelection upon the inner join, in order to apply
// possible decorrelate optimizations. The ON clause is actually treated as a WHERE clause now.
if joinPlan.JoinType == logicalop.InnerJoin {
    sel := logicalop.LogicalSelection{Conditions: onCondition}.Init(b.ctx, b.getSelectOffset())
    sel.SetChildren(joinPlan)
    return sel, nil
}
```

**步骤 14.6**：对于外连接，则将 `ON` 条件直接附加到 `JOIN` 节点上。

```go
joinPlan.AttachOnConds(onCondition)
```

**步骤 15**：如果是没有 `ON` 子句或 `USING` 子句（`joinPlan.JoinType == logicalop.InnerJoin`），则视为笛卡尔积，为关联算子打上 `CartesianJoin` 标记。

```go
// If a inner join without "ON" or "USING" clause, it's a cartesian
// product over the join tables.
joinPlan.CartesianJoin = true
return joinPlan, nil
```
