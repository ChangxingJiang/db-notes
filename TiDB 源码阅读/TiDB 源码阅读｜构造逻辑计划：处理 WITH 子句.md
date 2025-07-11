# TiDB 源码阅读｜构造逻辑计划：处理 WITH 子句

`buildWith` 函数用于构造 SQL 语句中 `WITH` 子句的逻辑执行计划。该函数的返回值类型为 `cteInfo` 类型数组，其中包含了所有公用表表达式（Common Table Expression，CTE）的相关信息。

在 `buildWith` 函数中，先检查 CTE 名称的唯一性，然后遍历 `WITH` 子句中的每个 CTE，创建 `cteInfo` 结构体实例并调用 `buildCte` 函数为其构造逻辑执行计划。在 `buildCte` 函数中，对于非递归 CTE，直接调用 `buildResultSetNode` 函数生成逻辑执行计划；对于递归 CTE，调用 `buildRecursiveCTE` 函数生成逻辑执行计划。

【TODO：最终构造结果图】

## `cteInfo` 结构体介绍

`cteInfo` 结构体定义在 `pkg/planner/core/planbuilder.go` 文件中，用于存储 CTE 的各种信息：

- `def` ：指向 AST 中的 CommonTableExpression 节点，包含 CTE 的原始定义
- `nonRecursive` ：用于决定 CTE 是否可见。如果 CTE 以 `WITH RECURSIVE` 开头，则 `nonRecursive` 为 false，这意味着它在其定义中是可见的
- `useRecursive` ：用于记录 CTE 定义中的子查询是否引用了自身。这有助于识别种子部分和递归部分
- `isBuilding` ：标记 CTE 是否正在构建中
- `isDistinct` ：表示种子部分和递归部分之间的 UNION 操作是否是 DISTINCT 的
- `seedLP` ：CTE 种子部分的逻辑计划
- `recurLP` ：CTE 递归部分的逻辑计划
- `storageID` ：此 CTE 的存储标识符
- `optFlag` ：整个 CTE 的优化标志
- `enterSubquery` 和 `recursiveRef` ：用于检查"递归表必须只被引用一次，且不在任何子查询中"
- `limitLP` ：带有 LIMIT 子句的逻辑计划
- `seedStat` ：在 logicalCTE 和 logicalCTETable 之间共享的统计信息
- `cteClass` ：引用相同表的 LogicalCTE 应共享相同的 CteClass
- `isInline` ：当 CTE 被使用时，决定是否可以内联
- `forceInlineByHintOrVar` ：当 CTE 被 merge() 提示或会话变量 "tidb_opt_force_inline_cte=true" 标记时为 true
- `containRecursiveForbiddenOperator` ：如果 CTE 在查询中包含聚合、窗口函数、ORDER BY、DISTINCT 和 LIMIT（查询中对包含这些运算符的其他 CTE 的间接引用也会被计数）
- `consumerCount` ：在预处理阶段计算，记录当前 CTE 有多少个消费者

## `buildWith` 函数处理流程

`buildWith` 函数定义在 `pkg/planner/core/logical_plan_builder.go` 文件中。`buildWith` 函数是 TiDB 处理公用表表达式（CTE）的关键组件，它负责将 SQL 语句中的 WITH 子句转换为对应的逻辑执行计划。通过管理 CTE 的各种属性和关系，为查询优化器提供了处理复杂查询结构的基础。`buildWith` 函数的签名如下：

```go
func (b *PlanBuilder) buildWith(ctx context.Context, w *ast.WithClause) ([]*cteInfo, error)
```

**步骤 1**：检查 CTE 名称的唯一性

首先，函数会检查 WITH 子句中所有 CTE 的名称是否唯一。它创建一个小写名称的映射表，用于快速查找是否有重复的 CTE 名称。如果发现重复名称，函数会返回 `ErrNonUniqTable` 错误。

```go
// Check CTE name must be unique.
b.nameMapCTE = make(map[string]struct{})
for _, cte := range w.CTEs {
    if _, ok := b.nameMapCTE[cte.Name.L]; ok {
        return nil, plannererrors.ErrNonUniqTable
    }
    b.nameMapCTE[cte.Name.L] = struct{}{}
}
```

**步骤 2**：处理 WITH 子句中的每个 CTE 定义

函数首先初始化一个数组用于存储 CTE 信息，然后循环处理每个 CTE 定义。

```go
ctes := make([]*cteInfo, 0, len(w.CTEs))
for _, cte := range w.CTEs {
    // 后续步骤将在这个循环中进行
}
```

在循环中，函数执行以下子步骤来处理每个 CTE：

**步骤 2.1**：创建 CTE 信息对象并初始化基本属性

为每个 CTE 创建一个新的 `cteInfo` 结构体实例，并设置其基本属性，包括原始定义、递归标志、构建状态、存储 ID 和统计信息等。

```go
b.outerCTEs = append(b.outerCTEs, &cteInfo{
    def:           cte, 
    nonRecursive:  !w.IsRecursive, 
    isBuilding:    true, 
    storageID:     b.allocIDForCTEStorage, 
    seedStat:      &property.StatsInfo{}, 
    consumerCount: cte.ConsumerCount
})
b.allocIDForCTEStorage++
```

**步骤 2.2**：保存并设置优化标志

保存当前的优化标志，并将其设置为列剪裁标志（`FlagPruneColumns`），确保在构建 CTE 的过程中开启列剪裁优化。

```go
saveFlag := b.optFlag
// Init the flag to flagPrunColumns, otherwise it's missing.
b.optFlag = rule.FlagPruneColumns
```

**步骤 2.3**：检查是否需要强制内联 CTE

根据会话变量检查是否需要强制内联 CTE。如果会话变量 `tidb_opt_force_inline_cte` 设置为 true，则标记当前 CTE 为强制内联。

> 内联 CTE 是一种优化技术，它将 CTE 的定义直接替换到引用它的地方，而不是将 CTE 作为一个独立的执行单元。这类似于编程语言中的函数内联。强制内联可以减少中间结果的存储和传输开销，但可能在某些情况下增加查询的复杂性。内联决策通常由优化器基于成本模型自动做出，但强制内联选项允许用户覆盖优化器的决定，特别是当用户确信内联会带来更好性能时。
>
> TiDB 提供了 `tidb_opt_force_inline_cte` 系统变量来控制是否强制内联 CTE。当设置为 ON 时，所有非递归 CTE 都会被内联。用户也可以通过 `MERGE()` 提示来针对特定 CTE 启用内联，即使全局设置是关闭的。这种灵活性允许用户根据具体查询特点和性能需求调整 CTE 的处理方式。

```go
if b.ctx.GetSessionVars().EnableForceInlineCTE() {
    b.outerCTEs[len(b.outerCTEs)-1].forceInlineByHintOrVar = true
}
```

**步骤 2.4**：构建 CTE 的逻辑执行计划

调用 `buildCte` 函数构建 CTE 的具体逻辑执行计划。该函数会根据 CTE 是否递归采用不同的构建策略：
- 对于非递归 CTE，直接调用 `buildResultSetNode` 构建查询计划
- 对于递归 CTE，通过 `buildRecursiveCTE` 函数处理递归结构，识别种子部分和递归部分

```go
_, err := b.buildCte(ctx, cte, w.IsRecursive)
if err != nil {
    return nil, err
}
```

**步骤 2.5**：更新 CTE 信息状态

更新 CTE 的优化标志，将从 `buildCte` 中获得的优化标志保存到 CTE 信息中，并将 `isBuilding` 标记设为 false，表示 CTE 的构建已完成。

```go
b.outerCTEs[len(b.outerCTEs)-1].optFlag = b.optFlag
b.outerCTEs[len(b.outerCTEs)-1].isBuilding = false
```

**步骤 2.6**：恢复优化标志

恢复之前保存的优化标志，确保 CTE 构建过程中的优化设置不影响后续的处理。

```go
b.optFlag = saveFlag
```

**步骤 2.7**：处理 handleHelper

`buildCte()` 会向 `handleHelper` 中推入一个条目，但根据注释，构建 CTE 不应影响 `handleColHelper`，因此从 `handleHelper` 中弹出一个条目，以确保 CTE 构建不影响整体的 `handleColHelper`。

```go
b.handleHelper.popMap()
```

**步骤 2.8**：添加 cteInfo 到结果数组

```go
ctes = append(ctes, b.outerCTEs[len(b.outerCTEs)-1])
```

**步骤 3**：返回包含所有 CTE 信息的数组。

```go
return ctes, nil
```

## `buildCte` 函数处理流程

`buildCte` 函数是构建 CTE 逻辑执行计划的核心函数，根据 CTE 是递归还是非递归采取不同的处理策略。该函数的签名如下：

```go
func (b *PlanBuilder) buildCte(ctx context.Context, cte *ast.CommonTableExpression, isRecursive bool) (p base.LogicalPlan, err error)
```

### 非递归 CTE 的处理流程

当 `isRecursive` 为 false 时，`buildCte` 函数采用较为简单的处理逻辑：

**步骤 1**：标记构建状态

函数首先保存并设置 `buildingCTE` 标志，表明当前正在构建 CTE。

```go
saveBuildingCTE := b.buildingCTE
b.buildingCTE = true
defer func() {
    b.buildingCTE = saveBuildingCTE
}()
```

**步骤 2**：构建结果集节点

直接调用 `buildResultSetNode` 函数来构建查询的逻辑执行计划。【TODO：`buildResultSetNode` 函数】

```go
p, err := b.buildResultSetNode(ctx, cte.Query.Query, true)
if err != nil {
    return nil, err
}
```

**步骤 3**：调整输出列名

调用 `adjustCTEPlanOutputName` 函数调整逻辑计划的输出列名，使其与 CTE 定义一致。

```go
p, err = b.adjustCTEPlanOutputName(p, cte)
if err != nil {
    return nil, err
}
```

**步骤 4**：保存逻辑计划

将构建好的逻辑计划保存到 `cteInfo` 的 `seedLP` 字段中。

```go
cInfo := b.outerCTEs[len(b.outerCTEs)-1]
cInfo.seedLP = p
```

### 递归 CTE 的处理流程

当 `isRecursive` 为 true 时，处理逻辑更为复杂，需要调用 `buildRecursiveCTE` 函数来处理递归结构：

**步骤 1**：初始化递归标志

保存并设置递归标志，用于后续处理。

```go
saveCheck := b.buildingRecursivePartForCTE
b.buildingRecursivePartForCTE = false
```

**步骤 2**：调用 `buildRecursiveCTE` 函数处理递归 CTE

```go
err = b.buildRecursiveCTE(ctx, cte.Query.Query)
if err != nil {
    return nil, err
}
```

**步骤 3**：移除递归标志

```go
b.buildingRecursivePartForCTE = saveCheck
```

## `buildRecursiveCTE` 函数处理流程

`buildRecursiveCTE` 函数是专门构建递归 CTE 逻辑执行计划的函数。该函数的签名如下：

```go
func (b *PlanBuilder) buildRecursiveCTE(ctx context.Context, cte ast.ResultSetNode) error
```

如果当前语句不是 `UNION`、`EXCEPT` 或 `INTERSECT` 语句（`SetOprStmt`），则直接执行非递归 CTE 处理流程的步骤 2 - 步骤 4。

```go
p, err := b.buildResultSetNode(ctx, x, true)
if err != nil {
    // Refine the error message.
    if errors.ErrorEqual(err, plannererrors.ErrCTERecursiveRequiresNonRecursiveFirst) {
        err = plannererrors.ErrCTERecursiveRequiresUnion.GenWithStackByArgs(cInfo.def.Name.String())
    }
    return err
}
p, err = b.adjustCTEPlanOutputName(p, cInfo.def)
if err != nil {
    return err
}
cInfo.seedLP = p
return nil
```

否则，执行如下步骤：

**步骤 1**：处理嵌套的 `WITH` 子句

如果递归 CTE 中包含嵌套的 `WITH` 子句，则需要先调用 `buildWith` 函数递归地处理这些嵌套的 WITH 子句。在处理完成后，将嵌套的 `WITH` 语句置为空，避免重复构建。

```go
if x.With != nil {
    l := len(b.outerCTEs)
    sw := x.With
    defer func() {
        b.outerCTEs = b.outerCTEs[:l]
        x.With = sw
    }()
    _, err := b.buildWith(ctx, x.With)
    if err != nil {
        return err
    }
}
x.With = nil
```

**步骤 2**：为 `SetOptStmt` 中的每个部分构造逻辑执行计划

该步骤的整体逻辑是处理递归 CTE 中的集合操作语句（`UNION`、`EXCEPT` 或 `INTERSECT`），将其分解为种子部分和递归部分。在这个过程中，函数首先依次构建每个子查询的逻辑计划，并通过检测是否引用了 CTE 自身来判断当前处理的是种子部分还是递归部分。种子部分不包含对 CTE 自身的引用，而递归部分必须包含。函数还会验证种子部分和递归部分之间的 `UNION` 类型，确保它们符合递归 CTE 的语法规则，并正确处理相关的 `LIMIT` 子句和输出列名，最终构建完整的递归 CTE 逻辑计划。

在此过程中，使用两个重要的标志变量来跟踪和控制递归 CTE 的构建：

- `expectSeed`：布尔变量，表示当前是否处于种子部分的处理阶段。初始值为 `true`，表示函数开始时期望处理种子部分。当发现第一个包含递归引用的子查询时，会将其设置为 `false`，标志着种子部分处理结束，开始处理递归部分。

- `cInfo.useRecursive`：`cteInfo` 结构体中的布尔字段，用于记录当前正在处理的子查询是否引用了 CTE 自身。当构建子查询的逻辑计划时，如果发现对 CTE 自身的引用，该标志会被设置为 `true`。这个标志帮助函数识别递归部分，并确保种子部分不包含递归引用，而递归部分必须包含至少一个递归引用。

遍历 `SetOptStmt` 中的每个语句，执行如下逻辑：

> `SetOptStmt` 结构体定义在 `pkg/parser/ast/dml.go` 中，其定义如下：
>
> ```go
> type SetOprStmt struct {
> 	dmlNode
> 
> 	IsInBraces bool
> 	SelectList *SetOprSelectList
> 	OrderBy    *OrderByClause
> 	Limit      *Limit
> 	With       *WithClause
> }

**步骤 2.1**：构建当前 Select 语句的逻辑计划

根据当前 Select 语句的类型（普通 `SelectStmt` 或嵌套的 `SetOprSelectList`），调用相应的构建函数生成逻辑计划，并获取下一个 `UNION`、`EXCEPT` 或 `INTERSECT` 的操作类型。

```go
var p base.LogicalPlan
var err error
originalLen := b.handleHelper.stackTail

var afterOpr *ast.SetOprType
switch y := x.SelectList.Selects[i].(type) {
case *ast.SelectStmt:
    p, err = b.buildSelect(ctx, y)
    afterOpr = y.AfterSetOperator
case *ast.SetOprSelectList:
    p, err = b.buildSetOpr(ctx, &ast.SetOprStmt{SelectList: y, With: y.With})
    afterOpr = y.AfterSetOperator
}
```

**步骤 2.2**：维护 `handleHelper` 栈

处理 handleHelper 栈，确保即使在错误情况下也能正确恢复栈状态。这对于处理递归 CTE 特别重要，因为在构建递归部分时可能会遇到预期的错误。

```go
if err == nil {
    b.handleHelper.popMap()
} else {
    // 处理特殊情况：当构建第一个递归部分时预期会有错误
    // 但 handleHelper 栈需要正确恢复，避免栈不平衡
    for b.handleHelper.stackTail > originalLen {
        b.handleHelper.popMap()
    }
}
```

此时：

- 如果是递归引用，且之前没有添加过递归，则执行步骤 2.3，创建递归种子
- 如果是递归引用，但之前已经出现过递归，则执行步骤 2.5，向 `recursive` 中添加
- 如果不是递归引用，则执行步骤 2.4，不进行额外处理

**步骤 2.3**：首个递归处理

```go
// 如果第一个子查询就包含递归引用，则报错
if i == 0 {
    return plannererrors.ErrCTERecursiveRequiresNonRecursiveFirst.GenWithStackByArgs(cInfo.def.Name.String())
}

// 检查递归 CTE 的限制条件
if x.OrderBy != nil {
    return plannererrors.ErrNotSupportedYet.GenWithStackByArgs("ORDER BY over UNION in recursive Common Table Expression")
}

// 保存原始 LIMIT 子句，因为 LIMIT 应用于整个 CTE 而非仅种子部分
oriLimit := x.Limit
x.Limit = nil

// 验证种子部分和递归部分之间的 UNION 类型
if afterOpr != nil {
    if *afterOpr != ast.Union && *afterOpr != ast.UnionAll {
        return plannererrors.ErrNotSupportedYet.GenWithStackByArgs(fmt.Sprintf("%s between seed part and recursive part, hint: The operator between seed part and recursive part must bu UNION[DISTINCT] or UNION ALL", afterOpr.String()))
    }
    cInfo.isDistinct = *afterOpr == ast.Union
}

// 重置状态标志，准备构建种子部分
expectSeed = false
cInfo.useRecursive = false

// 构建种子部分的逻辑计划
saveSelect := x.SelectList.Selects
x.SelectList.Selects = x.SelectList.Selects[:i]
p, err = b.buildSetOpr(ctx, x)
if err != nil {
    return err
}
b.handleHelper.popMap()
x.SelectList.Selects = saveSelect

// 调整输出列名并保存种子部分的逻辑计划
p, err = b.adjustCTEPlanOutputName(p, cInfo.def)
if err != nil {
    return err
}
cInfo.seedLP = p

// 重新构建计划，开始处理递归部分
i--
b.buildingRecursivePartForCTE = true
x.Limit = oriLimit
continue
```

**步骤 2.4**：非递归处理

```go
if err != nil {
    return err
}
```

**步骤 2.5**：处理其他递归部分

```go
if err != nil {
    return err
}

// 验证递归部分内部的 UNION 操作类型
if afterOpr != nil {
    if *afterOpr != ast.Union && *afterOpr != ast.UnionAll {
        return plannererrors.ErrNotSupportedYet.GenWithStackByArgs(fmt.Sprintf("%s between recursive part's selects, hint: The operator between recursive part's selects must bu UNION[DISTINCT] or UNION ALL", afterOpr.String()))
    }
}

// 确保递归部分确实包含对 CTE 自身的引用
if !cInfo.useRecursive {
    return plannererrors.ErrCTERecursiveRequiresNonRecursiveFirst.GenWithStackByArgs(cInfo.def.Name.String())
}

// 重置递归标志并收集递归部分的逻辑计划
cInfo.useRecursive = false
recursive = append(recursive, p)
tmpAfterSetOptsForRecur = append(tmpAfterSetOptsForRecur, afterOpr)
```

如果没有识别到递归部分（`len(recursive) == 0`），则执行步骤 2.6，否则执行步骤 2.7 - 2.9

**步骤 2.6**：处理非递归 CTE 的情况

则将整个 CTE 作为非 CTE 处理：

```go
// 即使 SQL 指定了 "WITH RECURSIVE"，如果没有递归引用，CTE 仍是非递归的
p, err := b.buildSetOpr(ctx, x)
if err != nil {
    return err
}
p, err = b.adjustCTEPlanOutputName(p, cInfo.def)
if err != nil {
    return err
}
cInfo.seedLP = p
return nil
```

**步骤 2.7**：构建递归部分的完整逻辑计划

调用 `buildUnion` 函数和 `buildProjection4CTEUnion` 函数，将收集到的递归部分组合成完整的逻辑执行计划，并与种子部分关联。

```go
// 构建递归部分的逻辑计划
recurPart, err := b.buildUnion(ctx, recursive, tmpAfterSetOptsForRecur)
if err != nil {
    return err
}

// 为 CTE 的 UNION 操作构建投影计划，确保种子部分和递归部分的输出结构一致
recurPart, err = b.buildProjection4CTEUnion(ctx, cInfo.seedLP, recurPart)
if err != nil {
    return err
}

// 保存递归部分的逻辑计划
cInfo.recurLP = recurPart
```

**步骤 2.8**：处理 LIMIT 子句

如果原始语句包含 LIMIT 子句，为 CTE 构建 LIMIT 逻辑计划：

```go
// 仅当 x 是 SetOprStmt 时才需要处理 LIMIT
if x.Limit != nil {
    limit, err := b.buildLimit(cInfo.seedLP, x.Limit)
    if err != nil {
        return err
    }
    limit.SetChildren(limit.Children()[:0]...)
    cInfo.limitLP = limit
}
```

**步骤 2.9**：完成构建

推入一个空的 handleHelper 映射，表示 CTE 构建完成：

```go
b.handleHelper.pushMap(nil)
return nil
```
