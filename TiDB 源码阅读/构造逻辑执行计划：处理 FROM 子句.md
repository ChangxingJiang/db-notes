# TiDB 源码阅读｜构造逻辑执行计划：处理 FROM 子句

在 SQL 查询语句中，FROM 子句用于指定查询的数据源，可以是表、视图、子查询或者这些的组合（通过 JOIN 连接）。本文将分析 TiDB 处理 FROM 子句的相关代码逻辑，主要关注 `buildTableRefs`、`buildTableDual` 和 `buildResultSetNode` 三个函数的实现。

## `buildTableRefs` 函数处理流程

【TODO】待补充输出算子结构

`buildTableRefs` 函数定义在 `pkg/planner/core/logical_plan_builder.go` 文件中，负责构建 SQL 语句中 FROM 子句的逻辑执行计划。该函数接收一个 `TableRefsClause` 类型的参数，返回对应的逻辑执行计划。

函数签名如下：

```go
func (b *PlanBuilder) buildTableRefs(ctx context.Context, from *ast.TableRefsClause) (p base.LogicalPlan, err error)
```

**步骤 1**：处理空 `FROM` 子句情况

如果没有 `FROM` 子句（即 `from == nil`），则调用 `buildTableDual` 函数创建一个 `LogicalTableDual` 算子作为虚拟数据源。

```go
if from == nil {
    p = b.buildTableDual()
    return
}
```

**步骤 2**：添加 defer 函数重置 CTE 引用状态

通过 defer 函数，确保在构建完成后重置所有外部 CTE 的递归引用标志 `recursiveRef`。

```go
defer func() {
    // After build the resultSetNode, need to reset it so that it can be referenced by outer level.
    for _, cte := range b.outerCTEs {
        cte.recursiveRef = false
    }
}()
```

**步骤 3**：调用 `buildResultSetNode` 函数构建结果集节点

调用 `buildResultSetNode` 函数处理 `FROM` 子句，该函数会根据不同的节点类型（表、子查询、JOIN 等）构建相应的逻辑计划。传递 `false` 表示当前构建的不是 CTE（公共表表达式）查询。

```go
return b.buildResultSetNode(ctx, from.TableRefs, false)
```

## `buildTableDual` 函数处理流程

输出算子结构：`LogicalTableDual`

`buildTableDual` 函数定义在 `pkg/planner/core/logical_plan_builder.go` 文件中，用于构建一个 `LogicalTableDual` 算子，该算子会生成空虚拟数据行的逻辑执行计划，用于没有 `FROM` 子句的 `SELECT` 语句，或者在某些情况下作为替代数据源。

**步骤 1**：添加一个空的处理器映射

调用 `pushMap(nil)` 向处理器堆栈中添加一个空的映射。

```go
b.handleHelper.pushMap(nil)
```

**步骤 2**：创建并初始化 `LogicalTableDual` 算子

创建一个行数为 1 的 `LogicalTableDual` 逻辑执行计划算子，并使用当前上下文和查询块偏移量进行初始化。`RowCount: 1` 表示该算子会生成一行数据。【TODO：增加 `LogicalTableDual` 算子的文档链接】。

```go
return logicalop.LogicalTableDual{RowCount: 1}.Init(b.ctx, b.getSelectOffset())
```

## `buildResultSetNode` 函数处理流程

【TODO】待补充输出算子结构

`buildResultSetNode` 函数定义在 `pkg/planner/core/logical_plan_builder.go` 文件中，负责根据不同类型的节点构建相应的逻辑计划。函数签名如下：

```go
func (b *PlanBuilder) buildResultSetNode(ctx context.Context, node ast.ResultSetNode, isCTE bool) (p base.LogicalPlan, err error)
```

### 处理 `Join` 类型节点

调用 `PlanBuilder.buildJoin` 函数处理关联的逻辑执行计划。【TODO：`buildJoin` 函数】

```go
return b.buildJoin(ctx, x)
```

### 处理 `TableSource` 类型节点

先根据不同情况执行不同的处理规则：

**情况 1**：处理 `TableSource` 中为子查询（`SelectStmt`）类型节点

先准备 CTE 检查环境，然后调用 `PlanBuilder.buildSelect` 函数构建子查询的逻辑执行计划。【TODO：`buildSelect` 函数链接】

```go
ci := b.prepareCTECheckForSubQuery()
defer resetCTECheckForSubQuery(ci)
b.optFlag = b.optFlag | rule.FlagConstantPropagation
p, err = b.buildSelect(ctx, v)
```

**情况 2**：处理 `TableSource` 中为集合操作（`SetOprStmt`）类型节点

同样先需要准备 CTE 检查环境，然后调用 `PlanBuilder.buildSetOpr` 函数构建集合操作的逻辑执行计划。【TODO：`buildSetOpr` 函数】

```go
ci := b.prepareCTECheckForSubQuery()
defer resetCTECheckForSubQuery(ci)
p, err = b.buildSetOpr(ctx, v)
```

**情况 3**：处理 `TableSource` 中为表名引用（`TableName`）类型节点

直接调用 `PlanBuilder.buildDataSource` 函数构建数据源的逻辑执行计划。【TODO：`buildDataSource` 函数】

```go
p, err = b.buildDataSource(ctx, v, &x.AsName)
isTableName = true
```

在处理完 `TableSource` 内部节点的基础逻辑执行计划后，还需要执行后续处理：

**步骤 1**：如果输出类名没有表别名，则为输出列名设置表别名

```go
for _, name := range p.OutputNames() {
    if name.Hidden {
        continue
    }
    if x.AsName.L != "" {
        name.TblName = x.AsName
    }
}
```

**步骤 2**：根据会话（session）配置信息，指定非表名节点的查询块别名。

```go
var plannerSelectBlockAsName []ast.HintTable
if p := b.ctx.GetSessionVars().PlannerSelectBlockAsName.Load(); p != nil {
    plannerSelectBlockAsName = *p
}
if len(plannerSelectBlockAsName) > 0 && !isTableName {
    plannerSelectBlockAsName[p.QueryBlockOffset()] = ast.HintTable{DBName: p.OutputNames()[0].DBName, TableName: p.OutputNames()[0].TblName}
}
```

**步骤 3**：检查结果集中是否存在重复的列名，如果存在则返回错误信息。

```go
dupNames := make(map[string]struct{}, len(p.Schema().Columns))
for _, name := range p.OutputNames() {
    colName := name.ColName.O
    if _, ok := dupNames[colName]; ok {
        return nil, plannererrors.ErrDupFieldName.GenWithStackByArgs(colName)
    }
    dupNames[colName] = struct{}{}
}
```

### 处理 `SelectStmt` 类型节点

调用 `PlanBuilder.buildSelect` 函数构建 `SELECT` 语句的逻辑执行计划。【TODO：`buildSelect` 函数链接】

```go
return b.buildSelect(ctx, x)
```

### 处理 `SetOprStmt` 类型节点

调用 `PlanBuilder.buildSetOpr` 函数构建集合操作（如 `UNION`、`INTERSECT` 等）的逻辑执行计划。【TODO：`buildSetOpr` 函数】

```go
return b.buildSetOpr(ctx, x)
```

### 处理未支持的其他类型节点

如果遇到未支持的节点类型，则返回错误。

```go
return nil, plannererrors.ErrUnsupportedType.GenWithStack("Unsupported ast.ResultSetNode(%T) for buildResultSetNode()", x)
```

## `buildDataSource` 函数处理流程

`buildDataSource` 函数定义在 `pkg/planner/core/logical_plan_builder.go` 文件中，负责构建表数据源的逻辑执行计划。该函数接收表名、数据库名和表别名作为参数，返回对应的逻辑执行计划。

函数签名如下：

```go
func (b *PlanBuilder) buildDataSource(ctx context.Context, tn *ast.TableName, asName *ast.CIStr) (base.LogicalPlan, error)
```

输出算子结构：

- 场景 1：【TODO】`PlanBuilder.tryBuildCTE` 函数返回值
- 场景 2：【TODO】`PlanBuilder.buildMemTable` 函数返回值
- 场景 3：【TODO】`PlanBuilder.BuildDataSourceFromView` 函数返回值
- 场景 4：`LogicalTableDual`（当前表为空序列）
- 场景 5：`LogicalUnionScan -> DataSource`（当前表存在脏更新）
- 场景 6：`DataSource`（其他情况）

**步骤 1**：处理数据库名

如果数据库名为空，则尝试从 CTE（公共表表达式）构建数据源，若无法从 CTE 构建，则使用当前会话的默认数据库名。【TODO：`tryBuildCTE` 函数】

```go
b.optFlag |= rule.FlagPredicateSimplification  // 设置谓词简化优化标志
dbName := tn.Schema
sessionVars := b.ctx.GetSessionVars()

if dbName.L == "" {
    p, err := b.tryBuildCTE(ctx, tn, asName)  // 尝试从 CTE 构建
    if err != nil || p != nil {
        return p, err
    }
    dbName = ast.NewCIStr(sessionVars.CurrentDB)
}
```

**步骤 2**：获取信息 Schema 并处理视图中的表引用

对于视图中的表，始终忽略本地临时表，例如：如果用户创建了一个普通表 `t1` 和一个引用 `t1` 的视图 `v1`，之后又创建了一个同名的本地临时表 `t1`；此时执行 `select * from v1` 时，仍然应该返回普通表 `t1` 中的所有记录，而不是临时表 `t1` 中的数据。

```go
is := b.is  // 从 PlanBuilder 实例中获取 InfoSchema 类型实例
if len(b.buildingViewStack) > 0 {
    is = temptable.DetachLocalTemporaryTableInfoSchema(is)
}
```

**步骤 3**：根据数据库名和表名，从 InfoSchema 实例中获取表实例

```go
tbl, err := is.TableByName(ctx, dbName, tn.Name)  // 从 InfoSchema 类型实例中获取 Table 类型的表实例
if err != nil {
    return nil, err
}
```

**步骤 4**：尝试调用 `tryLockMDLAndUpdateSchemaIfNecessary` 函数锁定元数据

```go
tbl, err = tryLockMDLAndUpdateSchemaIfNecessary(ctx, b.ctx, dbName, tbl, b.is)
if err != nil {
    return nil, err
}
```

**步骤 5**：检查当前用户是否有权限访问该表

```go
tableInfo := tbl.Meta()  // 从 Table 类型实例中获取 TableInfo 类型的表元数据信息实例

var authErr error
if sessionVars.User != nil {
    authErr = plannererrors.ErrTableaccessDenied.FastGenByArgs("SELECT", sessionVars.User.AuthUsername, sessionVars.User.AuthHostname, tableInfo.Name.L)
}
b.visitInfo = appendVisitInfo(b.visitInfo, mysql.SelectPriv, dbName.L, tableInfo.Name.L, "", authErr)
```

**步骤 6**：如果当前表为虚拟表，则调用 `PlanBuilder.buildMemTable` 函数构造虚拟表的逻辑执行计划。【TODO：`buildMemTable` 函数】

```go
if tbl.Type().IsVirtualTable() {
    if tn.TableSample != nil {
        return nil, expression.ErrInvalidTableSample.GenWithStackByArgs("Unsupported TABLESAMPLE in virtual tables")
    }
    return b.buildMemTable(ctx, dbName, tableInfo)
}
```

**步骤 7**：初始化表名变量 `tblName`

如果表有别名，则使用别名作为表名，否则使用原始表名。这个表名变量将在后续的访问路径构建和数据源初始化中使用。

```go
tblName := tableInfo.Name.L
if asName.L != "" {
    tblName = asName.L
}
```

**步骤 8**：处理分区表

如果表是分区表（`tableInfo.GetPartitionInfo() != nil`），则执行如下步骤。

**步骤 8.1**：根据会话变量和统计信息是否可用，判断是否启用动态分区采集功能

如果当前会话变量（`b.ctx.GetSessionVars().IsDynamicPartitionPruneEnabled()`）和全局环境（`b.ctx.GetSessionVars().StmtCtx.UseDynamicPruneMode`）均启动了动态分区裁剪模式，则会根据全局统计信息是否可用等条件，判断是否实际使用动态分区裁剪功能。

```go
h := domain.GetDomain(b.ctx).StatsHandle()
tblStats := h.GetTableStats(tableInfo)
isDynamicEnabled := b.ctx.GetSessionVars().IsDynamicPartitionPruneEnabled()
globalStatsReady := tblStats.IsAnalyzed()
skipMissingPartition := b.ctx.GetSessionVars().SkipMissingPartitionStats
// 如果启用了 tidb_skip_missing_partition_stats，全局统计信息可视为存在
allowDynamicWithoutStats := fixcontrol.GetBoolWithDefault(b.ctx.GetSessionVars().GetOptimizerFixControlMap(), fixcontrol.Fix44262, skipMissingPartition)

// 如果动态分区裁剪未启用或全局统计信息未准备好，不会在查询中启用动态裁剪模式
usePartitionProcessor := !isDynamicEnabled || (!globalStatsReady && !allowDynamicWithoutStats)

if usePartitionProcessor {
    b.optFlag = b.optFlag | rule.FlagPartitionProcessor
    b.ctx.GetSessionVars().StmtCtx.UseDynamicPruneMode = false
    if isDynamicEnabled {
        b.ctx.GetSessionVars().StmtCtx.AppendWarning(
            fmt.Errorf("disable dynamic pruning due to %s has no global stats", tableInfo.Name.String()))
    }
}
```

**步骤 8.2**：根据指定的分区，创建新的分区表实例

当用户在 SQL 查询中通过 `PARTITION` 子句显式指定要访问的分区时，TiDB 会根据分区名查找对应的分区 ID，并创建一个仅包含指定分区的分区表实例。

```go
pt := tbl.(table.PartitionedTable)
// check partition by name.
if len(tn.PartitionNames) > 0 {
    pids := make(map[int64]struct{}, len(tn.PartitionNames))
    for _, name := range tn.PartitionNames {
        pid, err := tables.FindPartitionByName(tableInfo, name.L)
        if err != nil {
            return nil, err
        }
        pids[pid] = struct{}{}
    }
    pt = tables.NewPartitionTableWithGivenSets(pt, pids)
}
b.partitionedTable = append(b.partitionedTable, pt)
```

**步骤 8.3**：如果当前表不是分区表（`tableInfo.GetPartitionInfo() == nil`），但在 SQL 查询中通过 `PARTITION` 子句显示指定了分区（`len(tn.PartitionNames) != 0`），则返回错误。

```go
return nil, plannererrors.ErrPartitionClauseOnNonpartitioned
```

**步骤 9**：获取可能的访问路径

调用 `getPossibleAccessPaths` 函数获取所有可能的表访问路径的列表（`[]*util.AccessPath`），并根据隔离级别和存储引擎进行过滤。

```go
possiblePaths, err := getPossibleAccessPaths(b.ctx, b.TableHints(), tn.IndexHints, tbl, dbName, tblName, b.isForUpdateRead, b.optFlag&rule.FlagPartitionProcessor > 0)
if err != nil {
    return nil, err
}
```

**步骤 10**：如果当前表为视图，则调用 `PlanBuilder.BuildDataSourceFromView` 函数构造视图的逻辑执行计划。【TODO：`buildDataSourceFromView` 函数】

```go
if tableInfo.IsView() {
    if tn.TableSample != nil {
        return nil, expression.ErrInvalidTableSample.GenWithStackByArgs("Unsupported TABLESAMPLE in views")
    }
    // ... 获取视图相关的提示信息 ...
    return b.BuildDataSourceFromView(ctx, dbName, tableInfo, currentQBNameMap4View, currentViewHints)
}
```

**步骤 11**：如果当前表为空序列，则调用 `PlanBuilder.buildTableDual` 函数生成空虚拟数据行的逻辑执行计划。

根据表的类型进行不同的处理：
- 对于序列，返回 `TableDual` 算子（类似于空表）

```go
if tableInfo.IsSequence() {
    if tn.TableSample != nil {
        return nil, expression.ErrInvalidTableSample.GenWithStackByArgs("Unsupported TABLESAMPLE in sequences")
    }
    return b.buildTableDual(), nil
}
```

**步骤 12**：调用 `filterPathByIsolationRead` 函数根据隔离读设置过滤访问路径

```go
if !(b.isForUpdateRead && b.ctx.GetSessionVars().TxnCtx.IsExplicit) {
    if b.capFlag&canExpandAST == 0 {
        possiblePaths, err = filterPathByIsolationRead(b.ctx, possiblePaths, tblName, dbName)
        if err != nil {
            return nil, err
        }
    }
}
```

**步骤 13**：检查是否需要启用生成列替代优化

如果在生成列上存在索引，则尝试替换生成列。遍历索引中的每个列，如果该列是生成列（`colInfo.IsGenerated()`）且该列为虚拟生成列（`!colInfo.GeneratedStored`），即需要在读取时即时计算的列，则尝试将对生成列的引用替换为原始列和对应的表达式（`b.optFlag |= rule.FlagGcSubstitute`）。

```go
for _, index := range tableInfo.Indices {
    if index.State != model.StatePublic {
        continue
    }
    for _, indexCol := range index.Columns {
        colInfo := tbl.Cols()[indexCol.Offset]
        if colInfo.IsGenerated() && !colInfo.GeneratedStored {
            b.optFlag |= rule.FlagGcSubstitute
            break
        }
    }
}
```

**步骤 14**：根据 SQL 语句类型，采用不同方式获取表中列的数组

- 更新语句（`inUpdateStmt`）：使用 `WritableCols` 函数获取表中所有可写的列，包括那些正在进行模式变更的列从而避免在分布式环境中，由于模式变更的不一致性导致数据丢失。
- 删除语句（`Delete`）：使用 `DeletableCols` 函数获取表中所有可删除的列。
- 其他语句：使用 `Cols` 函数获取表的所有列，用于常规查询操作。

```go
var columns []*table.Column
if b.inUpdateStmt {
    columns = tbl.WritableCols()
} else if b.inDeleteStmt {
    columns = tbl.DeletableCols()
} else {
    columns = tbl.Cols()
}
```

**步骤 15**：提取并验证索引合并提示（IndexMergeHint）

此步骤从表提示（TableHints）中提取索引合并提示，并验证指定的索引名是否有效。索引合并提示用于指导优化器使用多个索引合并的方式来访问表数据。

```go
// extract the IndexMergeHint
var indexMergeHints []h.HintedIndex
if hints := b.TableHints(); hints != nil {
    for i, hint := range hints.IndexMergeHintList {
        if hint.Match(dbName, tblName) {
            hints.IndexMergeHintList[i].Matched = true
            // 检查 IndexMergeHint 中的索引名是否有效
            invalidIdxNames := make([]string, 0, len(hint.IndexHint.IndexNames))
            for _, idxName := range hint.IndexHint.IndexNames {
                hasIdxName := false
                for _, path := range possiblePaths {
                    if path.IsTablePath() {
                        if idxName.L == "primary" {
                            hasIdxName = true
                            break
                        }
                        continue
                    }
                    if idxName.L == path.Index.Name.L {
                        hasIdxName = true
                        break
                    }
                }
                if !hasIdxName {
                    invalidIdxNames = append(invalidIdxNames, idxName.String())
                }
            }
            if len(invalidIdxNames) == 0 {
                indexMergeHints = append(indexMergeHints, hint)
            } else {
                // 如果有无效的索引名，则添加警告
                errMsg := fmt.Sprintf("use_index_merge(%s) is inapplicable, check whether the indexes (%s) "+
                    "exist, or the indexes are conflicted with use_index/ignore_index/force_index hints.",
                    hint.IndexString(), strings.Join(invalidIdxNames, ", "))
                b.ctx.GetSessionVars().StmtCtx.SetHintWarning(errMsg)
            }
        }
    }
}
```

**步骤 16**：创建并初始化 DataSource 逻辑算子

首先复制一份访问路径列表，并计算需要的列数（加 1 是为了额外的句柄列），然后创建 `DataSource` 逻辑算子，并设置其属性。

```go
allPaths := make([]*util.AccessPath, len(possiblePaths))
copy(allPaths, possiblePaths)
countCnt := len(columns) + 1 // +1 for an extra handle column

ds := logicalop.DataSource{
    DBName:                 dbName,
    TableAsName:            asName,
    Table:                  tbl,
    TableInfo:              tableInfo,
    PhysicalTableID:        tableInfo.ID,
    AstIndexHints:          tn.IndexHints,
    IndexHints:             b.TableHints().IndexHintList,
    IndexMergeHints:        indexMergeHints,
    PossibleAccessPaths:    possiblePaths,
    AllPossibleAccessPaths: allPaths,
    Columns:                make([]*model.ColumnInfo, 0, countCnt),
    PartitionNames:         tn.PartitionNames,
    TblCols:                make([]*expression.Column, 0, countCnt),
    PreferPartitions:       make(map[int][]ast.CIStr),
    IS:                     b.is,
    IsForUpdateRead:        b.isForUpdateRead,
}.Init(b.ctx, b.getSelectOffset())
```

`DataSource` 逻辑算子中各个字段含义及来源如下：

- `DBName`：数据库名，来自解析 SQL 语句获得的数据库名或当前会话的默认数据库
- `TableAsName`：表的别名，来自 SQL 语句中的 AS 子句
- `Table`：底层表对象，从 InfoSchema 中获取
- `TableInfo`：表的元数据信息，通过 `tbl.Meta()` 获取
- `PhysicalTableID`：表的物理 ID，直接使用 `tableInfo.ID`
- `AstIndexHints`：AST 中的索引提示，来自 SQL 语句的解析结果
- `IndexHints`：当前查询的索引提示列表，通过 `b.TableHints().IndexHintList` 获取
- `IndexMergeHints`：索引合并提示，在上一步中从表提示中提取并验证
- `PossibleAccessPaths`：可能的访问路径，通过 `getPossibleAccessPaths` 获取
- `AllPossibleAccessPaths`：所有可能的访问路径的副本，用于保留原始路径信息
- `Columns`：表的列信息，初始为空，容量为 `countCnt`
- `PartitionNames`：分区名列表，来自 SQL 语句的解析结果
- `TblCols`：表的列表达式，初始为空，容量为 `countCnt`
- `PreferPartitions`：首选分区映射，初始为空，用于优化分区表访问
- `IS`：InfoSchema 对象，用于获取元数据
- `IsForUpdateRead`：是否为 UPDATE 语句的读取操作

**步骤 17**：遍历表中列的数组，补充 `DataSource` 实例中的信息

遍历在 *步骤 14* 中生成的表中列的数组：

- 构造 `FieldName` 实例并添加到 `names` 数组
- 构造 `Column` 实例并添加到 `schema` 数字
- 如果为主键列（`col.IsPKHandleColumn(tableInfo)`），则更新 `handleCols` 变量
- 填充 `DataSource` 实例中的 `TblCols` 字段

```go
var handleCols util.HandleCols
schema := expression.NewSchema(make([]*expression.Column, 0, countCnt)...)
names := make([]*types.FieldName, 0, countCnt)

for i, col := range columns {
    ds.Columns = append(ds.Columns, col.ToInfo())
    names = append(names, &types.FieldName{
        DBName:      dbName,
        TblName:     tableInfo.Name,
        ColName:     col.Name,
        OrigTblName: tableInfo.Name,
        OrigColName: col.Name,
        NotExplicitUsable: col.State != model.StatePublic,
    })
    newCol := &expression.Column{
        UniqueID: sessionVars.AllocPlanColumnID(),
        ID:       col.ID,
        RetType:  col.FieldType.Clone(),
        OrigName: names[i].String(),
        IsHidden: col.Hidden,
    }
    if col.IsPKHandleColumn(tableInfo) {
        handleCols = util.NewIntHandleCols(newCol)
    }
    schema.Append(newCol)
    ds.TblCols = append(ds.TblCols, newCol)
}
```

**步骤 18**：处理句柄列的特殊情况

如果当前查询字段中没有主键列（`handleCols == nil`），则需要根据表的类型添加合适的句柄列，以便后续处理。

对于使用聚簇索引的表（`tableInfo.IsCommonHandle`）：
- 查找主键索引（`tables.FindPrimaryIndex(tableInfo)`）
- 创建公共句柄列（`util.NewCommonHandleCols`）用于表示聚簇索引键

对于使用隐式行 ID 的表：
- 创建额外的句柄列（`ds.NewExtraHandleSchemaCol()`）
- 将这个额外的句柄列添加到 `schema`、`names` 和 `TblCols` 中
- 这个额外的句柄列使用特殊的名称 `model.ExtraHandleName`（通常是 `_tidb_rowid`）

```go
if handleCols == nil {
    if tableInfo.IsCommonHandle {
        primaryIdx := tables.FindPrimaryIndex(tableInfo)
        handleCols = util.NewCommonHandleCols(tableInfo, primaryIdx, ds.TblCols)
    } else {
        extraCol := ds.NewExtraHandleSchemaCol()
        handleCols = util.NewIntHandleCols(extraCol)
        ds.Columns = append(ds.Columns, model.NewExtraHandleColInfo())
        schema.Append(extraCol)
        names = append(names, &types.FieldName{
            DBName:      dbName,
            TblName:     tableInfo.Name,
            ColName:     model.ExtraHandleName,
            OrigColName: model.ExtraHandleName,
        })
        ds.TblCols = append(ds.TblCols, extraCol)
    }
}
```

**步骤 19**：补充 `DataSource` 实例的相关信息

直接更新 `DataSource` 实例的如下 3 个字段：

- `HandleCols`：主键列，用于唯一标识行
- `UnMutableHandleCols`：不可变主键列
- `SampleInfo`：处理表采样信息（如果 SQL 中有 `TABLESAMPLE` 子句）

调用函数更新 `DataSource` 的 2 项信息：

- `SetSchema`：数据源的输出 Schema
- `SetOutputNames`：数据源的输出列名称

更新 `PlanBuilder` 实例的如下 2 个字段：

- `handleHelper`：将当前表 ID 到句柄列的映射添加到栈中
- `isSampling`：标记当前查询是否包含表采样

此外，还调用 `setPreferredStoreType` 函数根据用户提供的表提示设置首选存储类型。

```go
ds.HandleCols = handleCols                      // 设置 DataSource 的句柄列字段，用于唯一标识行
ds.UnMutableHandleCols = handleCols             // 设置不可变句柄列，用于确保操作过程中句柄列不被修改
handleMap := make(map[int64][]util.HandleCols)  // 创建表 ID 到句柄列的映射表
handleMap[tableInfo.ID] = []util.HandleCols{handleCols}  // 将当前表的句柄列添加到映射表中
b.handleHelper.pushMap(handleMap)               // 将句柄映射表推入 handleHelper 栈中，用于后续查询处理
ds.SetSchema(schema)                            // 设置数据源的输出 Schema
ds.SetOutputNames(names)                        // 设置数据源的输出列名称
setPreferredStoreType(ds, b.TableHints())       // 根据用户提供的表提示设置首选存储类型
ds.SampleInfo = tablesampler.NewTableSampleInfo(tn.TableSample, schema, b.partitionedTable)  // 处理表采样信息（如果 SQL 中有 TABLESAMPLE 子句）
b.isSampling = ds.SampleInfo != nil             // 设置查询构建器的采样标志，表示当前查询是否包含表采样
```

**步骤 20**：将虚拟生成列重写为 `expression.Expression` 表达式

遍历 DataSource 的 Schema 中的所有列，如果是虚拟生成列（`columns[i].IsGenerated()` 且 `!columns[i].GeneratedStored`），则调用 `PlanBuilder.rewrite` 函数将生成列的原始表达式（AST 节点）转换为优化器可以处理的 `expression.Expression` 类型。

```go
for i, colExpr := range ds.Schema().Columns {
    var expr expression.Expression
    if i < len(columns) {
        if columns[i].IsGenerated() && !columns[i].GeneratedStored {
            var err error
            originVal := b.allowBuildCastArray  // 保存原始的 allowBuildCastArray 标志值
            b.allowBuildCastArray = true        // 临时允许构建类型转换数组，以支持生成列表达式中的类型转换
            expr, _, err = b.rewrite(ctx, columns[i].GeneratedExpr.Clone(), ds, nil, true)  // 重写生成列表达式
            b.allowBuildCastArray = originVal   // 恢复原始的 allowBuildCastArray 标志值
            if err != nil {
                return nil, err
            }
            colExpr.VirtualExpr = expr.Clone()  // 将重写后的表达式克隆并保存到列的 VirtualExpr 字段
        }
    }
}
```

**步骤 21**：初始化聚簇索引的主键信息

对于使用聚簇索引的表（`tableInfo.IsCommonHandle`），调用 `IndexInfo2Cols` 函数，将表的主键索引信息转换为表达式列数组（`[]*expression.Column`）和列长度数组（`[]int`），更新到 `DataSource` 实例的 `CommonHandleCols` 字段和 `CommonHandleLens` 字段。

```go
if tableInfo.IsCommonHandle {
    ds.CommonHandleCols, ds.CommonHandleLens = expression.IndexInfo2Cols(ds.Columns, ds.Schema().Columns, tables.FindPrimaryIndex(tableInfo))
}
```

**步骤 22**：为所有访问路径初始化索引列和索引列长度

遍历所有可能的访问路径（`ds.AllPossibleAccessPaths`），如果该访问路径为表路径（`!path.IsIntHandlePath`），则调用 `IndexInfo2Cols` 函数，初始化其索引列数组和索引列长度数组，更新到 `AccessPath` 实例的 `FullIdxCols` 字段和 `FullIdxColLens` 字段。

如果索引中包含 `tidb_shard()` 前缀且索引列数量大于 1，例如 `index(tidb_shard(a), a)`，则为二级唯一索引设置 `IsUkShardIndexPath` 标记。【TODO：待深入处理】

```go
for _, path := range ds.AllPossibleAccessPaths {
    if !path.IsIntHandlePath {
        path.FullIdxCols, path.FullIdxColLens = expression.IndexInfo2Cols(ds.Columns, ds.Schema().Columns, path.Index)

        if !path.IsCommonHandlePath {
            // tidb_shard 表达式必须是索引的第一列
            col := path.FullIdxCols[0]
            if col != nil &&
                expression.GcColumnExprIsTidbShard(col.VirtualExpr) &&
                len(path.Index.Columns) > 1 &&
                path.Index.Unique {
                path.IsUkShardIndexPath = true
                ds.ContainExprPrefixUk = true
            }
        }
    }
}
```

**步骤 23**：初始化返回结果变量

```go
var result base.LogicalPlan = ds
```

**步骤 24**：处理未提交事务的脏数据

如果当前表满足如下任意一个条件：

- 存在脏更新，即事务中未提交的修改（`tableHasDirtyContent(b.ctx, tableInfo)`）
- 为本地临时表（`tableInfo.TempTableType == model.TempTableLocal`）
- 启用了表缓存（`tableInfo.TableCacheStatusType == model.TableCacheStatusEnable`）

则创建一个 `LogicalUnionScan` 算子，将 `DataSource` 设置为其子节点，用于合并未提交的事务数据与存储的数据。

对于分区表（`tableInfo.Partition != nil`），在非分区处理器模式下（`b.optFlag&rule.FlagPartitionProcessor == 0`），还需要通过 `addExtraPhysTblIDColumn4DS` 函数为 `DataSource` 添加额外的 `ExtraPhysTblIDCol`，以便在事务缓冲区处理中正确识别数据来源的分区。

```go
dirty := tableHasDirtyContent(b.ctx, tableInfo)
if dirty || tableInfo.TempTableType == model.TempTableLocal || tableInfo.TableCacheStatusType == model.TableCacheStatusEnable {
    us := logicalop.LogicalUnionScan{HandleCols: handleCols}.Init(b.ctx, b.getSelectOffset())
    us.SetChildren(ds)
    if tableInfo.Partition != nil && b.optFlag&rule.FlagPartitionProcessor == 0 {
        // Adding ExtraPhysTblIDCol for UnionScan (transaction buffer handling)
        // Not using old static prune mode
        // Single TableReader for all partitions, needs the PhysTblID from storage
        _ = addExtraPhysTblIDColumn4DS(ds)
    }
    result = us
}
```

**步骤 25**：在会话变量中记录记录表的脏更新状态

```go
if sessionVars.StmtCtx.TblInfo2UnionScan == nil {
    sessionVars.StmtCtx.TblInfo2UnionScan = make(map[*model.TableInfo]bool)
}
sessionVars.StmtCtx.TblInfo2UnionScan[tableInfo] = dirty
```

**步骤 26**：返回最终逻辑计划

```go
return result, nil
```
