# 构造逻辑执行计划：处理 GROUP BY 子句

## `buildSelect` 函数处理 `GROUP BY` 子句的流程

`buildSelect` 函数用于构建 `SELECT` 语句的逻辑执行计划，定义在 `tidb/pkg/planner/core/logical_plan_builder.go` 中。在处理 `GROUP BY` 子句时，除重写表达式外不额外改变逻辑执行计划算子。主要分为以下三个步骤：

**步骤 1**：调用 `resolveGbyExprs` 函数解析 `GROUP BY` 子句中的表达式，获取解析后的表达式列表 `gbyExprs`。

```go
var gbyExprs []ast.ExprNode
if sel.GroupBy != nil {
    gbyExprs, err = b.resolveGbyExprs(p, sel.GroupBy, sel.Fields.Fields)
    if err != nil {
        return nil, err
    }
}
```

**步骤 2**：如果启用了 `ONLY_FULL_GROUP_BY` 模式，则调用 `checkOnlyFullGroupBy` 函数检查是否满足 `ONLY_FULL_GROUP_BY` 模式要求。

```go
if b.ctx.GetSessionVars().SQLMode.HasOnlyFullGroupBy() && sel.From != nil && !b.ctx.GetSessionVars().OptimizerEnableNewOnlyFullGroupByCheck {
    err = b.checkOnlyFullGroupBy(p, sel)
    if err != nil {
        return nil, err
    }
}
```

**步骤 3**：调用 `rewriteGbyExprs` 函数将 `GROUP BY` 表达式重写为可执行的表达式。

```go
if sel.GroupBy != nil {
    p, gbyCols, rollup, err = b.rewriteGbyExprs(ctx, p, sel.GroupBy, gbyExprs)
    if err != nil {
        return nil, err
    }
}
```

## `resolveGbyExprs` 函数处理流程

`resolveGbyExprs` 函数用于解析 GROUP BY 子句中的表达式，将其转换为可以处理的形式（`tidb/pkg/planner/core/logical_plan_builder.go`）。

**步骤 1**：获取逻辑计划的 Schema 和输出字段

```go
schema := p.Schema()
names := p.OutputNames()
if join, ok := p.(*logicalop.LogicalJoin); ok && join.FullSchema != nil {
    schema = join.FullSchema
    names = join.FullNames
}
```

**步骤 2**：创建 `gbyResolver` 解析器

函数创建一个 `gbyResolver` 解析器，用于解析 `GROUP BY` 表达式。该解析器包含上下文、字段、`schema`、`names` 和 `skipAggMap` 等信息。

```go
resolver := &gbyResolver{
    ctx:        b.ctx,
    fields:     fields,
    schema:     schema,
    names:      names,
    skipAggMap: b.correlatedAggMapper,
}
```

**步骤 3**：遍历 `GROUP BY` 子句中的每一项，使用解析器对表达式进行解析，并将解析结果添加到结果列表 `exprs` 中。

```go
exprs := make([]ast.ExprNode, 0, len(gby.Items))
for _, item := range gby.Items {
    resolver.inExpr = false
    resolver.exprDepth = 0
    resolver.isParam = false
    retExpr, _ := item.Expr.Accept(resolver)
    if resolver.err != nil {
        return exprs, errors.Trace(resolver.err)
    }
    if !resolver.isParam {
        item.Expr = retExpr.(ast.ExprNode)
    }

    exprs = append(exprs, retExpr.(ast.ExprNode))
}
```

**步骤 4**：返回解析后的 `GROUP BY` 表达式列表。

```go
return exprs, nil
```

## `checkOnlyFullGroupBy` 函数处理流程

在 MySQL 5.7.5 及以版本中，在启用 `ONLY_FULL_GROUP_BY` 模式的情况下，也允许 `GROUP BY` 子句中未命名的非聚合列在 `SELECT` 列表、`HAVING` 条件或 `ORDER BY` 列表中被使用，前提是该列的值是唯一确定的，可以从 `WHERE` 子句中提取到这些单值列。例如：

```sql
SELECT a, SUM(b) FROM mytable WHERE a = 'abc';
```

然而在 `ONLY_FULL_GROUP_BY` 模式下，以下查询页可能是无效的，因为 `SELECT` 列表中的非聚合列 `address` 并没有出现在 `GROUP BY` 子句中：

```sql
SELECT name, address, MAX(age) FROM t GROUP BY name;
```

关于 `ONLY_FULL_GROUP_BY` 模式的更多细节可以查看 MySQL 文档：[https://dev.mysql.com/doc/refman/5.7/en/group-by-handling.html](https://dev.mysql.com/doc/refman/5.7/en/group-by-handling.html)

`checkOnlyFullGroupBy` 函数用于检查查询是否符合 `ONLY_FULL_GROUP_BY` 模式的要求（`tidb/pkg/planner/core/logical_plan_builder.go`）。该函数根据是否存在 `GROUP BY` 子句，调用不同的处理函数：

```go
func (b *PlanBuilder) checkOnlyFullGroupBy(p base.LogicalPlan, sel *ast.SelectStmt) (err error) {
	if sel.GroupBy != nil {
		err = b.checkOnlyFullGroupByWithGroupClause(p, sel)
	} else {
		err = b.checkOnlyFullGroupByWithOutGroupClause(p, sel)
	}
	return err
}
```

## `checkOnlyFullGroupByWithGroupClause` 函数处理流程

`checkOnlyFullGroupByWithGroupClause` 函数用于检查带有 `GROUP BY` 子句的查询是否满足 `ONLY_FULL_GROUP_BY` 模式的要求（`tidb/pkg/planner/core/logical_plan_builder.go`）。

**步骤 1**：收集 `GROUP BY` 子句中的列和表达式，将列名存储到 `gbyOrSingleValueColNames` 作为可以在 `SELECT` 列表、`HAVING`条件和 `ORDER BY` 列表中被直接使用的列名，将非列名表达式存储到 `gbyExprs` 中。

```go
gbyOrSingleValueColNames := make(map[*types.FieldName]struct{}, len(sel.Fields.Fields))
gbyExprs := make([]ast.ExprNode, 0, len(sel.Fields.Fields))
for _, byItem := range sel.GroupBy.Items {
    expr := getInnerFromParenthesesAndUnaryPlus(byItem.Expr)
    if colExpr, ok := expr.(*ast.ColumnNameExpr); ok {
        addGbyOrSingleValueColName(p, colExpr.Name, gbyOrSingleValueColNames)
    } else {
        gbyExprs = append(gbyExprs, expr)
    }
}
```

**步骤 2**：调用 `extractSingeValueColNamesFromWhere` 函数，从 `WHERE` 子句中提取单值列，将单值列补充到可以被直接使用的列名中。

```go
extractSingeValueColNamesFromWhere(p, sel.Where, gbyOrSingleValueColNames)
```

**步骤 3**：遍历 `SELECT` 列表中的表达式，调用 `checkExprInGroupByOrIsSingleValue` 函数检查它们是否属于如下任一场景：

- 聚合函数
- 存在于 `GROUP BY` 子句中的聚合列
- 存在于 `WHERE` 子句中的单值列

如果某个表达式不满足这些条件，会将其添加到 `notInGbyOrSingleValueColNames` 集合中。

```go
notInGbyOrSingleValueColNames := make(map[*types.FieldName]ErrExprLoc, len(sel.Fields.Fields))
for offset, field := range sel.Fields.Fields {
    if field.Auxiliary {
        continue
    }
    checkExprInGroupByOrIsSingleValue(p, getInnerFromParenthesesAndUnaryPlus(field.Expr), offset, ErrExprInSelect, gbyOrSingleValueColNames, gbyExprs, notInGbyOrSingleValueColNames)
}
```

**步骤 4**：遍历 `ORDER BY` 子句中的表达式，调用 `checkExprInGroupByOrIsSingleValue` 函数检查它们，条件于 `SELECT` 列表中的表达式相同。如果某个 `ORDER BY` 项已经在 `SELECT` 列表中被检查过了，则跳过检查。

```go
if sel.OrderBy != nil {
    for offset, item := range sel.OrderBy.Items {
        if colName, ok := item.Expr.(*ast.ColumnNameExpr); ok {
            index, err := resolveFromSelectFields(colName, sel.Fields.Fields, false)
            if err != nil {
                return err
            }
            // If the ByItem is in fields list, it has been checked already in above.
            if index >= 0 {
                continue
            }
        }
        checkExprInGroupByOrIsSingleValue(p, getInnerFromParenthesesAndUnaryPlus(item.Expr), offset, ErrExprInOrderBy, gbyOrSingleValueColNames, gbyExprs, notInGbyOrSingleValueColNames)
    }
}
```

**步骤 5**：如果没有找到违反 `ONLY_FULL_GROUP_BY` 规则的表达式，函数直接返回。

```go
if len(notInGbyOrSingleValueColNames) == 0 {
    return nil
}
```

**步骤 6**：如果存在不满足条件的表达式，函数会尝试通过函数依赖关系来解决问题。调用 `buildWhereFuncDepend` 函数构建 `WHERE` 子句的函数依赖关系，调用 `buildJoinFuncDepend` 函数构建 `JOIN` 条件的函数依赖关系。

```go
whereDepends, err := buildWhereFuncDepend(p, sel.Where)
if err != nil {
    return err
}
joinDepends, err := b.buildJoinFuncDepend(p, sel.From.TableRefs)
if err != nil {
    return err
}
```

**步骤 7**：检查不满足步骤 3 和步骤 4 条件的 `notInGbyOrSingleValueColNames` 集合中的表达式，调用 `checkColFuncDepend` 函数检查其是否通过函数依赖关系于 `GROUP BY` 列关联。如果存在依赖关系，则该列被视为有效，否则被视为无效。

```go
tblMap := make(map[*model.TableInfo]struct{}, len(notInGbyOrSingleValueColNames))
for name, errExprLoc := range notInGbyOrSingleValueColNames {
    tblInfo := b.tblInfoFromCol(sel.From.TableRefs, name)
    if tblInfo == nil {
        continue
    }
    if _, ok := tblMap[tblInfo]; ok {
        continue
    }
    if checkColFuncDepend(p, name, tblInfo, gbyOrSingleValueColNames, whereDepends, joinDepends) {
        tblMap[tblInfo] = struct{}{}
        continue
    }
    switch errExprLoc.Loc {
    case ErrExprInSelect:
        if sel.GroupBy.Rollup {
            return plannererrors.ErrFieldInGroupingNotGroupBy.GenWithStackByArgs(strconv.Itoa(errExprLoc.Offset + 1))
        }
        return plannererrors.ErrFieldNotInGroupBy.GenWithStackByArgs(errExprLoc.Offset+1, errExprLoc.Loc, name.DBName.O+"."+name.TblName.O+"."+name.OrigColName.O)
    case ErrExprInOrderBy:
        return plannererrors.ErrFieldNotInGroupBy.GenWithStackByArgs(errExprLoc.Offset+1, errExprLoc.Loc, sel.OrderBy.Items[errExprLoc.Offset].Expr.Text())
    }
    return nil
}
return nil
```

## `checkOnlyFullGroupByWithOutGroupClause` 函数处理流程

`checkOnlyFullGroupByWithOutGroupClause` 函数用于检查不带 `GROUP BY` 子句但包含聚合函数的查询是否满足 `ONLY_FULL_GROUP_BY` 模式的要求（`tidb/pkg/planner/core/logical_plan_builder.go`）。

**步骤 1**：初始化列解析器 `colResolverForOnlyFullGroupBy`，用于收集查询中的非聚合列和聚合函数信息。

```go
resolver := colResolverForOnlyFullGroupBy{
    firstOrderByAggColIdx: -1,
}
```

**步骤 2**：遍历 `SELECT` 字段列表，使用列解析器识别和收集其中的非聚合列和聚合函数。

```go
resolver.curClause = fieldList
for idx, field := range sel.Fields.Fields {
    resolver.exprIdx = idx
    field.Accept(&resolver)
}
```

**步骤 3**：如果在 `SELECT` 列表中发现了非聚合列，函数会继续处理 `HAVING` 和 `ORDER BY` 子句，收集它们中的非聚合列和聚合函数。

```go
if len(resolver.nonAggCols) > 0 {
    if sel.Having != nil {
        sel.Having.Expr.Accept(&resolver)
    }
    if sel.OrderBy != nil {
        resolver.curClause = orderByClause
        for idx, byItem := range sel.OrderBy.Items {
            resolver.exprIdx = idx
            byItem.Expr.Accept(&resolver)
        }
    }
}
```

**步骤 4**：如果 `ORDER BY` 中存在聚合函数且 `SELECT` 列表中有非聚合列，则返回错误，例如：

```sql
`SELECT a FROM t WHERE a = 1 ORDER BY COUNT(b)`
```

```go
if resolver.firstOrderByAggColIdx != -1 && len(resolver.nonAggCols) > 0 {
    return plannererrors.ErrAggregateOrderNonAggQuery.GenWithStackByArgs(resolver.firstOrderByAggColIdx + 1)
}
```

**步骤 5**：如果查询不包含聚合函数或所有列都是聚合函数，则直接返回 `nil`，表示查询满足 `ONLY_FULL_GROUP_BY` 模式的要求。

```go
if !resolver.hasAggFuncOrAnyValue || len(resolver.nonAggCols) == 0 {
    return nil
}
```

**步骤 6**：从 `WHERE` 子句中提取单值列，这些列在查询上下文中被限制为单一值。

```go
singleValueColNames := make(map[*types.FieldName]struct{}, len(sel.Fields.Fields))
extractSingeValueColNamesFromWhere(p, sel.Where, singleValueColNames)
```

**步骤 7**：构建来自 `WHERE` 子句和 `JOIN` 条件的函数依赖关系。

```go
whereDepends, err := buildWhereFuncDepend(p, sel.Where)
if err != nil {
    return err
}

joinDepends, err := b.buildJoinFuncDepend(p, sel.From.TableRefs)
if err != nil {
    return err
}
```

**步骤 8**：遍历所有非聚合列，检查它们是否是单值列或是否通过函数依赖关系与单值列关联。如果所有非聚合列都通过了检查，函数返回 `nil`，表示查询满足 `ONLY_FULL_GROUP_BY` 模式的要求；否则返回相应的错误。

```go
tblMap := make(map[*model.TableInfo]struct{}, len(resolver.nonAggCols))
for i, colName := range resolver.nonAggCols {
    idx, err := expression.FindFieldName(p.OutputNames(), colName)
    if err != nil || idx < 0 {
        return plannererrors.ErrMixOfGroupFuncAndFields.GenWithStackByArgs(resolver.nonAggColIdxs[i]+1, colName.Name.O)
    }
    fieldName := p.OutputNames()[idx]
    if _, ok := singleValueColNames[fieldName]; ok {
        continue
    }
    tblInfo := b.tblInfoFromCol(sel.From.TableRefs, fieldName)
    if tblInfo == nil {
        continue
    }
    if _, ok := tblMap[tblInfo]; ok {
        continue
    }
    if checkColFuncDepend(p, fieldName, tblInfo, singleValueColNames, whereDepends, joinDepends) {
        tblMap[tblInfo] = struct{}{}
        continue
    }
    return plannererrors.ErrMixOfGroupFuncAndFields.GenWithStackByArgs(resolver.nonAggColIdxs[i]+1, colName.Name.O)
}
return nil
```

## `rewriteGbyExprs` 函数处理流程

`rewriteGbyExprs` 函数用于将 `GROUP BY` 子句中的表达式重写为优化器可以处理的表达式形式（`tidb/pkg/planner/core/logical_plan_builder.go`）。

**步骤 1**：初始化一个空的表达式切片，用于存储重写后的 `GROUP BY` 表达式，用于处理 `ROLLUP` 或聚合函数。

```go
exprs := make([]expression.Expression, 0, len(gby.Items))
```

**步骤 2**：遍历之前由 `resolveGbyExprs` 解析得到的表达式列表，对每个表达式调用 `b.rewrite` 函数进行重写，将 AST 表达式转换为表达式框架中的可执行表达式。

```go
for _, item := range items {
    expr, np, err := b.rewrite(ctx, item, p, nil, true)
    if err != nil {
        return nil, nil, false, err
    }

    exprs = append(exprs, expr)
    p = np
}
```

**步骤 3**：返回更新后的逻辑计划、重写后的表达式列表以及 `GROUP BY` 子句的 `Rollup` 标志。

```go
return p, exprs, gby.Rollup, nil
```
