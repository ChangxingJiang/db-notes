# TiDB 源码阅读｜逻辑执行计划的构造方法

逻辑执行计划是 SQL 查询优化的第一步，它将 SQL 语句的抽象语法树（AST）转换为结构化的逻辑算子树，为后续的优化和物理计划生成奠定基础。本文将详细介绍 TiDB 中逻辑执行计划的通用构造过程，并以 `SELECT` 语句为例介绍 TiDB 中逻辑执行计划的具体构造过程。

## 通用逻辑执行计划的构造流程

在 TiDB 中，逻辑执行计划的构造是从 `Optimize` 函数开始的，该函数位于 `pkg/planner/optimize.go` 文件中。它接收一个 AST 节点，然后通过一系列步骤将其转换为逻辑执行计划。

### `buildLogicalPlan` 函数流程

`buildLogicalPlan` 函数是构造逻辑计划的核心函数，它位于 `pkg/planner/optimize.go` 文件中。该函数按照以下步骤构建逻辑计划：

**步骤 1**：初始化准备

这一步骤为逻辑计划的构建准备了干净的环境，确保每次构建都从一个确定的状态开始。计划 ID 和列 ID 是为了给计划中的算子和列分配唯一标识符，而清除子查询映射和扩展列映射则是为了避免之前查询的信息干扰当前查询的处理。重置重写阶段信息则是为了记录后续重写过程的性能指标。

```go
// 初始化计划 ID 和列 ID
sctx.GetSessionVars().PlanID.Store(0)
sctx.GetSessionVars().PlanColumnID.Store(0)

// 清除子查询映射和扩展列映射
sctx.GetSessionVars().MapScalarSubQ = nil
sctx.GetSessionVars().MapHashCode2UniqueID4ExtendedCol = nil

// 重置重写阶段信息
sctx.GetSessionVars().RewritePhaseInfo.Reset()
```

**步骤 2**：调用 `builder.Build` 构建逻辑计划并记录重写阶段耗时

这是整个函数的核心步骤，调用 `PlanBuilder.Build` 方法将 AST 节点转换为逻辑计划树。同时，函数记录了构建过程的耗时，这对于后续的性能分析和优化非常有价值。`PlanBuilder.Build` 会根据 AST 节点的类型（如 SELECT、INSERT、UPDATE 等）调用相应的构建函数，生成对应的逻辑算子树。如果在构建过程中遇到错误，函数会立即返回错误信息，终止后续处理。

```go
beginRewrite := time.Now()
p, err := builder.Build(ctx, node)
if err != nil {
    return nil, err
}
sctx.GetSessionVars().RewritePhaseInfo.DurationRewrite = time.Since(beginRewrite)
```

**步骤 3**：收集表信息

在逻辑计划构建完成后，需要收集查询涉及的表信息，这些信息对于后续的权限检查、统计信息收集和执行计划优化都很重要。函数根据计划的类型选择不同的方式收集表信息：如果是 `Execute` 类型的计划（执行预编译语句），则从预编译语句中获取表信息；否则，从构建器的访问信息中获取表信息。这样可以确保后续处理阶段能够准确了解查询涉及的数据表。

```go
if exec, ok := p.(*core.Execute); ok && exec.PrepStmt != nil {
    sctx.GetSessionVars().StmtCtx.Tables = core.GetDBTableInfo(exec.PrepStmt.VisitInfos)
} else {
    sctx.GetSessionVars().StmtCtx.Tables = core.GetDBTableInfo(builder.GetVisitInfo())
}
```

**步骤 4**：返回构建好的逻辑计划

最后，函数返回构建好的逻辑计划和可能的错误。这个逻辑计划将作为后续优化阶段的输入，经过一系列的优化规则处理后，最终转换为物理执行计划。整个 `buildLogicalPlan` 函数的设计体现了 TiDB 查询优化器的模块化思想，将复杂的查询处理过程分解为清晰的步骤，便于维护和扩展。

```go
return p, nil
```

### `PlanBuilder.Build` 函数流程

`PlanBuilder.Build` 函数是实际构建逻辑计划的入口，它位于 `pkg/planner/core/planbuilder.go` 文件中。该函数根据不同的 AST 节点类型，调用相应的构建函数，构造逻辑执行计划。

**步骤 1**：设置解析上下文

在这一步骤中，函数将解析上下文（`resolveCtx`）设置为传入节点的解析上下文。解析上下文包含了名称解析所需的各种信息，如数据库、表和列的引用关系，这对于正确构建逻辑计划至关重要。通过保存这个上下文，`PlanBuilder` 可以在后续构建过程中随时访问这些信息，确保名称引用的正确性和一致性。

```go
b.resolveCtx = node.GetResolveContext()
```

**步骤 2**：设置优化标志

函数设置了列剪裁（Column Pruning）的优化标志。列剪裁是一种重要的优化技术，它只保留查询中实际需要的列，减少不必要的数据读取和处理。通过在构建逻辑计划的早期阶段就设置这个标志，可以确保后续的优化过程能够充分利用这一技术，提高查询效率。

```go
b.optFlag |= rule.FlagPruneColumns
```

**步骤 3**：根据 AST 节点类型，调用相应的构建函数，构造逻辑执行计划

这是函数的核心部分，它通过一个大型的 `switch` 语句，根据 AST 节点的具体类型，调用相应的专门构建函数。

```go
switch x := node.Node.(type) {
case *ast.AdminStmt:
    return b.buildAdmin(ctx, x)
// ... 其他语句类型
case *ast.InsertStmt:
    return b.buildInsert(ctx, x)
// ... 其他语句类型
case *ast.SelectStmt:
    if x.SelectIntoOpt != nil {
        return b.buildSelectInto(ctx, x)
    }
    return b.buildSelect(ctx, x)
// ... 其他语句类型
}
```

**步骤 4**：对于不支持的节点类型，返回错误

```go
return nil, plannererrors.ErrUnsupportedType.GenWithStack("Unsupported type %T", node.Node)
```

## `SELECT` 语句逻辑执行计划的构造流程

`SELECT` 语句是 SQL 中最常用的语句之一，其逻辑执行计划的构造过程比较复杂。`PlanBuilder.buildSelect` 函数负责构建 `SELECT` 语句的逻辑执行计划，它位于 `pkg/planner/core/logical_plan_builder.go` 文件中。下面详细介绍该函数的主要步骤：

**步骤 1**：初始化准备

在这一步骤中，函数首先设置查询块偏移量和表提示信息，这些信息对于后续的查询处理非常重要。同时，函数会检查递归 CTE（公共表表达式）的相关限制，确保在递归 CTE 中不使用不支持的操作，如 `DISTINCT`、`ORDER BY` 或 `LIMIT` 等。此外，如果 SQL 语句中包含 `STRAIGHT_JOIN` 提示，函数会设置直接连接标志，指示优化器按照 FROM 子句中表的顺序进行连接，而不是自动选择最优的连接顺序。

> CTE（Common Table Expression，公共表表达式）是一种命名的临时结果集，可以在单个 SQL 语句中多次引用。在 TiDB 中，CTE 通过 `WITH` 子句定义，允许用户将复杂查询分解为更简单的部分。CTE 有两种类型：非递归 CTE 和递归 CTE。非递归 CTE 是一个普通的子查询，而递归 CTE 则可以引用自身，适用于处理层次化数据。递归 CTE 有特定的语法限制，例如不允许在递归部分使用 `DISTINCT`、`GROUP BY`、聚合函数、`ORDER BY` 或 `LIMIT` 等，这是为了确保递归过程能够正确终止。当系统检测到这些限制被违反时，会在此步骤中报错，确保查询的正确执行。

```go
b.pushSelectOffset(sel.QueryBlockOffset)
b.pushTableHints(sel.TableHints, sel.QueryBlockOffset)
defer func() {
    b.popSelectOffset()
    // table hints are only visible in the current SELECT statement.
    b.popTableHints()
}()
if b.buildingRecursivePartForCTE {
    if sel.Distinct || sel.OrderBy != nil || sel.Limit != nil {
        return nil, plannererrors.ErrNotSupportedYet.GenWithStackByArgs("ORDER BY / LIMIT / SELECT DISTINCT in recursive query block of Common Table Expression")
    }
    if sel.GroupBy != nil {
        return nil, plannererrors.ErrCTERecursiveForbidsAggregation.FastGenByArgs(b.genCTETableNameForError())
    }
}
if sel.SelectStmtOpts != nil {
    origin := b.inStraightJoin
    b.inStraightJoin = sel.SelectStmtOpts.StraightJoin
    defer func() { b.inStraightJoin = origin }()
}
```

**步骤 2**：处理 `WITH` 子句（公共表表达式）

如果 `SELECT` 语句中包含 `WITH` 子句，则调用 `buildWith` 函数构建公共表表达式（CTE）。`WITH` 子句允许在查询中定义临时命名结果集，这些结果集可以在查询的其他部分引用，从而提高查询的可读性和可维护性。构建 CTE 的过程包括解析 CTE 定义、构建 CTE 的逻辑计划，并将其注册到 CTE 上下文中，以便后续在查询中引用。【TODO: `buildWith` 函数】

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

**步骤 3**：构建表引用（FROM 子句）

这是 SELECT 语句处理的核心步骤之一，调用 `buildTableRefs` 函数构建 `FROM` 子句中的表引用。对于单表查询，会创建 `DataSource` 算子；对于多表查询，会创建 `Join` 算子。这一步骤将 SQL 语句中的表引用转换为逻辑算子，为后续的查询处理奠定基础。【TODO: `buildTableRefs` 函数】

```go
p, err = b.buildTableRefs(ctx, sel.From)
if err != nil {
    return nil, err
}
```

**步骤 4**：处理通配符和别名

调用 `unfoldWildStar` 函数展开选择列表中的通配符（如 `*`），将其转换为具体的列引用。同时，如果需要，函数会为选择字段添加别名，这对于视图创建等场景特别重要，因为视图需要为每个输出列指定名称。【TODO: `unfoldWildStar` 函数】

```go
originalFields := sel.Fields.Fields
sel.Fields.Fields, err = b.unfoldWildStar(p, sel.Fields.Fields)
if err != nil {
    return nil, err
}
if b.capFlag&canExpandAST != 0 {
    // To be compatible with MySQL, we add alias name for each select field when creating view.
    sel.Fields.Fields, err = b.addAliasName(ctx, sel, p)
    if err != nil {
        return nil, err
    }
    originalFields = sel.Fields.Fields
}
```

**步骤 5**：处理 `GROUP BY` 子句

如果 `SELECT` 语句中包含 `GROUP BY` 子句，调用 `resolveGbyExprs` 函数解析 `GROUP BY` 表达式，并为后续构建聚合算子做准备。在 `ONLY_FULL_GROUP_BY` 模式下，函数还会检查 `GROUP BY` 的合法性，确保非聚合列都出现在 `GROUP BY` 子句中。【TODO: `resolveGbyExprs` 函数】

```go
if sel.GroupBy != nil {
    p, gbyCols, rollup, err = b.resolveGbyExprs(ctx, p, sel.GroupBy, sel.Fields.Fields)
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
```

**步骤 6**：处理窗口函数

如果 `SELECT` 语句中包含窗口函数，则会调用 `resolveWindowFunction` 函数检测 `SELECT` 字段中的窗口函数，并解析窗口函数规范。窗口函数允许在查询结果的行子集上执行计算，如排名、累计求和等。处理窗口函数的过程包括检查函数参数、分组窗口函数、构建窗口函数算子等。【TODO: `resolveWindowFunction` 函数】

```go
hasWindowFuncField := b.detectSelectWindow(sel)
// Some SQL statements define WINDOW but do not use them. But we also need to check the window specification list.
// For example: select id from t group by id WINDOW w AS (ORDER BY uids DESC) ORDER BY id;
// We don't use the WINDOW w, but if the 'uids' column is not in the table t, we still need to report an error.
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

**步骤 7**：解析 `HAVING` 和 `ORDER BY` 子句

调用 `resolveHavingAndOrderBy` 函数解析 `HAVING` 和 `ORDER BY` 子句中的表达式，并处理其中的聚合函数。这一步骤确保在构建投影之前正确处理 `HAVING` 和 `ORDER BY` 子句中的表达式，特别是当查询包含 `select a+1 as b from t having sum(b) < 0` 这样的语句时，需要将 `sum(b)` 替换为 `sum(a+1)`。【TODO: `resolveHavingAndOrderBy` 函数】

```go
// We must resolve having and order by clause before build projection,
// because when the query is "select a+1 as b from t having sum(b) < 0", we must replace sum(b) to sum(a+1),
// which only can be done before building projection and extracting Agg functions.
havingMap, orderMap, err = b.resolveHavingAndOrderBy(ctx, sel, p)
if err != nil {
    return nil, err
}
```

**步骤 8**：处理相关聚合函数

调用 `resolveCorrelatedAggregates` 函数处理子查询中的相关聚合函数。在构建聚合和投影之前，需要解析子查询中的相关聚合函数，例如，`select (select count(a)) from t` 中的 `count(a)` 应该在外部查询的上下文中计算。因此，需要从子查询中提取这些聚合函数，并将它们放入 `SELECT` 字段列表中。【TODO: `resolveCorrelatedAggregates` 函数】

```go
// We have to resolve correlated aggregate inside sub-queries before building aggregation and building projection,
// for instance, count(a) inside the sub-query of "select (select count(a)) from t" should be evaluated within
// the context of the outer query. So we have to extract such aggregates from sub-queries and put them into
// SELECT field list.
correlatedAggMap, err = b.resolveCorrelatedAggregates(ctx, sel, p)
if err != nil {
    return nil, err
}
```

**步骤 9**：构建 `WHERE` 子句

如果 `SELECT` 语句中包含 `WHERE` 子句，则调用 `buildSelection` 函数构建选择算子，用于过滤数据。【TODO: `buildSelection` 函数】

```go
if sel.Where != nil {
    p, err = b.buildSelection(ctx, p, sel.Where, nil)
    if err != nil {
        return nil, err
    }
}
```

**步骤 10**：处理锁信息

如果 `SELECT` 语句中包含锁信息（如 `FOR UPDATE`），则会构建锁算子，用于在事务中锁定相关行。【TODO: 构造锁算子】

```go
l := sel.LockInfo
if l != nil && l.LockType != ast.SelectLockNone {
    var tableList []*ast.TableName
    var isExplicitSetTablesNames bool
    if len(l.Tables) == 0 && sel.From != nil {
        // It's for stmt like `select xxx from table t, t1 where t.a = t1.a for update`
        nodeW := resolve.NewNodeWWithCtx(sel.From, b.resolveCtx)
        tableList = ExtractTableList(nodeW, false)
    } else if len(l.Tables) != 0 {
        // It's for stmt like `select xxx from table t, t1 where t.a = t1.a for update of t`
        isExplicitSetTablesNames = true
        tableList = l.Tables
    }
    for _, tName := range tableList {
        // CTE has no *model.HintedTable, we need to skip it.
        tNameW := b.resolveCtx.GetTableName(tName)
        if tNameW == nil {
            continue
        }
        if isExplicitSetTablesNames {
            // If `LockTableIDs` map is empty, it will lock all records from all tables.
            // Besides, it will only lock the metioned in `of` part.
            b.ctx.GetSessionVars().StmtCtx.LockTableIDs[tNameW.TableInfo.ID] = struct{}{}
        }
        dbName := tName.Schema.L
        if dbName == "" {
            dbName = b.ctx.GetSessionVars().CurrentDB
        }
        var authErr error
        if user := b.ctx.GetSessionVars().User; user != nil {
            authErr = plannererrors.ErrTableaccessDenied.GenWithStackByArgs("SELECT with locking clause", user.AuthUsername, user.AuthHostname, tNameW.Name.L)
        }
        b.visitInfo = appendVisitInfo(b.visitInfo, mysql.DeletePriv|mysql.UpdatePriv|mysql.LockTablesPriv, dbName, tNameW.Name.L, "", authErr)
    }
    p, err = b.buildSelectLock(p, l)
    if err != nil {
        return nil, err
    }
}
```

**步骤 11**：处理聚合函数

如果 `SELECT` 字段中包含聚合函数，则会构建聚合算子。如果存在 `ROLLUP` 语法，还需要构建 Expand 算子，用于复制数据以满足不同的分组布局。

在这一步骤中，系统会根据需要调用三个关键函数：
1. `extractAggFuncsInSelectFields` 函数：当检测到 `SELECT` 字段中包含聚合函数（如 `SUM`、`COUNT`、`AVG` 等）时调用，用于从 SELECT 字段列表中提取所有聚合函数表达式。该函数会遍历 SELECT 字段，识别其中的聚合函数，并将它们存储在 `aggFuncs` 列表中，同时建立聚合函数到索引的映射关系（`totalMap`），以便后续在投影算子中引用这些聚合函数的结果。【TODO: `extractAggFuncsInSelectFields` 函数】
2. `buildExpand` 函数：仅当查询中使用了 `ROLLUP` 语法时调用。`ROLLUP` 是一种特殊的分组操作，它会生成多层次的聚合结果。为了支持这一操作，系统需要构建 Expand 算子，该算子会复制输入数据并为不同层次的聚合添加标记，使得后续的聚合算子能够按照不同的分组方式处理数据。【TODO: `buildExpand` 函数】
3. `buildAggregation` 函数：当确定需要构建聚合操作时调用，它接收聚合函数列表、分组列和相关聚合映射作为输入，根据这些信息构建聚合算子。该函数会生成一个将原始数据分组并应用聚合函数的逻辑计划节点，同时返回一个映射表，用于将原始聚合函数索引映射到聚合算子输出列的索引。【TODO: `buildAggregation` 函数】

```go
hasAgg := b.detectSelectAgg(sel)
needBuildAgg := hasAgg
if hasAgg {
    if b.buildingRecursivePartForCTE {
        return nil, plannererrors.ErrCTERecursiveForbidsAggregation.GenWithStackByArgs(b.genCTETableNameForError())
    }

    aggFuncs, totalMap = b.extractAggFuncsInSelectFields(sel.Fields.Fields)
    // len(aggFuncs) == 0 and sel.GroupBy == nil indicates that all the aggregate functions inside the SELECT fields
    // are actually correlated aggregates from the outer query, which have already been built in the outer query.
    // The only thing we need to do is to find them from b.correlatedAggMap in buildProjection.
    if len(aggFuncs) == 0 && sel.GroupBy == nil {
        needBuildAgg = false
    }
}
if needBuildAgg {
    // if rollup syntax is specified, Expand OP is required to replicate the data to feed different grouping layout.
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

**步骤 12**：构建投影

函数调用 `buildProjection` 函数构建投影算子，处理 SELECT 字段列表。投影算子用于选择、重命名和计算列，是 SELECT 语句处理的核心部分。【TODO：`buildProjection` 函数】

> 投影（Projection）是查询处理中的一个重要操作，它负责从输入数据中选择、转换和计算特定的列，生成查询结果的列集合。在 TiDB 中，投影操作通过 `LogicalProjection` 算子实现，该算子接收输入数据并根据 `SELECT` 子句中指定的表达式计算输出列。投影操作可以执行多种转换，包括列选择（如 `SELECT a, b FROM t`）、列重命名（如 `SELECT a AS x FROM t`）、表达式计算（如 `SELECT a+b AS sum FROM t`）以及函数应用（如 `SELECT CONCAT(first_name, ' ', last_name) AS full_name FROM users`）。
>
> 在执行计划中，投影算子通常位于数据源算子（如表扫描）之后，或者位于聚合、排序等其他算子之后，用于处理这些算子的输出结果。投影是一个纯计算操作，不涉及数据的重新排序或分组，因此其时间复杂度为 O(n)，其中 n 是输入行数。在列存储系统中，投影操作特别高效，因为它只需要读取和处理查询中实际涉及的列，而不是整行数据。
>
> 在步骤 12 中，`buildProjection` 函数根据 `SELECT` 字段列表和之前步骤中收集的信息（如聚合函数映射），构建投影表达式，创建 `LogicalProjection` 算子，并将其连接到计划树中。这个函数还会返回投影表达式列表和原始长度，这些信息将在后续步骤中用于处理 `HAVING` 子句、窗口函数和 `DISTINCT` 操作。

```go
var oldLen int
// According to https://dev.mysql.com/doc/refman/8.0/en/window-functions-usage.html,
// we can only process window functions after having clause, so `considerWindow` is false now.
p, projExprs, oldLen, err = b.buildProjection(ctx, p, sel.Fields.Fields, totalMap, nil, false, sel.OrderBy != nil)
if err != nil {
    return nil, err
}
```

**步骤 13**：处理 `HAVING` 子句

如果 `SELECT` 语句中包含 `HAVING` 子句，函数会构建选择算子，用于过滤聚合结果。【TODO：`buildSelection` 函数】

```go
    if sel.Having != nil {
        b.curClause = havingClause
        p, err = b.buildSelection(ctx, p, sel.Having.Expr, havingMap)
        if err != nil {
            return nil, err
        }
    }
```

**步骤 14**：处理窗口函数

如果 `SELECT` 字段中包含窗口函数，函数会构建窗口函数算子，并构建新的投影算子。

通过 `buildWindowSpecs` 函数以构造窗口，该函数解析窗口规范，包括分区（PARTITION BY）、排序（ORDER BY）和窗口帧（如 ROWS BETWEEN）等信息。窗口规范可以被命名并在多个窗口函数之间共享，这提高了查询的可读性和可维护性。系统会验证窗口规范的有效性，例如检查引用的列是否存在，排序表达式是否有效等。构造好的窗口规范将被存储在 `windowSpecs` 中，供后续窗口函数使用。【TODO：`buildWindowSpecs` 函数】

构造窗口函数的过程如下：

1. 通过 `extractWindowFuncs` 函数从 `SELECT` 字段列表中提取所有窗口函数表达式【TODO：`extractWindowFuncs` 函数】
2. 通过 `checkWindowFuncArgs` 函数检查窗口函数的参数是否有效【TODO：`checkWindowFuncArgs` 函数】
3. 通过 `groupWindowFuncs` 函数将具有相同窗口规范的窗口函数分组，以优化执行【TODO：`groupWindowFuncs` 函数】
4. 通过 `buildWindowFunctions` 函数为每组窗口函数构建窗口算子。这些算子负责根据窗口规范对数据进行分区、排序，并计算窗口函数的结果。窗口函数的计算结果通过 `windowMapper` 映射到新的投影表达式中，使得这些结果可以被输出或在后续处理中使用。【TODO：`buildWindowFunctions` 函数】

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

**步骤 15**：处理 `DISTINCT`

如果 `SELECT` 语句中包含 `DISTINCT` 关键字，函数会调用 `buildDistinct` 函数构建去重算子，用于去除重复行。【TODO：`buildDistinct` 函数】

```go
if sel.Distinct {
    p, err = b.buildDistinct(p, oldLen)
    if err != nil {
        return nil, err
    }
}
```

**步骤 16**：处理 `ORDER BY` 子句

如果 `SELECT` 语句中包含 `ORDER BY` 子句，函数会构建排序算子，用于对结果集进行排序。在某些情况下，可以省略 `ORDER BY` 子句，如子查询中的 `ORDER BY`（除非子查询包含 `LIMIT` 子句）。【TODO：`buildSortWithCheck` 函数和 `buildSort` 函数】

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

**步骤 17**：处理 `LIMIT` 子句

如果 `SELECT` 语句中包含 `LIMIT` 子句，函数会调用 `buildLimit` 函数构建限制算子，用于限制结果集的大小。【TODO：`buildLimit` 函数】

```go
if sel.Limit != nil {
    p, err = b.buildLimit(p, sel.Limit)
    if err != nil {
        return nil, err
    }
}
```

**步骤 18**：最终处理和返回

函数恢复原始字段列表，并根据需要调整输出字段。如果输出列数发生变化，函数会构建一个新的投影算子，以确保输出列数与预期一致。最后，函数返回构建好的逻辑执行计划。【TODO：`tryToBuildSequence` 函数】

```go
sel.Fields.Fields = originalFields
if oldLen != p.Schema().Len() {
    proj := logicalop.LogicalProjection{Exprs: expression.Column2Exprs(p.Schema().Columns[:oldLen])}.Init(b.ctx, b.getSelectOffset())
    proj.SetChildren(p)
    schema := expression.NewSchema(p.Schema().Clone().Columns[:oldLen]...)
    for _, col := range schema.Columns {
        col.UniqueID = b.ctx.GetSessionVars().AllocPlanColumnID()
    }
    proj.SetOutputNames(p.OutputNames()[:oldLen])
    proj.SetSchema(schema)
    return b.tryToBuildSequence(currentLayerCTEs, proj), nil
}

return b.tryToBuildSequence(currentLayerCTEs, p), nil
```