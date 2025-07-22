# TiDB 源码阅读｜构造逻辑执行计划：SELECT 语句

## `buildSelect` 函数

`PlanBuilder.buildSelect` 函数负责构建 `SELECT` 语句的逻辑执行计划，它位于 `pkg/planner/core/logical_plan_builder.go` 文件中，函数签名如下：

```go
func (b *PlanBuilder) buildSelect(ctx context.Context, sel *ast.SelectStmt) (p base.LogicalPlan, err error)
```

**步骤 1**：设置查询块偏移量和表提示信息

在开始构建 `SELECT` 语句的逻辑执行计划前，首先需要设置查询块偏移量 `PlanBuilder.QueryBlockOffset` 和表提示信息 `PlanBuilder.TableHints`。这些信息将在当前 `SELECT` 语句的处理过程中使用，处理完成后通过 `defer` 函数进行清理。

```go
b.pushSelectOffset(sel.QueryBlockOffset)
b.pushTableHints(sel.TableHints, sel.QueryBlockOffset)
defer func() {
    b.popSelectOffset()
    b.popTableHints()
}()
```

**步骤 2**：如果当前正在构造递归 CTE，则需要检查是否在递归 CTE 中不支持的操作，包括 `DISTINCT` 关键字（`sel.Distinct`）、`ORDER BY` 子句（`sel.OrderBy != nil`）、`LIMIT` 子句（`sel.Limit != nil`）或 `GROUP BY` 子句（`sel.GroupBy != nil`）。

> CTE（Common Table Expression，公共表表达式）是一种命名的临时结果集，可以在单个 SQL 语句中多次引用。在 TiDB 中，CTE 通过 `WITH` 子句定义，允许用户将复杂查询分解为更简单的部分。CTE 有两种类型：非递归 CTE 和递归 CTE。非递归 CTE 是一个普通的子查询，而递归 CTE 则可以引用自身，适用于处理层次化数据。递归 CTE 有特定的语法限制，例如不允许在递归部分使用 `DISTINCT`、`GROUP BY`、聚合函数、`ORDER BY` 或 `LIMIT` 等，这是为了确保递归过程能够正确终止。当系统检测到这些限制被违反时，会在此步骤中报错，确保查询的正确执行。

```go
if b.buildingRecursivePartForCTE {
    if sel.Distinct || sel.OrderBy != nil || sel.Limit != nil {
        return nil, plannererrors.ErrNotSupportedYet.GenWithStackByArgs("ORDER BY / LIMIT / SELECT DISTINCT in recursive query block of Common Table Expression")
    }
    if sel.GroupBy != nil {
        return nil, plannererrors.ErrCTERecursiveForbidsAggregation.FastGenByArgs(b.genCTETableNameForError())
    }
}
```

**步骤 3**：如果 SQL 语句中包含 `STRAIGHT_JOIN` 提示，则会在 `PlanBuilder.inStraightJoin` 字段中打上 `STRAIGHT_JOIN` 标记，指示优化器按照 `FROM` 子句中表的顺序进行连接，而不是自动选择最优的连接顺序。这些信息将在当前 `SELECT` 语句的处理过程中使用，处理完成后通过 `defer` 函数进行清理。

```go
if sel.SelectStmtOpts != nil {
    origin := b.inStraightJoin
    b.inStraightJoin = sel.SelectStmtOpts.StraightJoin
    defer func() { b.inStraightJoin = origin }()
}
```

**步骤 4**：声明计划构建需要的变量：

- `aggFuncs`：聚合函数的列表
- `havingMap`：`HAVING` 子句中聚合函数到下标的映射表
- `orderMap`：`ORDER BY` 子句中聚合函数到下标的映射表
- `totalMap`：聚合函数到下标的映射表
- `windowAggMap`：窗口函数到下标的映射表
- `correlatedAggMap`：相关聚合函数到下标的映射表
- `gbyCols`：`GROUP BY` 语句中的分组项的列表
- `projExprs`：投影字段的列表
- `rollup`：是否为 `ROLLUP` 分组模式

```go
var (
    aggFuncs                      []*ast.AggregateFuncExpr
    havingMap, orderMap, totalMap map[*ast.AggregateFuncExpr]int
    windowAggMap                  map[*ast.AggregateFuncExpr]int
    correlatedAggMap              map[*ast.AggregateFuncExpr]int
    gbyCols                       []expression.Expression
    projExprs                     []expression.Expression
    rollup                        bool
)
```

**步骤 5**：如果 SQL 语句包含 `FOR UPDATE` 锁定子句，则将 `PlanBuilder.isForUpdateRead` 标记置为 `true`，表明需要以更新模式读取数据。这将影响后续构建的算子行为，确保在事务中正确锁定相关行。

```go
if isForUpdateReadSelectLock(sel.LockInfo) {
    b.isForUpdateRead = true
}
```

**步骤 6**：如果查询中包含 CTE 合并提示（`MERGE` 提示）且当前是 CTE 查询，则更新 `PlanBuilder.outerCTEs` 中的标记 `forceInlineByHintOrVar`；若当前不是 CTE 查询（可能是 CTE 查询内的子查询或外部非 CTE 查询），则给出警告。

```go
if hints := b.TableHints(); hints != nil && hints.CTEMerge {
    if b.buildingCTE {
        if b.isCTE {
            // 如果当前查询使用合并提示并且是 CTE 查询，则更新当前查询的提示信息
            b.outerCTEs[len(b.outerCTEs)-1].forceInlineByHintOrVar = true
        } else if !b.buildingRecursivePartForCTE {
            // 如果子查询不是 CTE 但使用了 `MERGE()` 提示，则显示警告
            b.ctx.GetSessionVars().StmtCtx.SetHintWarning(
                "Hint merge() is inapplicable. " +
                    "Please check whether the hint is used in the right place, " +
                    "you should use this hint inside the CTE.")
        }
    } else if !b.buildingCTE && !b.isCTE {
        // 如果当前查询不是 CTE 查询，则显示警告
        b.ctx.GetSessionVars().StmtCtx.SetHintWarning(
            "Hint merge() is inapplicable. " +
                "Please check whether the hint is used in the right place, " +
                "you should use this hint inside the CTE.")
    }
}
```

**步骤 7**：处理 `WITH` 子句

如果 `SELECT` 语句中包含 `WITH` 子句（`sel.With != nil`），则调用 [`PlanBuilder.buildWith` 函数](https://github.com/ChangxingJiang/db-notes/blob/main/TiDB%20%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB/%E6%9E%84%E9%80%A0%E9%80%BB%E8%BE%91%E6%89%A7%E8%A1%8C%E8%AE%A1%E5%88%92%EF%BC%9A%E5%A4%84%E7%90%86%20WITH%20%E5%AD%90%E5%8F%A5.md) 构建公共表表达式（CTE）的列表，存储到 `currentLayerCTEs` 变量中。

`WITH` 子句中创建的 CTE 将存储到 `PlanBuilder.outerCTEs` 字段中，在当前 `SELECT` 语句的处理过程中使用，处理完成后通过 `defer` 函数进行清理。

```go
var currentLayerCTEs []*cteInfo
if sel.With != nil {
    l := len(b.outerCTEs)
    defer func() {
        b.outerCTEs = b.outerCTEs[:l]
    }()
    currentLayerCTEs, err = b.buildWith(ctx, sel.With)
    if err != nil {
        return nil, err
    }
}
```

**步骤 8**：调用 [`PlanBuilder.buildTableRefs` 函数](https://github.com/ChangxingJiang/db-notes/blob/main/TiDB%20%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB/%E6%9E%84%E9%80%A0%E9%80%BB%E8%BE%91%E6%89%A7%E8%A1%8C%E8%AE%A1%E5%88%92%EF%BC%9A%E5%A4%84%E7%90%86%20FROM%20%E5%AD%90%E5%8F%A5.md)，根据 `FROM` 子句构造逻辑算子。

在不考虑子查询、关联、脏更新等复杂情况的前提下，对于当前查询来说，`FROM` 子句中的节点就是当前逻辑执行计划树的叶子节点，也是实际与物理存储交互的节点。但如果 `FROM` 子句中的表为子查询，则实际为子查询的逻辑执行计划树的根节点。根据是否包含 `FROM` 子句、单表或多表关联、是否存在脏更新等条件，生成不同的逻辑算子。

```go
p, err = b.buildTableRefs(ctx, sel.From)
if err != nil {
    return nil, err
}
```

**步骤 9**：调用 [`unfoldWildStar` 函数](https://github.com/ChangxingJiang/db-notes/blob/main/TiDB%20%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB/%E6%9E%84%E9%80%A0%E9%80%BB%E8%BE%91%E6%89%A7%E8%A1%8C%E8%AE%A1%E5%88%92%EF%BC%9A%E9%80%9A%E9%85%8D%E7%AC%A6%E5%B1%95%E5%BC%80.md) 展开选择列表中的通配符（如 `*`），将其转换为具体的列引用。

```go
originalFields := sel.Fields.Fields
sel.Fields.Fields, err = b.unfoldWildStar(p, sel.Fields.Fields)
if err != nil {
    return nil, err
}
```

**步骤 10**：调用 [`addAliasName` 函数](https://github.com/ChangxingJiang/db-notes/blob/main/TiDB%20%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB/%E6%9E%84%E9%80%A0%E9%80%BB%E8%BE%91%E6%89%A7%E8%A1%8C%E8%AE%A1%E5%88%92%EF%BC%9A%E8%BE%93%E5%87%BA%E5%88%97%E5%91%BD%E5%90%8D.md)，为输出列字段添加别名。

```go
if b.capFlag&canExpandAST != 0 {
    // To be compatible with MySQL, we add alias name for each select field when creating view.
    sel.Fields.Fields, err = b.addAliasName(ctx, sel, p)
    if err != nil {
        return nil, err
    }
    originalFields = sel.Fields.Fields
}
```

**步骤 11**：处理 `GROUP BY` 分组算子，逻辑详见 [构造逻辑执行计划：处理 GROUP BY 子句](https://github.com/ChangxingJiang/db-notes/blob/main/TiDB%20%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB/%E6%9E%84%E9%80%A0%E9%80%BB%E8%BE%91%E6%89%A7%E8%A1%8C%E8%AE%A1%E5%88%92%EF%BC%9A%E5%A4%84%E7%90%86%20GROUP%20BY%20%E5%AD%90%E5%8F%A5.md)。

- 调用 `PlanBuilder.resolveGbyExprs` 函数解析 `GROUP BY` 子句中的表达式，获取解析后的表达式列表 `gbyExprs`；
- 如果启用了 `ONLY_FULL_GROUP_BY` 模式，则调用 `PlanBuilder.checkOnlyFullGroupBy` 函数检查是否满足 `ONLY_FULL_GROUP_BY` 模式要求；
- 调用 `PlanBuilder.rewriteGbyExprs` 函数将 `GROUP BY` 表达式重写为可执行的表达式。

```go
var gbyExprs []ast.ExprNode
if sel.GroupBy != nil {
    gbyExprs, err = b.resolveGbyExprs(p, sel.GroupBy, sel.Fields.Fields)
    if err != nil {
        return nil, err
    }
}

if b.ctx.GetSessionVars().SQLMode.HasOnlyFullGroupBy() && sel.From != nil && !b.ctx.GetSessionVars().OptimizerEnableNewOnlyFullGroupByCheck {
    err = b.checkOnlyFullGroupBy(p, sel)
    if err != nil {
        return nil, err
    }
}

if sel.GroupBy != nil {
    p, gbyCols, rollup, err = b.rewriteGbyExprs(ctx, p, sel.GroupBy, gbyExprs)
    if err != nil {
        return nil, err
    }
}
```

**步骤 12**：在处理 `HAVING` 子句之前，进行窗口函数的第一阶段处理，检查并解析窗口函数，详见 [构造逻辑执行计划：窗口函数](https://github.com/ChangxingJiang/db-notes/blob/main/TiDB%20%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB/%E6%9E%84%E9%80%A0%E9%80%BB%E8%BE%91%E6%89%A7%E8%A1%8C%E8%AE%A1%E5%88%92%EF%BC%9A%E7%AA%97%E5%8F%A3%E5%87%BD%E6%95%B0.md)。

- 调用 `PlanBuilder.detectSelectWindow` 函数，检测 `SELECT` 列表和 `ORDER BY` 子句中是否包含了窗口函数;
- 如果在 `SELECT` 列表和 `ORDER BY` 子句中包含了窗口函数（`hasWindowFuncField`），或者通过 `WINDOW` 语句定义了窗口（`sel.WindowSpecs != nil`），则：
  - 如果当前正在构建递归 CTE（`b.buildingRecursivePartForCTE`），则报错;
  - 调用 `PlanBuilder.resolveWindowFunction` 函数解析所有窗口函数，并返回一个窗口聚合函数映射表。

```go
hasWindowFuncField := b.detectSelectWindow(sel)
if hasWindowFuncField || sel.WindowSpecs != nil {
    if b.buildingRecursivePartForCTE {
        return nil, plannererrors.ErrCTERecursiveForbidsAggregation.FastGenByArgs(b.genCTETableNameForError())
    }

    windowAggMap, err = b.resolveWindowFunction(sel, p)
    if err != nil {
        return nil, err
    }
}
```

**步骤 13**：解析 `SELECT` 语句中的 `HAVING` 和 `ORDER BY` 子句，提取聚合函数，并处理相关的辅助字段，以保证诸如 `select a+1 as b from t having sum(b) < 0` 中的 `sum(b)` 可以被替换为 `sum(a+1)`，详见 [构造逻辑执行计划：解析 HAVING 和 ORDER BY 子句](https://github.com/ChangxingJiang/db-notes/blob/main/TiDB%20%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB/%E6%9E%84%E9%80%A0%E9%80%BB%E8%BE%91%E6%89%A7%E8%A1%8C%E8%AE%A1%E5%88%92%EF%BC%9A%E8%A7%A3%E6%9E%90%20HAVING%20%E5%92%8C%20ORDER%20BY%20%E5%AD%90%E5%8F%A5.md)。

```go
havingMap, orderMap, err = b.resolveHavingAndOrderBy(ctx, sel, p)
if err != nil {
    return nil, err
}
```

**步骤 14**：调用 `resolveCorrelatedAggregates` 函数收集相关聚合函数，用于后续构造聚合算子。

```go
correlatedAggMap, err = b.resolveCorrelatedAggregates(ctx, sel, p)
if err != nil {
    return nil, err
}
```

**步骤 15**：在构建投影算子之前，将当前逻辑计划的所有输出列名保存到 `PlanBuilder.allNames` 字段中。这是为了支持 `default()` 函数的正确解析，尤其是在 `ORDER BY` 子句中使用时。`default()` 函数需要找到对应的列名，但不需要列中的值。

例如，在 `select a from t order by default(b)` 语句中，列 `b` 不会出现在选择字段中。由于 `buildSort` 是在 `buildProjection` 之后执行的，所以需要在构建投影之前获取并保存所有输出列名。这样在处理 `ORDER BY` 子句中的 `default(b)` 时，才能正确找到列 `b`，即使它不在最终的选择字段中。

```go
b.allNames = append(b.allNames, p.OutputNames())
defer func() { b.allNames = b.allNames[:len(b.allNames)-1] }()
```

**步骤 16**：如果 `SELECT` 语句中包含 `WHERE` 子句（`sel.Where != nil`），则调用 `PlanBuilder.buildSelection` 函数构建筛选算子，用于过滤数据，详见 [构造逻辑执行计划：条件筛选](https://github.com/ChangxingJiang/db-notes/blob/main/TiDB%20%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB/%E6%9E%84%E9%80%A0%E9%80%BB%E8%BE%91%E6%89%A7%E8%A1%8C%E8%AE%A1%E5%88%92%EF%BC%9A%E6%9D%A1%E4%BB%B6%E7%AD%9B%E9%80%89.md)。

```go
if sel.Where != nil {
    p, err = b.buildSelection(ctx, p, sel.Where, nil)
    if err != nil {
        return nil, err
    }
}
```

**步骤 17**：如果`SELECT` 语句中是否包含锁定信息（`l != nil`），且锁类型不为空（`l.LockType != ast.SelectLockNone`），则构造锁算子，用于在事务中锁定相关行，详见 [构造逻辑执行计划：锁算子](https://github.com/ChangxingJiang/db-notes/blob/main/TiDB%20%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB/%E6%9E%84%E9%80%A0%E9%80%BB%E8%BE%91%E6%89%A7%E8%A1%8C%E8%AE%A1%E5%88%92%EF%BC%9A%E9%94%81%E7%AE%97%E5%AD%90.md)。

```go
l := sel.LockInfo
if l != nil && l.LockType != ast.SelectLockNone {
    // ...... 处理逻辑 ......
}
```

**步骤 18**：重置 `handleHelper` 的映射栈

在处理完 `WHERE`、`FOR UPDATE` 等子句后，需要重置 `PlanBuilder.handleHelper` 的映射栈。`handleHelper` 主要用于处理 `SELECT` 语句中的 handle（主键/唯一键）相关信息。这里先弹出当前的映射，再压入一个新的空映射，为后续的聚合、投影等操作做准备，确保 handle 相关信息不会被错误地继承到下一个阶段。

```go
b.handleHelper.popMap()
b.handleHelper.pushMap(nil)
```

**步骤 19**：如果 `SELECT` 字段中包含聚合函数（`hasAgg`），则会构建聚合算子；如果存在 `ROLLUP` 语法，则还需要构建 Expand 算子，用于复制数据以满足不同的分组布局。详见 [构造逻辑执行计划：聚合函数](https://github.com/ChangxingJiang/db-notes/blob/main/TiDB%20%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB/%E6%9E%84%E9%80%A0%E9%80%BB%E8%BE%91%E6%89%A7%E8%A1%8C%E8%AE%A1%E5%88%92%EF%BC%9A%E8%81%9A%E5%90%88%E5%87%BD%E6%95%B0.md)。

- 调用 `PlanBuilder.detectSelectAgg` 函数检测 SELECT 语句的 `SELECT` 字段列表、`HAVING` 子句和 `ORDER BY` 子句中是否包含聚合函数。
- 如果检测到聚合函数，则调用 `extractAggFuncsInSelectFields` 函数从 `SELECT` 字段中提取所有的聚合函数，并构建我聚合函数到输出下标的索引。
- 如果需要构建聚合操作（`needBuildAgg` 为 `true`），则需要执行如下逻辑：
  - 如果指定了 `ROLLUP` 语法，需要调用 `PlanBuilder.buildExpand` 函数构建 Expand 算子来复制数据以适应不同的分组布局。
  - 调用 `PlanBuilder.buildAggregation` 函数构建聚合算子。

```go
hasAgg := b.detectSelectAgg(sel)
needBuildAgg := hasAgg
if hasAgg {
    if b.buildingRecursivePartForCTE {
        return nil, plannererrors.ErrCTERecursiveForbidsAggregation.GenWithStackByArgs(b.genCTETableNameForError())
    }

    aggFuncs, totalMap = b.extractAggFuncsInSelectFields(sel.Fields.Fields)
    if len(aggFuncs) == 0 && sel.GroupBy == nil {
        needBuildAgg = false
    }
}
if needBuildAgg {
    if rollup {
        p, gbyCols, err = b.buildExpand(p, gbyCols)
        if err != nil {
            return nil, err
        }
    }
    var aggIndexMap map[int]int
    p, aggIndexMap, err = b.buildAggregation(ctx, p, aggFuncs, gbyCols, correlatedAggMap)
    if err != nil {
        return nil, err
    }
    for agg, idx := range totalMap {
        totalMap[agg] = aggIndexMap[idx]
    }
}
```

**步骤 20**：调用 `PlanBuilder.buildProjection` 函数，根据 `SELECT` 字段列表以及之前步骤中收集的信息，构建投影算子，详见 [构造逻辑执行计划：投影算子](https://github.com/ChangxingJiang/db-notes/blob/main/TiDB%20%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB/%E6%9E%84%E9%80%A0%E9%80%BB%E8%BE%91%E6%89%A7%E8%A1%8C%E8%AE%A1%E5%88%92%EF%BC%9A%E6%8A%95%E5%BD%B1%E7%AE%97%E5%AD%90.md)。

```go
var oldLen int
p, projExprs, oldLen, err = b.buildProjection(ctx, p, sel.Fields.Fields, totalMap, nil, false, sel.OrderBy != nil)
if err != nil {
    return nil, err
}
```

**步骤 21**：如果 `SELECT` 语句中包含 `HAVING` 子句（`sel.Having != nil`），则调用 `PlanBuilder.buildSelection` 函数构建筛选算子，用于过滤聚合后的数据，详见 [构造逻辑执行计划：条件筛选](https://github.com/ChangxingJiang/db-notes/blob/main/TiDB%20%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB/%E6%9E%84%E9%80%A0%E9%80%BB%E8%BE%91%E6%89%A7%E8%A1%8C%E8%AE%A1%E5%88%92%EF%BC%9A%E6%9D%A1%E4%BB%B6%E7%AD%9B%E9%80%89.md)。

```go
if sel.Having != nil {
    b.curClause = havingClause
    p, err = b.buildSelection(ctx, p, sel.Having.Expr, havingMap)
    if err != nil {
        return nil, err
    }
}
```

**步骤 22**：在处理完 `HAVING` 子句后，进行窗口函数的第二阶段处理，构建窗口函数的执行计划，详见 [构造逻辑执行计划：窗口函数](https://github.com/ChangxingJiang/db-notes/blob/main/TiDB%20%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB/%E6%9E%84%E9%80%A0%E9%80%BB%E8%BE%91%E6%89%A7%E8%A1%8C%E8%AE%A1%E5%88%92%EF%BC%9A%E7%AA%97%E5%8F%A3%E5%87%BD%E6%95%B0.md)。

- 调用 `buildWindowSpecs` 函数，处理 SQL 中通过 `WINDOW` 子句定义的窗口规格 (Window Specifications)；
- 如果在 `SELECT` 列表和 `ORDER BY` 子句中包含了窗口函数（`hasWindowFuncField`），或者通过 `WINDOW` 语句定义了窗口（`sel.WindowSpecs != nil`），则执行后续逻辑：
  - 调用 `extractWindowFuncs` 函数，从 `SELECT` 字段列表中提取所有窗口函数；
  - 调用 `PlanBuilder.checkWindowFuncArgs` 函数检查窗口函数的参数是否合法；
  - 调用 `PlanBuilder.groupWindowFuncs` 函数将窗口函数按照其窗口规格分组，以便相同窗口规格的窗口函数可以共用一个窗口算子；
  - 基于分组后的窗口函数和有序的窗口规格，调用 `PlanBuilder.buildWindowFunctions` 构建窗口函数的执行计划；
  - 如果 SQL 确实包含窗口函数字段（`hasWindowFuncField`），而不全是未使用的命名窗口规格，则需要调用 [`PlanBuilder.buildProjection` 函数](https://github.com/ChangxingJiang/db-notes/blob/main/TiDB%20%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB/%E6%9E%84%E9%80%A0%E9%80%BB%E8%BE%91%E6%89%A7%E8%A1%8C%E8%AE%A1%E5%88%92%EF%BC%9A%E6%8A%95%E5%BD%B1%E7%AE%97%E5%AD%90.md) 构建投影算子。

```go
b.windowSpecs, err = buildWindowSpecs(sel.WindowSpecs)
if err != nil {
    return nil, err
}

var windowMapper map[*ast.WindowFuncExpr]int
if hasWindowFuncField || sel.WindowSpecs != nil {
    windowFuncs := extractWindowFuncs(sel.Fields.Fields)
    // we need to check the func args first before we check the window spec
    err := b.checkWindowFuncArgs(ctx, p, windowFuncs, windowAggMap)
    if err != nil {
        return nil, err
    }
    groupedFuncs, orderedSpec, err := b.groupWindowFuncs(windowFuncs)
    if err != nil {
        return nil, err
    }
    p, windowMapper, err = b.buildWindowFunctions(ctx, p, groupedFuncs, orderedSpec, windowAggMap)
    if err != nil {
        return nil, err
    }
    // `hasWindowFuncField == false` means there's only unused named window specs without window functions.
    // In such case plan `p` is not changed, so we don't have to build another projection.
    if hasWindowFuncField {
        // Now we build the window function fields.
        p, projExprs, oldLen, err = b.buildProjection(ctx, p, sel.Fields.Fields, windowAggMap, windowMapper, true, false)
        if err != nil {
            return nil, err
        }
    }
}
```

**步骤 23**：如果 `SELECT` 语句中包含 `DISTINCT` 关键字（`sel.Distinct`），则会调用 [`PlanBuilder.buildDistinct` 函数](https://github.com/ChangxingJiang/db-notes/blob/main/TiDB%20%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB/%E6%9E%84%E9%80%A0%E9%80%BB%E8%BE%91%E6%89%A7%E8%A1%8C%E8%AE%A1%E5%88%92%EF%BC%9ADISTINCT%20%E7%AE%97%E5%AD%90.md) 构建去重算子，用于去除重复行。

```go
if sel.Distinct {
    p, err = b.buildDistinct(p, oldLen)
    if err != nil {
        return nil, err
    }
}
```

**步骤 24**：如果 `SELECT` 语句中包含 `ORDER BY` 子句（`sel.OrderBy != nil`），则会根据需要判断是否需要调用 `PlanBuilder.buildSortWithCheck` 或 `PlanBuilder.buildSort` 构建排序算子，详见 [构造逻辑执行计划：排序算子](https://github.com/ChangxingJiang/db-notes/blob/main/TiDB%20%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB/%E6%9E%84%E9%80%A0%E9%80%BB%E8%BE%91%E6%89%A7%E8%A1%8C%E8%AE%A1%E5%88%92%EF%BC%9A%E6%8E%92%E5%BA%8F%E7%AE%97%E5%AD%90.md)。

- 如果正在构建公共表表达式（CTE）（`b.buildingCTE`），则会将其标记为递归禁止的操作符；
- 判断 `ORDER BY` 子句是否需要保留；
- 在需要保留 `ORDER BY` 的情况下，构造逻辑排序算子。

```go
if sel.OrderBy != nil {
    // flag it if cte contain order by
    if b.buildingCTE {
        b.outerCTEs[len(b.outerCTEs)-1].containRecursiveForbiddenOperator = true
    }
    // We need to keep the ORDER BY clause for the following cases:
    // 1. The select is top level query, order should be honored
    // 2. The query has LIMIT clause
    // 3. The control flag requires keeping ORDER BY explicitly
    if len(b.qbOffset) == 1 || sel.Limit != nil || !b.ctx.GetSessionVars().RemoveOrderbyInSubquery {
        if b.ctx.GetSessionVars().SQLMode.HasOnlyFullGroupBy() {
            p, err = b.buildSortWithCheck(ctx, p, sel.OrderBy.Items, orderMap, windowMapper, projExprs, oldLen, sel.Distinct)
        } else {
            p, err = b.buildSort(ctx, p, sel.OrderBy.Items, orderMap, windowMapper)
        }
        if err != nil {
            return nil, err
        }
    }
}
```

**步骤 25**：如果 `SELECT` 语句中包含 `LIMIT` 子句（`sel.Limit != nil`），则调用 [`buildLimit` 函数](https://github.com/ChangxingJiang/db-notes/blob/main/TiDB%20%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB/%E6%9E%84%E9%80%A0%E9%80%BB%E8%BE%91%E6%89%A7%E8%A1%8C%E8%AE%A1%E5%88%92%EF%BC%9ALIMIT%20%E7%AE%97%E5%AD%90.md) 构建限制算子，用于限制结果集的大小。

```go
if sel.Limit != nil {
    p, err = b.buildLimit(p, sel.Limit)
    if err != nil {
        return nil, err
    }
}
```

**步骤 26**：将 AST 中的 `SELECT` 字段恢复为原始字段列表。

```go
sel.Fields.Fields = originalFields
```

**步骤 27**：如果输出列数发生变化（`oldLen != p.Schema().Len()`），则构建一个新的 [`LogicalProjection` 逻辑算子](https://github.com/ChangxingJiang/db-notes/blob/main/TiDB%20%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB/%E9%80%BB%E8%BE%91%E6%89%A7%E8%A1%8C%E8%AE%A1%E5%88%92%E8%8A%82%E7%82%B9.md#logicalprojection%E6%8A%95%E5%BD%B1%E6%93%8D%E4%BD%9C%E9%80%BB%E8%BE%91%E8%AE%A1%E5%88%92)（投影算子），并将原始算子添加为新投影算子的子节点，以确保输出列数与预期一致。

```go
if oldLen != p.Schema().Len() {
    proj := logicalop.LogicalProjection{Exprs: expression.Column2Exprs(p.Schema().Columns[:oldLen])}.Init(b.ctx, b.getSelectOffset())
    proj.SetChildren(p)
    schema := expression.NewSchema(p.Schema().Clone().Columns[:oldLen]...)
    for _, col := range schema.Columns {
        col.UniqueID = b.ctx.GetSessionVars().AllocPlanColumnID()
    }
    proj.SetOutputNames(p.OutputNames()[:oldLen])
    proj.SetSchema(schema)
    p = proj
}
```

**步骤 28**：调用 `PlanBuilder.tryToBuildSequence` 函数，构造 [`LogicalSequence` 逻辑算子](https://github.com/ChangxingJiang/db-notes/blob/main/TiDB%20%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB/%E9%80%BB%E8%BE%91%E6%89%A7%E8%A1%8C%E8%AE%A1%E5%88%92%E8%8A%82%E7%82%B9.md#logicalsequence%E5%BA%8F%E5%88%97%E9%80%BB%E8%BE%91%E8%AE%A1%E5%88%92)，将 `WITH` 语句中的 CTE 和原始查询计划合并为一个序列。

```go
return b.tryToBuildSequence(currentLayerCTEs, p), nil
```

## `tryToBuildSequence` 函数

`tryToBuildSequence` 函数用于尝试构建一个逻辑序列算子（LogicalSequence），它将 CTE（公共表表达式）和原始查询计划组合成一个序列执行。该函数位于 `pkg/planner/core/logical_plan_builder.go` 文件中，函数签名如下：

```go
func (b *PlanBuilder) tryToBuildSequence(ctes []*cteInfo, p base.LogicalPlan) base.LogicalPlan
```

**步骤 1**：检查会话变量 `EnableMPPSharedCTEExecution` 是否启用，如果没有启用，则直接返回原始逻辑算子。

```go
if !b.ctx.GetSessionVars().EnableMPPSharedCTEExecution {
    return p
}
```

**步骤 2**：遍历 CTE 信息列表，如果发现任何递归 CTE（`!ctes[i].nonRecursive`），则直接返回原始计划，因为序列算子只支持非递归 CTE。同时，检查每个 CTE 是否是内联（`ctes[i].isInline`）的或没有 CTE 类（`ctes[i].cteClass == nil`），如果是，则从列表中删除这个 CTE。

```go
for i := len(ctes) - 1; i >= 0; i-- {
    if !ctes[i].nonRecursive {
        return p
    }
    if ctes[i].isInline || ctes[i].cteClass == nil {
        ctes = slices.Delete(ctes, i, i+1)
    }
}
```

**步骤 3**：如果没有需要处理的 CTE，则直接返回原始逻辑算子。

```go
if len(ctes) == 0 {
    return p
}
```

**步骤 4**：为每个需要处理的 CTE 创建一个 [`LogicalCTE` 逻辑算子](https://github.com/ChangxingJiang/db-notes/blob/main/TiDB%20%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB/%E9%80%BB%E8%BE%91%E6%89%A7%E8%A1%8C%E8%AE%A1%E5%88%92%E8%8A%82%E7%82%B9.md#logicalctecte-%E9%80%BB%E8%BE%91%E8%AE%A1%E5%88%92)，这些算子将作为序列算子的子节点。每个 LogicalCTE 算子包含 CTE 类、CTE 名称、种子统计信息等，并标记为仅用作存储。

```go
lctes := make([]base.LogicalPlan, 0, len(ctes)+1)
for _, cte := range ctes {
    lcte := logicalop.LogicalCTE{
        Cte:               cte.cteClass,
        CteAsName:         cte.def.Name,
        CteName:           cte.def.Name,
        SeedStat:          cte.seedStat,
        OnlyUsedAsStorage: true,
    }.Init(b.ctx, b.getSelectOffset())
    lcte.SetSchema(getResultCTESchema(cte.seedLP.Schema(), b.ctx.GetSessionVars()))
    lctes = append(lctes, lcte)
}
```

**步骤 5**：设置优化标记

```go
b.optFlag |= rule.FlagPushDownSequence
```

**步骤 6**：创建 [`LogicalSequence` 逻辑算子](https://github.com/ChangxingJiang/db-notes/blob/main/TiDB%20%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB/%E9%80%BB%E8%BE%91%E6%89%A7%E8%A1%8C%E8%AE%A1%E5%88%92%E8%8A%82%E7%82%B9.md#logicalsequence%E5%BA%8F%E5%88%97%E9%80%BB%E8%BE%91%E8%AE%A1%E5%88%92)，将所有 CTE 算子和原始逻辑算子作为其子节点。序列算子的输出名称与原始逻辑算子相同。

```go
seq := logicalop.LogicalSequence{}.Init(b.ctx, b.getSelectOffset())
seq.SetChildren(append(lctes, p)...)
seq.SetOutputNames(p.OutputNames().Shallow())
return seq
```
