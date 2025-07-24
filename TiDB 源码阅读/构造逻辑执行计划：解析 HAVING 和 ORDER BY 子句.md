# TiDB 源码阅读｜构造逻辑执行计划：解析 HAVING 和 ORDER BY 子句

在 `Planbuilder.buildSelect` 函数（位于 `tidb/pkg/planner/core/logical_plan_builder.go`）中，必须在构建投影（Projection）之前调用 `resolveHavingAndOrderBy` 函数，解析 `HAVING` 和 `ORDER BY` 子句，以保证诸如 `select a+1 as b from t having sum(b) < 0` 中的 `sum(b)` 可以被替换为 `sum(a+1)`。

```go
havingMap, orderMap, err = b.resolveHavingAndOrderBy(ctx, sel, p)
if err != nil {
    return nil, err
}
```

## `resolveHavingAndOrderBy` 函数处理流程

`resolveHavingAndOrderBy` 函数定义在 `tidb/pkg/planner/core/logical_plan_builder.go` 中，用于解析 `SELECT` 语句中的 `HAVING` 和 `ORDER BY` 子句，提取聚合函数，并处理相关的辅助字段。函数签名如下：

```go
func (b *PlanBuilder) resolveHavingAndOrderBy(ctx context.Context, sel *ast.SelectStmt, p base.LogicalPlan) (havingAggMapper, _ map[*ast.AggregateFuncExpr]int, err error)
```

**步骤 1**：创建 `havingWindowAndOrderbyExprResolver` 解析器实例，包括当前逻辑计划、`SELECT` 字段列表、聚合映射等初始化其相关字段。

```go
extractor := &havingWindowAndOrderbyExprResolver{
    p:            p,
    selectFields: sel.Fields.Fields,
    aggMapper:    make(map[*ast.AggregateFuncExpr]int),
    colMapper:    b.colMapper,
    outerSchemas: b.outerSchemas,
    outerNames:   b.outerNames,
}
```

**步骤 2**：如果存在 `GROUP BY`，则将其分组项赋值给解析器。

```go
if sel.GroupBy != nil {
    extractor.gbyItems = sel.GroupBy.Items
}
```

**步骤 3**：解析 `HAVING` 子句中的表达式，并将更新后的表达式回写。

```go
if sel.Having != nil {
    extractor.curClause = havingClause
    n, ok := sel.Having.Expr.Accept(extractor)
    if !ok {
        return nil, nil, errors.Trace(extractor.err)
    }
    sel.Having.Expr = n.(ast.ExprNode)
}
```

**步骤 4**：将 `HAVING` 子句中提取到的聚合函数映射保存到 `havingAggMapper` 准备返回，并重置解析器中的聚合映射。

```go
havingAggMapper = extractor.aggMapper
extractor.aggMapper = make(map[*ast.AggregateFuncExpr]int)
```

**步骤 5**：解析 `ORDER BY` 子句中每个排序项的表达式，并将更新后的表达式回写。

- 如果存在 `ORDER BY` 子句，设置当前解析阶段为 `orderByClause`。
- 遍历 `ORDER BY` 的每个项，递归提取聚合函数。
- 跳过包含窗口函数的表达式。
- 解析后的表达式回写到 `item.Expr`。

```go
if sel.OrderBy != nil {
    extractor.curClause = orderByClause
    for _, item := range sel.OrderBy.Items {
        extractor.inExpr = false
        if ast.HasWindowFlag(item.Expr) {
            continue
        }
        n, ok := item.Expr.Accept(extractor)
        if !ok {
            return nil, nil, errors.Trace(extractor.err)
        }
        item.Expr = n.(ast.ExprNode)
    }
}
```

**步骤 6**：回写解析器中更新后的 `SELECT` 字段列表。

```go
sel.Fields.Fields = extractor.selectFields
```

**步骤 7**：从 `ORDER BY` 子句中的子查询项中获取相关的列（即使用了外层查询的列），并将这些列添加到外层查询的 `SELECT` 列表中，否则子查询将无法从外层模式中解析到该名称。

```go
if sel.OrderBy != nil {
    for _, byItem := range sel.OrderBy.Items {
        if _, ok := byItem.Expr.(*ast.SubqueryExpr); ok {
            // 关联聚合函数将在后续完全提取
            _, np, err := b.rewrite(ctx, byItem.Expr, p, nil, true)
            if err != nil {
                return nil, nil, errors.Trace(err)
            }
            correlatedCols := coreusage.ExtractCorrelatedCols4LogicalPlan(np)
            for _, cone := range correlatedCols {
                var colName *ast.ColumnName
                for idx, pone := range p.Schema().Columns {
                    if cone.UniqueID == pone.UniqueID {
                        pname := p.OutputNames()[idx]
                        colName = &ast.ColumnName{
                            Schema: pname.DBName,
                            Table:  pname.TblName,
                            Name:   pname.ColName,
                        }
                        break
                    }
                }
                if colName != nil {
                    columnNameExpr := &ast.ColumnNameExpr{Name: colName}
                    for _, field := range sel.Fields.Fields {
                        if c, ok := field.Expr.(*ast.ColumnNameExpr); ok && c.Name.Match(columnNameExpr.Name) && field.AsName.L == "" {
                            // 去重选择字段：如果已经存在一个，就不要追加
                            columnNameExpr = nil
                            break
                        }
                    }
                    if columnNameExpr != nil {
                        sel.Fields.Fields = append(sel.Fields.Fields, &ast.SelectField{
                            Auxiliary: true,
                            Expr:      columnNameExpr,
                        })
                    }
                }
            }
        }
    }
}
```

**步骤 8**：返回 `HAVING` 子句和 `ORDER BY` 子句中提取到的聚合函数映射。

```go
return havingAggMapper, extractor.aggMapper, nil
```
