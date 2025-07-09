# TiDB 源码阅读｜执行计划类型结构

TiDB 中的执行计划接口位于 `pkg/planner/core/base/plan_base.go` 文件中，包含如下三个层次：

1. **`Plan` 接口**：所有执行计划的基础接口，定义了执行计划的通用方法
2. **`LogicalPlan` 接口**：继承自 `Plan` 接口，定义了逻辑执行计划特有的方法，主要用于逻辑优化阶段
3. **`PhysicalPlan` 接口**：继承自 `Plan` 接口，定义了物理执行计划特有的方法，主要用于物理优化和执行阶段

## 执行计划的基础接口

`Plan` 接口定义了逻辑执行计划和物理执行计划的通用方法：

- **`Schema()`**：获取执行计划的 schema 信息，返回 `*expression.Schema`
- **`ID()`**：获取执行计划的唯一标识符，返回 `int`
- **`TP(...bool)`**：获取执行计划的类型，返回 `string`
- **`ExplainID(isChildOfINL ...bool)`**：获取在 `EXPLAIN` 语句中的标识符，返回 `fmt.Stringer`
- **`ExplainInfo()`**：返回需要被解释的算子信息，返回 `string`
- **`ReplaceExprColumns(replace map[string]*expression.Column)`**：替换计划表达式节点中的所有列引用，无返回值
- **`SCtx()`**：获取计划上下文，返回 `PlanContext`
- **`StatsInfo()`**：返回该计划的统计信息，返回 `*property.StatsInfo`
- **`OutputNames()`**：返回每列的输出名称，返回 `types.NameSlice`
- **`SetOutputNames(names types.NameSlice)`**：设置输出列名，无返回值
- **`QueryBlockOffset()`**：获取查询块偏移量，用于处理子查询中的提示，返回 `int`
- **`BuildPlanTrace()`**：构建计划跟踪信息，返回 `*tracing.PlanTrace`
- **`CloneForPlanCache(newCtx PlanContext)`**：为计划缓存克隆该计划，返回 `(Plan, bool)`

其中涉及的重要类型包括：

- **`types.NameSlice`**：在 `pkg/types/field_name.go` 中定义，是 `[]*FieldName` 类型的切片，用于存储列名信息。每个 `FieldName` 包含原始表名、原始列名、数据库名、表名和列名等信息，主要用于 MySQL 协议中标识字段，并提供了查找列名、内存使用计算等功能。

- **`PlanContext`**：在 `pkg/planner/planctx/context.go` 中定义的接口，提供了执行计划构建过程中所需的上下文信息。它包含获取 SQL 执行器、表达式上下文、会话变量、信息模式、事务等方法，是执行计划与底层存储、会话状态交互的桥梁。

- **`property.StatsInfo`**：在 `pkg/planner/property/stats_info.go` 中定义的结构体，存储计划输出的基本统计信息，用于成本估算。它包含行数（RowCount）、列的 NDV（不同值的数量）、直方图集合等信息，并提供了缩放统计信息、获取列组 NDV 等方法，是优化器进行成本估算的重要依据。

## 物理执行计划接口

`PhysicalPlan` 接口继承自 `Plan` 接口，额外定义了物理执行计划特有的方法：

- **`GetPlanCostVer1(taskType property.TaskType, option *optimizetrace.PlanCostOption)`**：计算并返回计划在模型版本 1 上的成本，返回 `(float64, error)`
- **`GetPlanCostVer2(taskType property.TaskType, option *optimizetrace.PlanCostOption, isChildOfINL ...bool)`**：计算并返回计划在模型版本 2 上的成本，返回 `(costusage.CostVer2, error)`
- **`Attach2Task(...Task)`**：将当前物理计划作为任务的父节点，并更新当前任务的成本，返回 `Task`
- **`ToPB(ctx *BuildPBContext, storeType kv.StoreType)`**：将物理计划转换为 TiPB 执行器，返回 `(*tipb.Executor, error)`
- **`GetChildReqProps(idx int)`**：获取子节点的所需属性，返回 `*property.PhysicalProperty`
- **`StatsCount()`**：返回该计划的统计信息计数，返回 `float64`
- **`ExtractCorrelatedCols()`**：提取物理计划内的相关列，返回 `[]*expression.CorrelatedColumn`
- **`Children()`**：获取所有子节点，返回 `[]PhysicalPlan`
- **`SetChildren(...PhysicalPlan)`**：设置计划的子节点，无返回值
- **`SetChild(i int, child PhysicalPlan)`**：设置计划的第 i 个子节点，无返回值
- **`ResolveIndices()`**：解析列的索引，使列能够通过索引评估行，返回 `error`
- **`StatsInfo()`**：返回计划的统计信息，返回 `*property.StatsInfo`
- **`SetStats(s *property.StatsInfo)`**：设置基础物理计划中的统计信息，无返回值
- **`ExplainNormalizedInfo()`**：返回用于生成摘要的算子标准化信息，返回 `string`
- **`Clone(newCtx PlanContext)`**：克隆该物理计划，返回 `(PhysicalPlan, error)`
- **`AppendChildCandidate(op *optimizetrace.PhysicalOptimizeOp)`**：将子物理计划添加到跟踪器中，无返回值
- **`MemoryUsage()`**：返回物理计划的内存使用量，返回 `int64`
- **`SetProbeParents([]PhysicalPlan)`**：设置 `probeParents` 字段，用于处理统计信息中行数的不一致性，无返回值
- **`GetEstRowCountForDisplay()`**：使用 StatsInfo 中的"单探测"行数和 probeParents 计算"所有探测"行数，返回 `float64`
- **`GetActualProbeCnt(*execdetails.RuntimeStatsColl)`**：使用运行时统计信息和 probeParents 计算实际的"探测"计数，返回 `int64`

其中涉及的重要类型包括：

- **`property.PhysicalProperty`**：在 `pkg/planner/property/physical_property.go` 中定义的结构体，表示父节点对子节点的物理属性要求。它包含排序项（SortItems）、任务类型（TaskTp）、期望行数（ExpectedCnt）、MPP 分区列和分区类型等信息，是物理优化阶段进行计划选择和成本估算的关键依据，并提供了属性比较、哈希计算等方法。

- **`optimizetrace.PhysicalOptimizeOp`**：在 `pkg/planner/util/optimizetrace/opt_tracer.go` 中定义的结构体，用于物理优化过程的追踪记录。它封装了 `tracing.PhysicalOptimizeTracer`，提供了添加候选计划、获取追踪器等方法，帮助记录和分析物理优化过程中的决策和计划转换，对于优化器调试和性能分析非常重要。

## 逻辑执行计划接口

`LogicalPlan` 接口继承自 `Plan` 接口，额外定义了逻辑执行计划特有的方法：

- **`HashCode()`**：编码逻辑计划以快速比较是否相等，返回 `[]byte`
- **`PredicatePushDown([]expression.Expression, *optimizetrace.LogicalOptimizeOp)`**：尽可能深入地下推 where/on/having 子句中的谓词，返回 `([]expression.Expression, LogicalPlan)`
- **`PruneColumns([]*expression.Column, *optimizetrace.LogicalOptimizeOp)`**：剪裁未使用的列，如有变更则返回新的逻辑计划，返回 `(LogicalPlan, error)`
- **`FindBestTask(prop *property.PhysicalProperty, planCounter *PlanCounterTp, op *optimizetrace.PhysicalOptimizeOp)`**：将逻辑计划转换为物理计划，返回 `(Task, int64, error)`
- **`BuildKeyInfo(selfSchema *expression.Schema, childSchema []*expression.Schema)`**：收集唯一键信息到 schema 中，无返回值
- **`PushDownTopN(topN LogicalPlan, opt *optimizetrace.LogicalOptimizeOp)`**：在逻辑优化期间下推 topN 或 limit 算子，返回 `LogicalPlan`
- **`DeriveTopN(opt *optimizetrace.LogicalOptimizeOp)`**：从 row_number 窗口函数的过滤器派生隐式 TopN，返回 `LogicalPlan`
- **`PredicateSimplification(opt *optimizetrace.LogicalOptimizeOp)`**：合并列及其等价类上的不同谓词，返回 `LogicalPlan`
- **`ConstantPropagation(parentPlan LogicalPlan, currentChildIdx int, opt *optimizetrace.LogicalOptimizeOp)`**：根据列等价关系生成新的常量谓词，返回 `LogicalPlan`
- **`PullUpConstantPredicates()`**：递归查找常量谓词，用于常量传播规则，返回 `[]expression.Expression`
- **`RecursiveDeriveStats(colGroups [][]*expression.Column)`**：在计划间递归派生统计信息，返回 `(*property.StatsInfo, bool, error)`
- **`DeriveStats(childStats []*property.StatsInfo, selfSchema *expression.Schema, childSchema []*expression.Schema, reloads []bool)`**：给定子统计信息，为当前计划节点派生统计信息，返回 `(*property.StatsInfo, bool, error)`
- **`ExtractColGroups(colGroups [][]*expression.Column)`**：从子算子中提取当前算子所需的列组，返回 `[][]*expression.Column`
- **`PreparePossibleProperties(schema *expression.Schema, childrenProperties ...[][]*expression.Column)`**：获取所有可能的顺序属性，返回 `[][]*expression.Column`
- **`ExhaustPhysicalPlans(*property.PhysicalProperty)`**：生成所有可能匹配所需属性的计划，返回 `([]PhysicalPlan, bool, error)`
- **`ExtractCorrelatedCols()`**：提取逻辑计划内的相关列，返回 `[]*expression.CorrelatedColumn`
- **`MaxOneRow()`**：表示该算子是否最多只返回一行，返回 `bool`
- **`Children()`**：获取所有子节点，返回 `[]LogicalPlan`
- **`SetChildren(...LogicalPlan)`**：设置计划的子节点，无返回值
- **`SetChild(i int, child LogicalPlan)`**：设置计划的第 i 个子节点，无返回值
- **`RollBackTaskMap(TS uint64)`**：回滚时间戳 TS 之后的所有 taskMap 日志，无返回值
- **`CanPushToCop(store kv.StoreType)`**：检查是否可以将此计划推送到特定存储（已弃用），返回 `bool`
- **`ExtractFD()`**：自底向上派生功能依赖集，返回 `*fd.FDSet`
- **`GetBaseLogicalPlan()`**：返回每个逻辑计划内部的基础逻辑计划，返回 `LogicalPlan`
- **`ConvertOuterToInnerJoin(predicates []expression.Expression)`**：如果匹配的行被过滤，则将外连接转换为内连接，返回 `LogicalPlan`
- **`SetPlanIDsHash(uint64)`**：设置子算子树的 ID 哈希值，无返回值
- **`GetPlanIDsHash()`**：获取子算子树的 ID 哈希值，返回 `uint64`
- **`GetWrappedLogicalPlan()`**：返回组表达式中包装的逻辑计划，返回 `LogicalPlan`

其中涉及的重要类型包括：

- **`optimizetrace.LogicalOptimizeOp`**：在 `pkg/planner/util/optimizetrace/opt_tracer.go` 中定义的结构体，用于逻辑优化过程的追踪记录。它封装了 `tracing.LogicalOptimizeTracer`，提供了记录规则优化前后的计划状态、追加优化步骤、记录最终逻辑计划等方法，帮助开发者理解和分析逻辑优化过程中各个规则的应用效果和计划变化，对于优化器的调试和性能分析至关重要。
