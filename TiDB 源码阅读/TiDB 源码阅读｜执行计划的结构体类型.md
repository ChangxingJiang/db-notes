# TiDB 源码阅读｜执行计划的结构体类型

在 TiDB 优化器框架中，`Plan` 结构体是最基础的执行计划接口实现，被 `BaseLogicalPlan` 和 `BasePhysicalPlan` 结构体所嵌入。`BaseLogicalPlan` 代表逻辑执行计划，用于查询优化阶段的逻辑算子表示；`BasePhysicalPlan` 代表物理执行计划，用于最终执行阶段的物理算子表示。

## Plan 结构体：基础执行计划

`Plan` 结构体定义在 `pkg/planner/core/operator/baseimpl/plan.go` 文件中，它是所有执行计划的基础结构，提供了执行计划所需的通用属性和方法，包括上下文信息、统计信息、计划类型和 ID 等基本数据。这些基础属性被所有类型的执行计划共享。

```go
// Plan Should be used as embedded struct in Plan implementations.
type Plan struct {
	ctx     planctx.PlanContext
	stats   *property.StatsInfo `plan-cache-clone:"shallow"`
	tp      string
	id      int
	qbBlock int // Query Block offset
}
```

- `ctx`：类型为 `planctx.PlanContext`，存储计划执行的上下文信息，用于访问会话变量和其他环境数据
- `stats`：类型为 `*property.StatsInfo`，保存计划的统计信息，标记为浅克隆（`plan-cache-clone:"shallow"`）
- `tp`：类型为 `string`，表示计划类型的字符串标识
- `id`：类型为 `int`，计划的唯一标识符
- `qbBlock`：类型为 `int`，表示查询块（Query Block）的偏移量

## BaseLogicalPlan 结构体：逻辑执行计划

`BaseLogicalPlan` 结构体定义在 `pkg/planner/core/operator/logicalop/base_logical_plan.go` 文件中，它嵌入了 `Plan` 结构体，并扩展了逻辑计划特有的属性和方法。`BaseLogicalPlan` 是所有逻辑执行计划的基类，用于支持查询优化过程中的逻辑算子表示和转换。

```go
// BaseLogicalPlan is the common structure that used in logical plan.
type BaseLogicalPlan struct {
	baseimpl.Plan

	taskMap map[string]base.Task
	// taskMapBak forms a backlog stack of taskMap, used to roll back the taskMap.
	taskMapBak []string
	// planIDsHash is the hash of the subtree root from this logical plan.
	planIDsHash uint64
	// taskMapBakTS stores the timestamps of logs.
	taskMapBakTS []uint64
	self         base.LogicalPlan
	maxOneRow    bool
	children     []base.LogicalPlan
	// fdSet is a set of functional dependencies(FDs) which powers many optimizations,
	// including eliminating unnecessary DISTINCT operators, simplifying ORDER BY columns,
	// removing Max1Row operators, and mapping semi-joins to inner-joins.
	// for now, it's hard to maintain in individual operator, build it from bottom up when using.
	fdSet *fd.FDSet

	// Flag is with that each bit has its meaning to mark this logical plan for special handling.
	Flag uint64
}
```

- `taskMap`：类型为 `map[string]base.Task`，用于记录不同属性对应的执行任务
- `taskMapBak`：类型为 `[]string`，taskMap 的备份堆栈，用于回滚 taskMap
- `planIDsHash`：类型为 `uint64`，从该逻辑计划子树根节点计算的哈希值
- `taskMapBakTS`：类型为 `[]uint64`，存储日志的时间戳
- `self`：类型为 `base.LogicalPlan`，指向实现该基础计划的具体逻辑计划
- `maxOneRow`：类型为 `bool`，标识该计划是否最多只会返回一行数据
- `children`：类型为 `[]base.LogicalPlan`，该计划的子计划列表
- `fdSet`：类型为 `*fd.FDSet`，函数依赖集合，用于各种优化操作
- `Flag`：类型为 `uint64`，用于标记特殊处理的位标志

## BasePhysicalPlan 结构体：物理执行计划

`BasePhysicalPlan` 结构体定义在 `pkg/planner/core/operator/physicalop/base_physical_plan.go` 文件中，它同样嵌入了 `Plan` 结构体，并扩展了物理计划特有的属性和方法。`BasePhysicalPlan` 是所有物理执行计划的基类，用于支持查询执行阶段的物理算子实现。

```go
// BasePhysicalPlan is the common structure that used in physical plan.
type BasePhysicalPlan struct {
	baseimpl.Plan

	childrenReqProps []*property.PhysicalProperty `plan-cache-clone:"shallow"`
	Self             base.PhysicalPlan
	children         []base.PhysicalPlan

	// used by the new cost interface
	PlanCostInit bool
	PlanCost     float64
	PlanCostVer2 costusage.CostVer2 `plan-cache-clone:"shallow"`

	// probeParents records the IndexJoins and Applys with this operator in their inner children.
	// Please see comments in op.PhysicalPlan for details.
	probeParents []base.PhysicalPlan `plan-cache-clone:"shallow"`

	// Only for MPP. If TiFlashFineGrainedShuffleStreamCount > 0:
	// 1. For ExchangeSender, means its output will be partitioned by hash key.
	// 2. For ExchangeReceiver/Window/Sort, means its input is already partitioned.
	TiFlashFineGrainedShuffleStreamCount uint64
}
```

- `childrenReqProps`：类型为 `[]*property.PhysicalProperty`，子计划的物理属性要求，标记为浅克隆（`plan-cache-clone:"shallow"`）
- `Self`：类型为 `base.PhysicalPlan`，指向实现该基础计划的具体物理计划
- `children`：类型为 `[]base.PhysicalPlan`，该计划的子计划列表
- `PlanCostInit`：类型为 `bool`，表示计划成本是否已初始化
- `PlanCost`：类型为 `float64`，存储计划的执行成本
- `PlanCostVer2`：类型为 `costusage.CostVer2`，计划成本的第二个版本，标记为浅克隆（`plan-cache-clone:"shallow"`）
- `probeParents`：类型为 `[]base.PhysicalPlan`，记录将该算子作为内部子节点的 IndexJoins 和 Applys，标记为浅克隆（`plan-cache-clone:"shallow"`）
- `TiFlashFineGrainedShuffleStreamCount`：类型为 `uint64`，仅用于 MPP 模式，表示 TiFlash 细粒度 Shuffle 的流数量 
