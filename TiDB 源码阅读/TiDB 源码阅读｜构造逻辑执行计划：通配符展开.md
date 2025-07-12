# TiDB 源码阅读｜构造逻辑计划：通配符展开

在 SQL 查询中，我们经常会使用 `SELECT *` 这样的通配符来选择表中的所有列。TiDB 在构造逻辑计划过程中，需要将这些通配符展开为具体的列引用。本文将分析 TiDB 中处理通配符的相关代码逻辑。

## 处理 SELECT 语句中的通配符

在 `PlannerBuilder.buildSelect` 函数中（定义在 `logical_plan_builder.go` 文件中），先将 AST 节点中的原始字段列表保存到 `originalFields` 中，然后调用 `unfoldWildStar` 函数将 AST 节点中的字段列表中的通配符展开为具体的列，并赋值回 AST 节点中。

```go
originalFields := sel.Fields.Fields
sel.Fields.Fields, err = b.unfoldWildStar(p, sel.Fields.Fields)
if err != nil {
    return nil, err
}
```

## `PlanBuilder.unfoldWildStar` 处理流程

`PlanBuilder.unfoldWildStar` 函数定义在 `logical_plan_builder.go` 文件中，遍历 `SELECT` 语句中的每一个字段，以展开其中的通配符。

对于每个 `SELECT` 字段，执行逻辑如下：

**步骤 1**：如果当前字段不是通配符（`field.WildCard == nil`），直接添加到结果列表中并继续处理下一个字段。

```go
if field.WildCard == nil {
    resultList = append(resultList, field)
    continue
}
```

**步骤 2**：如果为不带限定表名的通配符，且不是第 1 个字段，则返回错误

```go
if field.WildCard.Table.L == "" && i > 0 {
    return nil, plannererrors.ErrInvalidWildCard
}
```

**步骤 3**：调用 `unfoldWildStar` 函数展开当前表的通配符

对于符合条件的通配符，使用通用展开函数 `unfoldWildStar` 处理，将通配符展开为具体的列。这里传入的是输出名称列表（`p.OutputNames()`）和输出模式的列（`p.Schema().Columns`），用于获取当前逻辑计划节点的所有可能列。【TODO：通配符中字段是何时添加到 OutputNames() 中的？】

```go
list := unfoldWildStar(field, p.OutputNames(), p.Schema().Columns)
```

**步骤 4**：处理包含 `JOIN` 操作且带限定表名的通配符

如果当前逻辑计划是 `JOIN` 操作（`isJoin` 为 `true`），并且包含完整模式（`join.FullSchema != nil`），且通配符指定了特定表名（`field.WildCard.Table.L != ""`），则使用 `JOIN` 操作的完整模式和名称来展开通配符。

```go
if isJoin && join.FullSchema != nil && field.WildCard.Table.L != "" {
    list = unfoldWildStar(field, join.FullNames, join.FullSchema.Columns)
}
```

**步骤 5**：如果展开后列表为空，则返回错误

```go
if len(list) == 0 {
    return nil, plannererrors.ErrBadTable.GenWithStackByArgs(field.WildCard.Table)
}
```

**步骤 6**：将展开后的列添加到 `SELECT` 列表中以替换通配符

```go
resultList = append(resultList, list...)
```

## 通用 `unfoldWildStar` 函数分析

`unfoldWildStar` 函数定义在 `logical_plan_builder.go` 文件中，构造通配符对应的 `SELECT` 字段列表。在函数中，遍历 `outputName` 中的每一个输出字段，校验数据库名和表名与通配符是否一致，如果一致或通配符中没有指定，则创建新的 `SelectField` 结构体，其中：

- `Expr` 字段是一个 `ColumnNameExpr`，包含了列的完全限定信息（包括数据库名、表名和列名）

```go
func unfoldWildStar(field *ast.SelectField, outputName types.NameSlice, column []*expression.Column) (resultList []*ast.SelectField) {
	dbName := field.WildCard.Schema
	tblName := field.WildCard.Table
	for i, name := range outputName {
		col := column[i]
		if col.IsHidden {
			continue
		}
		if (dbName.L == "" || dbName.L == name.DBName.L) &&
			(tblName.L == "" || tblName.L == name.TblName.L) &&
			col.ID != model.ExtraHandleID && col.ID != model.ExtraPhysTblID {
			colName := &ast.ColumnNameExpr{
				Name: &ast.ColumnName{
					Schema: name.DBName,
					Table:  name.TblName,
					Name:   name.ColName,
				}}
			colName.SetType(col.GetStaticType())
			field := &ast.SelectField{Expr: colName}
			field.SetText(nil, name.ColName.O)
			resultList = append(resultList, field)
		}
	}
	return resultList
}
```
