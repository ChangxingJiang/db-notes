# TiDB 源码阅读｜优化器的整体结构

## 优化器整体流程

TiDB 优化器将 SQL 语句转换为执行计划的过程涉及多个阶段和函数，主要步骤如下：

1. 只读模式检查（`Optimize` 函数）
2. 设置资源组（`Optimize` 函数）
3. 若为预处理语句，则直接执行预处理语句（`Optimize` 函数）
4. 尝试查找缓存的执行计划，若找到则直接使用（`Optimize` 函数）
5. 处理 SQL 提示，设置会话变量和优化器行为（`optimizeNoCache` 函数）
6. 尝试构造点查快速执行计划（`optimizeNoCache` 函数）
7. 尝试采用绑定执行计划（`optimizeNoCache` 函数）
8. 事务预热（`optimizeNoCache` 函数）
9. 构建逻辑计划（`optimize` 函数）
10. 权限检查（`optimize` 函数）
11. 执行级联优化（`CascadesOptimize` 函数）或火山模型优化（`VolcanoOptimize` 函数）
12. 计划基线演进（`optimizeNoCache` 函数）

级联优化和火山模型优化包含的具体操作如下：

| 执行操作           | 级联优化（Cascades） | 火山模型优化（Volcano） |
|--------------------|:-------------------:|:----------------------:|
| 调整优化标志位      |         ✔           |           ✔            |
| 标准化优化/逻辑优化 |    ✔（标准化优化）    |      ×（逻辑优化）      |
| 检查笛卡尔积        |         ✔           |           ✔            |
| 提取函数依赖        |         ✔           |           ×            |
| 级联优化            |         ✔           |           ×            |
| 物理优化            | ✔（选择最优物理执行计划） | ✔（将逻辑计划转换为物理计划） |
| 后处理优化          |         ✔           |           ✔            |

## `Optimize` 函数流程

`Optimize` 函数是 TiDB 优化器的入口函数，位于 `pkg/planner/optimize.go` 文件中。它负责将 SQL 语句的抽象语法树（AST）转换为可执行的物理执行计划。这个函数处理了从语法分析到最终执行计划生成的整个优化过程，包括查询重写、逻辑优化、物理优化等多个阶段。

### 函数签名

```go
func Optimize(ctx context.Context, sctx sessionctx.Context, node *resolve.NodeW, is infoschema.InfoSchema) (plan base.Plan, slice types.NameSlice, retErr error)
```

参数说明：

- `ctx`：上下文对象，用于跟踪执行过程和传递控制信息
- `sctx`：会话上下文，包含当前会话的各种状态和变量
- `node`：包装后的 AST 节点，表示待优化的 SQL 语句
- `is`：信息模式（InfoSchema），包含数据库的元数据信息

返回值：

- `plan`：生成的物理执行计划
- `slice`：输出列的名称列表
- `retErr`：错误信息，如果优化过程中出现错误

### 执行流程

**步骤 1**：只读模式检查

如果系统处于只读模式（`vardef.RestrictedReadOnly.Load()` 或 `vardef.VarTiDBSuperReadOnly.Load()`），则需要验证当前语句是否允许在只读模式下执行（`allowInReadOnlyMode`）。【TODO:只读模式】

```go
if !sessVars.InRestrictedSQL && (vardef.RestrictedReadOnly.Load() || vardef.VarTiDBSuperReadOnly.Load()) {
    allowed, err := allowInReadOnlyMode(pctx, node.Node)
    if err != nil {
        return nil, nil, err
    }
    if !allowed {
        return nil, nil, errors.Trace(plannererrors.ErrSQLInReadOnlyMode)
    }
}
```

**步骤 2**：设置资源组

如果 SQL 语句中包含资源组相关的提示（hint），系统会设置相应的资源组（`sessVars.StmtCtx.StmtHints.HasResourceGroup`），并检查用户是否有使用指定资源组的权限，只有当用户具备 `RESOURCE_GROUP_ADMIN` 或 `RESOURCE_GROUP_USER` 权限时，才能成功应用资源组设置。

```go
defer func() {
    if retErr == nil {
        // Override the resource group if the hint is set.
        if sessVars.StmtCtx.StmtHints.HasResourceGroup {
            if vardef.EnableResourceControl.Load() {
                hasPriv := true
                // only check dynamic privilege when strict-mode is enabled.
                if vardef.EnableResourceControlStrictMode.Load() {
                    checker := privilege.GetPrivilegeManager(sctx)
                    if checker != nil {
                        hasRgAdminPriv := checker.RequestDynamicVerification(sctx.GetSessionVars().ActiveRoles, "RESOURCE_GROUP_ADMIN", false)
                        hasRgUserPriv := checker.RequestDynamicVerification(sctx.GetSessionVars().ActiveRoles, "RESOURCE_GROUP_USER", false)
                        hasPriv = hasRgAdminPriv || hasRgUserPriv
                    }
                }
                if hasPriv {
                    sessVars.StmtCtx.ResourceGroupName = sessVars.StmtCtx.StmtHints.ResourceGroup
                    // if we are in a txn, should update the txn resource name to let the txn
                    // commit with the hint resource group.
                    if txn, err := sctx.Txn(false); err == nil && txn != nil && txn.Valid() {
                        kv.SetTxnResourceGroup(txn, sessVars.StmtCtx.ResourceGroupName)
                    }
                } else {
                    err := plannererrors.ErrSpecificAccessDenied.GenWithStackByArgs("SUPER or RESOURCE_GROUP_ADMIN or RESOURCE_GROUP_USER")
                    sessVars.StmtCtx.AppendWarning(err)
                }
            } else {
                err := infoschema.ErrResourceGroupSupportDisabled
                sessVars.StmtCtx.AppendWarning(err)
            }
        }
    }
}()
```

> `defer` 是 Go 语言的关键字，用于延迟函数的执行直到当前函数返回。在这里，使用 `defer` 函数确保资源组设置在函数返回前执行，从而保证无论 `Optimize` 函数如何返回（正常返回或发生错误），defer 块中的代码都会被执行。这确保了资源组的设置在整个优化过程结束后才应用，避免中间状态影响优化过程。

**步骤 3**：执行预处理语句

如果是预处理语句，则直接调用 `OptimizeExecStmt` 函数来执行它。`OptimizeExecStmt` 函数位于同一文件 `pkg/planner/optimize.go` 中，专门用于处理 EXECUTE 语句的优化。【TODO:`EXECUTE` 语句执行逻辑】

```go
if _, ok := node.Node.(*ast.ExecuteStmt); ok {
    p, names, err := OptimizeExecStmt(ctx, sctx, node, is)
    return p, names, err
}
```

**步骤 4**：非预处理语句计划缓存

如果启用了非预处理语句的执行计划缓存（`sessVars.EnableNonPreparedPlanCache`），则尝试从缓存中获取执行计划；如果缓存命中，则直接返回缓存的执行计划。

```go
if stmtNode, isStmtNode := node.Node.(ast.StmtNode); sessVars.EnableNonPreparedPlanCache && isStmtNode {
    cachedPlan, names, ok, err := getPlanFromNonPreparedPlanCache(ctx, sctx, stmtNode, is)
    if err != nil {
        return nil, nil, err
    }
    if ok {
        return cachedPlan, names, nil
    }
}
```

**步骤 5**：无缓存优化

如果缓存未命中或不使用缓存，调用 `optimizeNoCache` 函数进行完整的优化过程。

```go
return optimizeNoCache(ctx, sctx, node, is)
```

## `optimizeNoCache` 函数流程

`optimizeNoCache` 函数是实际执行优化的核心部分，也位于 `pkg/planner/optimize.go` 文件中。

**步骤 1**：处理 SQL 提示

提取（`hint.ExtractTableHintsFromStmtNode`）并解析（`hint.ParseStmtHints`）SQL 语句中的提示，并根据提示设置会话变量和优化器行为。【TODO: SQL 提示】

```go
// SQL 提示处理代码
tableHints := hint.ExtractTableHintsFromStmtNode(node.Node, sessVars.StmtCtx)
originStmtHints, _, warns := hint.ParseStmtHints(tableHints,
    setVarHintChecker, hypoIndexChecker(ctx, is),
    sessVars.CurrentDB, byte(kv.ReplicaReadFollower))
sessVars.StmtCtx.StmtHints = originStmtHints
for _, warn := range warns {
    sessVars.StmtCtx.AppendWarning(warn)
}

// 处理 SET_VAR 提示
for name, val := range sessVars.StmtCtx.StmtHints.SetVars {
    oldV, err := sessVars.SetSystemVarWithOldStateAsRet(name, val)
    if err != nil {
        sessVars.StmtCtx.AppendWarning(err)
    }
    sessVars.StmtCtx.AddSetVarHintRestore(name, oldV)
}
```

**步骤 2**：处理严格模式下的 TiFlash

在严格模式下（`sessVars.SQLMode.HasStrictMode()`），对于非只读语句，临时移除 TiFlash 引擎访问权限。【TODO: 严格模式】

```go
if sessVars.SQLMode.HasStrictMode() && !IsReadOnly(node.Node, sessVars) {
    sessVars.StmtCtx.TiFlashEngineRemovedDueToStrictSQLMode = true
    _, hasTiFlashAccess := sessVars.IsolationReadEngines[kv.TiFlash]
    if hasTiFlashAccess {
        delete(sessVars.IsolationReadEngines, kv.TiFlash)
    }
    defer func() {
        sessVars.StmtCtx.TiFlashEngineRemovedDueToStrictSQLMode = false
        if hasTiFlashAccess {
            sessVars.IsolationReadEngines[kv.TiFlash] = struct{}{}
        }
    }()
}
```

**步骤 3**：尝试构造点查快速执行计划

尝试使用 `TryFastPlan` 函数生成点查的快速执行计划，如果成功生成则直接返回。

```go
if _, isolationReadContainTiKV := sessVars.IsolationReadEngines[kv.TiKV]; isolationReadContainTiKV {
    var fp base.Plan
    if fpv, ok := sctx.Value(core.PointPlanKey).(core.PointPlanVal); ok {
        // point plan is already tried in a multi-statement query.
        fp = fpv.Plan
    } else {
        fp = core.TryFastPlan(pctx, node)
    }
    if fp != nil {
        return fp, fp.OutputNames(), nil
    }
}
```

**步骤 4**：绑定执行计划

如果当前会话同时满足如下 3 个条件，则优化器会使用绑定的执行计划：【TODO: 绑定执行计划】

- 启用了计划基线功能（`enableUseBinding`）：允许为 SQL 语句绑定固定的执行计划，以解决执行计划不稳定的问题
- 当前节点为语句节点（`isStmtNode`）：只有语句节点才能绑定执行计划，例如 SELECT、INSERT 等语句
- 是否在绑定库中找到了当前 SQL 语句匹配的执行计划（`match`）

```go
enableUseBinding := sessVars.UsePlanBaselines
stmtNode, isStmtNode := node.Node.(ast.StmtNode)
binding, match, _ := bindinfo.MatchSQLBinding(sctx, stmtNode)

useBinding := enableUseBinding && isStmtNode && match
if isStmtNode {
    // add the extra Limit after matching the bind record
    stmtNode = core.TryAddExtraLimit(sctx, stmtNode)
    node = node.CloneWithNewNode(stmtNode)
}

var (
    names                      types.NameSlice
    bestPlan, bestPlanFromBind base.Plan
    chosenBinding              *bindinfo.Binding
    err                        error
)
if useBinding {
    var bindStmtHints hint.StmtHints
    originHints := hint.CollectHint(stmtNode)
    var warns []error
    if binding != nil && binding.IsBindingEnabled() {
        // 使用绑定中的提示重新优化
        hint.BindHint(stmtNode, binding.Hint)
        // ... 省略部分代码 ...
        plan, curNames, _, err := optimize(ctx, pctx, node, is)
        if err != nil {
            sessVars.StmtCtx.AppendWarning(errors.Errorf("binding %s failed: %v", binding.BindSQL, err))
        }
        bindStmtHints, warns, names, bestPlanFromBind, chosenBinding = curStmtHints, curWarns, curNames, plan, binding
    }
    // ... 省略部分代码 ...
}
```

**步骤 5**：事务预热

事务预热机制通过预先初始化和加载事务所需的资源，减少事务执行时的延迟，提高查询性能。

具体地，调用 `AdviseTxnWarmup` 函数（位于 `pkg/kv/txn.go` 文件中）实现事务的预热处理，为后续的执行计划做好准备。

```go
if err = pctx.AdviseTxnWarmup(); err != nil {
    return nil, nil, err
}
```

**步骤 6**：默认优化

这是 TiDB 常规优化过程的主要入口点。当无法从计划缓存或绑定执行计划中找到合适的执行计划时，系统会通过这一路径进行完整的优化过程。这一步骤确保了即使在没有预定义执行计划的情况下，查询也能得到合理的执行方案。

具体实现上：
- 如果没有从绑定中找到计划，使用原始提示（`originStmtHints`）进行优化
- 调用 `optimize` 函数执行完整的优化过程，包括逻辑优化和物理优化

```go
if bestPlan == nil {
    sessVars.StmtCtx.StmtHints = originStmtHints
    bestPlan, names, _, err = optimize(ctx, pctx, node, is)
    if err != nil {
        return nil, nil, err
    }
}
```

**步骤 7**：计划基线演进

计划基线演进是 TiDB 优化器的一项重要功能，它能够持续评估和改进查询的执行计划。在这一步骤中，系统会在特定条件下（`sessVars.EvolvePlanBaselines` 为 true，有绑定的执行计划 `bestPlanFromBind`，且查询没有 `select_limit` 限制）尝试生成新的执行计划，并与已绑定的计划进行比较。如果新计划不包含存储引擎提示（`HintReadFromStorage`），则可能被接受为改进方案。这种机制使得查询性能能够随着数据变化和系统更新而自动优化。

```go
savedStmtHints := sessVars.StmtCtx.StmtHints
defer func() {
    sessVars.StmtCtx.StmtHints = savedStmtHints
}()
if sessVars.EvolvePlanBaselines && bestPlanFromBind != nil &&
    sessVars.SelectLimit == math.MaxUint64 { // do not evolve this query if sql_select_limit is enabled
    // Check bestPlanFromBind firstly to avoid nil stmtNode.
    if _, ok := stmtNode.(*ast.SelectStmt); ok && !binding.Hint.ContainTableHint(hint.HintReadFromStorage) {
        sessVars.StmtCtx.StmtHints = originStmtHints
        defPlan, _, _, err := optimize(ctx, pctx, node, is)
        if err != nil {
            // Ignore this evolution task.
            return bestPlan, names, nil
        }
        defPlanHints := core.GenHintsFromPhysicalPlan(defPlan)
        for _, h := range defPlanHints {
            if h.HintName.String() == hint.HintReadFromStorage {
                return bestPlan, names, nil
            }
        }
    }
}
```

## `optimize` 函数流程

`optimize` 函数是 TiDB 常规优化过程的入口，位于 `pkg/planner/optimize.go` 文件中。它负责将 AST 节点转换为物理执行计划的核心过程，包括逻辑计划构建、权限验证、逻辑优化和物理优化等多个阶段。这个函数是优化器最核心的部分，会根据系统配置、统计信息和代价模型生成最优的执行计划。`optimize` 函数的调用发生在 `optimizeNoCache` 中，当计划缓存未命中或执行计划需要重新生成时被调用。

**步骤 1**：构建逻辑计划

在这一步骤中，系统首先初始化查询块提示处理器（`hint.NewQBHintHandler`）处理查询提示，然后从对象池中获取一个计划构建器（`PlanBuilder`）实例并进行初始化。最后，调用 `buildLogicalPlan` 函数（位于 `pkg/planner/core/planbuilder.go` 文件中）根据 AST 节点构建初始的逻辑计划。这个阶段是整个优化过程的起点，它将 SQL 的语法结构转换为优化器能够处理的逻辑执行计划。

```go
hintProcessor := hint.NewQBHintHandler(sctx.GetSessionVars().StmtCtx)
node.Node.Accept(hintProcessor)
defer hintProcessor.HandleUnusedViewHints()
builder := planBuilderPool.Get().(*core.PlanBuilder)
defer planBuilderPool.Put(builder.ResetForReuse())
builder.Init(sctx, is, hintProcessor)
p, err := buildLogicalPlan(ctx, sctx, node, builder)
if err != nil {
    return nil, nil, 0, err
}
```

**步骤 2**：权限检查

此步骤执行必要的安全检查，确保用户具有执行当前操作的权限。首先，系统获取当前会话的活跃角色（`ActiveRoles`），然后通过权限管理器（`privilege.GetPrivilegeManager`）检查用户是否有执行当前 SQL 操作所需的权限。接着，系统还会验证表锁（`core.CheckTableLock`）以确保没有冲突的锁定，并检查表的模式（`core.CheckTableMode`）以确保表处于可操作状态。这些安全检查是确保数据库系统安全性和一致性的重要部分。

```go
activeRoles := sessVars.ActiveRoles
if pm := privilege.GetPrivilegeManager(sctx); pm != nil {
    visitInfo := core.VisitInfo4PrivCheck(ctx, is, node.Node, builder.GetVisitInfo())
    if err := core.CheckPrivilege(activeRoles, pm, visitInfo); err != nil {
        return nil, nil, 0, err
    }
}

if err := core.CheckTableLock(sctx, is, builder.GetVisitInfo()); err != nil {
    return nil, nil, 0, err
}

if err := core.CheckTableMode(node); err != nil {
    return nil, nil, 0, err
}
```

**步骤 3**：逻辑优化与物理优化

这是优化过程的核心阶段，系统将初步构建的逻辑计划转换为最终的物理执行计划。首先，获取逻辑计划的输出列名称（`OutputNames`），然后检查当前计划是否为逻辑计划（`isLogicalPlan`）。如果不是逻辑计划，则直接返回；否则，调用 `core.RecheckCTE` 检查公共表表达式，并使用 `core.DoOptimize` 函数（位于 `pkg/planner/core/optimizer.go` 文件中）执行完整的逻辑优化和物理优化过程。在这个过程中，系统会应用各种优化规则，估计不同执行路径的代价，并选择代价最低的物理执行计划。同时，系统也记录了优化所花费的时间（`DurationOptimization`），用于性能监控。

```go
names := p.OutputNames()

// Handle the non-logical plan statement.
logic, isLogicalPlan := p.(base.LogicalPlan)
if !isLogicalPlan {
    return p, names, 0, nil
}

core.RecheckCTE(logic)

beginOpt := time.Now()
finalPlan, cost, err := core.DoOptimize(ctx, sctx, builder.GetOptFlag(), logic)
// TODO: capture plan replayer here if it matches sql and plan digest

sessVars.DurationOptimization = time.Since(beginOpt)
return finalPlan, names, cost, err
```

## `DoOptimize` 函数流程

`DoOptimize` 函数位于 `pkg/planner/core/optimizer.go` 文件中，是逻辑优化到物理优化的衔接点。它接收逻辑计划并通过一系列优化后返回物理执行计划。

```go
func DoOptimize(ctx context.Context, sctx sessionctx.Context, flag uint64, logic base.LogicalPlan) (base.Plan, float64, error) {
    // 省略错误处理代码...
    
    // 调用 doOptimize 函数进行实际优化
    flag |= flagUseNewCostInterface
    p, cost, err := doOptimize(ctx, sctx, flag, logic)
    return p, cost, err
}
```

`DoOptimize` 函数主要是一个包装函数，它添加了一些标志位（例如 `flagUseNewCostInterface`），然后调用 `doOptimize` 函数来执行实际的优化工作。这种设计模式允许在主优化流程外部添加一些额外的处理逻辑，如错误处理和标志设置，使核心优化代码保持清晰。

在启用级联优化器（`EnableCascadesPlanner` 为 true）的情况下，调用 `CascadesOptimize` 函数，主要执行基于规则的转换和成本模型的搜索。在默认情况下（`EnableCascadesPlanner` 为 false），调用 `VolcanoOptimize` 函数，主要执行传统的火山模型优化。

## `doOptimize` 函数流程

`doOptimize` 函数是 TiDB 优化器的核心实现，负责执行实际的优化过程，位于 `pkg/planner/core/optimizer.go` 文件中。它根据会话变量中的 `EnableCascadesPlanner` 设置来决定使用哪种优化器。

```go
func doOptimize(ctx context.Context, sctx base.PlanContext, flag uint64, logic base.LogicalPlan) (
	base.LogicalPlan, base.PhysicalPlan, float64, error) {
	if sctx.GetSessionVars().GetSessionVars().EnableCascadesPlanner {
		return CascadesOptimize(ctx, sctx, flag, logic)
	}
	return VolcanoOptimize(ctx, sctx, flag, logic)
}
```

## `CascadesOptimize` 函数流程

`CascadesOptimize` 函数是 TiDB 中实现级联优化器模型的入口，位于 `pkg/planner/core/optimizer.go` 文件中。

级联优化器是一种更现代化的查询优化方法，它将规则驱动和成本驱动的优化方法结合起来，通过规则变换生成等价的执行计划，并通过成本模型选择最优方案。

其中包括：

- 调整优化标志位（`adjustOptimizationFlags`）
- 标准化优化（`normalization`）
- 检查笛卡尔积（`existsCartesianProduct`）
- 级联优化（`cascadesOptimize`）
- 物理优化（physicalOptimize）：选择最优物理计划
- 后处理优化（`postOptimize`）

**步骤 1**：调整优化标志位

函数首先获取会话变量，并通过 `adjustOptimizationFlags` 函数调整优化标志，以适应当前的逻辑计划特性。

```go
sessVars := sctx.GetSessionVars()
flag = adjustOptimizationFlags(flag, logic)
```

**步骤 2**：标准化优化

调用 `normalizeOptimize` 函数对逻辑计划进行标准化处理，使其符合级联优化器的要求。

```go
logic, err := normalizeOptimize(ctx, flag, logic)
if err != nil {
    return nil, nil, 0, err
}
```

**步骤 3**：检查笛卡尔积

如果系统配置不允许笛卡尔积查询（`AllowCartesianProduct.Load()` 为 false），且当前逻辑计划中存在笛卡尔积（`existsCartesianProduct(logic)` 为 true），则返回错误。

```go
if !AllowCartesianProduct.Load() && existsCartesianProduct(logic) {
    return nil, nil, 0, errors.Trace(plannererrors.ErrCartesianProductUnsupported)
}
```

**步骤 4**：提取函数依赖

调用逻辑计划的 `ExtractFD` 方法提取函数依赖关系，这些依赖关系将用于后续的优化过程。

```go
logic.ExtractFD()
```

**步骤 5**：执行级联优化

创建级联优化器实例（`cascades.NewOptimizer`），并执行优化过程（`cas.Execute()`）。同时设置延迟销毁（`defer cas.Destroy()`）以确保资源正确释放。

```go
var cas *cascades.Optimizer
if cas, err = cascades.NewOptimizer(logic); err == nil {
    defer cas.Destroy()
    err = cas.Execute()
}
if err != nil {
    return nil, nil, 0, err
}
```

**步骤 6**：选择最优物理计划

通过迭代级联优化器的备忘录（Memo）中的所有逻辑计划，对每个逻辑计划调用 `physicalOptimize` 函数转换为物理计划，并选择代价最低的物理计划。

```go
var (
    physical base.PhysicalPlan
    cost     = math.MaxFloat64
)
cas.GetMemo().NewIterator().Each(func(oneLogic base.LogicalPlan) bool {
    planCounter := base.PlanCounterTp(sessVars.StmtCtx.StmtHints.ForceNthPlan)
    if planCounter == 0 {
        planCounter = -1
    }
    tmpPhysical, tmpCost, tmpErr := physicalOptimize(oneLogic, &planCounter)
    if tmpErr != nil {
        err = tmpErr
        return false
    }
    if tmpCost < cost {
        physical = tmpPhysical
        cost = tmpCost
    }
    return true
})
if err != nil {
    return nil, nil, 0, err
}
```

**步骤 7**：执行后优化

对选出的物理计划进行后优化处理（`postOptimize`），并根据会话变量设置记录优化跟踪信息。

```go
finalPlan := postOptimize(ctx, sctx, physical)
if sessVars.StmtCtx.EnableOptimizerCETrace {
    refineCETrace(sctx)
}
if sessVars.StmtCtx.EnableOptimizeTrace {
    sessVars.StmtCtx.OptimizeTracer.RecordFinalPlan(finalPlan.BuildPlanTrace())
}
```

## `VolcanoOptimize` 函数流程

`VolcanoOptimize` 函数是 TiDB 中传统火山模型优化器的实现，位于 `pkg/planner/core/optimizer.go` 文件中。

火山优化器模型最早由 Graefe 提出，是关系数据库中广泛使用的优化器模型，它采用动态规划和贪婪算法结合的方式为查询选择最佳执行计划。

其中包括：

- 调整优化标志位（`adjustOptimizationFlags`）
- 逻辑优化（`logicalOptimize`）
- 检查笛卡尔积（`existsCartesianProduct`）
- 物理优化（`physicalOptimize`）
- 后处理优化（`postOptimize`）

**步骤 1**：调整优化标志位

获取会话变量，并通过 `adjustOptimizationFlags` 函数调整优化标志，以适应当前的逻辑计划特性。

```go
sessVars := sctx.GetSessionVars()
flag = adjustOptimizationFlags(flag, logic)
```

**步骤 2**：逻辑优化

调用 `logicalOptimize` 函数对逻辑计划进行优化，应用各种逻辑优化规则（如谓词下推、列剪裁等）。

```go
logic, err := logicalOptimize(ctx, flag, logic)
if err != nil {
    return nil, nil, 0, err
}
```

**步骤 3**：检查笛卡尔积

如果系统配置不允许笛卡尔积查询（`AllowCartesianProduct.Load()` 为 false），且当前逻辑计划中存在笛卡尔积（`existsCartesianProduct(logic)` 为 true），则返回错误。

```go
if !AllowCartesianProduct.Load() && existsCartesianProduct(logic) {
    return nil, nil, 0, errors.Trace(plannererrors.ErrCartesianProductUnsupported)
}
```

**步骤 4**：物理优化

设置计划计数器（`planCounter`），然后调用 `physicalOptimize` 函数将逻辑计划转换为物理执行计划，同时计算计划的执行代价。

```go
planCounter := base.PlanCounterTp(sessVars.StmtCtx.StmtHints.ForceNthPlan)
if planCounter == 0 {
    planCounter = -1
}
physical, cost, err := physicalOptimize(logic, &planCounter)
if err != nil {
    return nil, nil, 0, err
}
```

**步骤 5**：执行后优化

对物理计划进行后优化处理（`postOptimize`），并根据会话变量设置记录优化跟踪信息。

```go
finalPlan := postOptimize(ctx, sctx, physical)

if sessVars.StmtCtx.EnableOptimizerCETrace {
    refineCETrace(sctx)
}
if sessVars.StmtCtx.EnableOptimizeTrace {
    sessVars.StmtCtx.OptimizeTracer.RecordFinalPlan(finalPlan.BuildPlanTrace())
}
```
