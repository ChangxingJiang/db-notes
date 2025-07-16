# TiDB 源码阅读｜构造逻辑执行计划：输出列命名

当执行 `SELECT` 语句时，每个输出列都需要一个明确的名称，这些名称可能来自原始列名、用户指定的别名，或者由系统根据表达式自动生成。TiDB 输出列命名的逻辑位于 `PlanBuilder.addAliasName` 函数中，定义在 `pkg/planner/core/logical_plan_builder.go` 文件中。

`PlanBuilder.addAliasName` 函数的签名如下：

```go
func (b *PlanBuilder) addAliasName(ctx context.Context, selectStmt *ast.SelectStmt, p base.LogicalPlan) (resultList []*ast.SelectField, err error)
```

该函数负责处理 `SELECT` 字段的别名映射，将 `SelectField` 字段中的 `asName` 属性修改为映射后的别名，返回新的 `SELECT` 字段（`SelectField`）的列表。别名映射规则如下：

1. 对于有别名 `SELECT` 字段（例如 `SELECT column_name AS alias_name`），此时输出列名为别名 `alias_name`，即不需要修改 `AsName` 属性
2. 对于没有指定别名的列引用表达式的 `SELECT` 字段（例如 `SELECT column_name`），此时输出列名为原始列名 `column_name`
3. 对于没有指定别名的非列引用表达式的 `SELECT` 字段（例如 `SELECT name_const('col', 100)`），此时输出列名为 "Name_exp_{列名}" 前缀或 "Name_exp_{数字}_{列名}"（存在重复），其中 `{列名}` 部分调用 `buildProjectionField` 函数生成。【TODO：`buildProjectionField` 函数】

`PlanBuilder.addAliasName` 函数的处理流程如下：

**步骤 1**：初始化输出名称列表

遍历 `SELECT` 语句中的所有字段，构造每个字段的输出名称的列表 `projOutNames`。名称构造规则如下：

- 对于列引用表达式（例如 `SELECT column_name`），如果指定了别名则使用别名，否则直接使用列名
- 对于其他表达式（例如 `SELECT name_const('col', 100)`），则调用 `buildProjectionField` 函数生成名称。

在生成的 `FieldName` 结构体中，包含 5 个属性：

- `TblName`：字段所属的表名
- `OrigTblName`：字段的原始表名
- `ColName`：最终用于输出的列名
- `OrigColName`：原始的列名
- `DBName`：字段所属的数据库名

```go
selectFields := selectStmt.Fields.Fields
projOutNames := make([]*types.FieldName, 0, len(selectFields))
for _, field := range selectFields {
	colNameField, isColumnNameExpr := field.Expr.(*ast.ColumnNameExpr)
	if isColumnNameExpr {
		colName := colNameField.Name.Name
		if field.AsName.L != "" {
			colName = field.AsName
		}
		projOutNames = append(projOutNames, &types.FieldName{
			TblName:     colNameField.Name.Table,
			OrigTblName: colNameField.Name.Table,
			ColName:     colName,
			OrigColName: colNameField.Name.Name,
			DBName:      colNameField.Name.Schema,
		})
	} else {
		// create view v as select name_const('col', 100);
		// The column in v should be 'col', so we call `buildProjectionField` to handle this.
		_, name, err := b.buildProjectionField(ctx, p, field, nil)
		if err != nil {
			return nil, err
		}
		projOutNames = append(projOutNames, name)
	}
}
```

**步骤 2**：重命名同名的匿名字段

匿名字段指的是在 SELECT 语句中没有显式指定别名（`newField.AsName.L == ""`），且表达式本身不是简单的列名（不是 `ast.ColumnNameExpr` 类型），而是诸如如常量、函数调用、表达式等的字段，例如 `1`、`a + b`、`UPPER(name)` 等就都是匿名字段。

遍历所有 `SELECT` 字段，标记哪些字段是匿名字段，并将非匿名字段的名称添加到去重映射中；然后遍历所有匿名字段，根据是否已存在同名字段，为其添加 "Name_exp_" 前缀或 "Name_exp_{数字}_" 前缀。

```go
// dedupMap is used for renaming a duplicated anonymous column
dedupMap := make(map[string]int)
anonymousFields := make([]bool, len(selectFields))

for i, field := range selectFields {
	newField := *field
	if newField.AsName.L == "" {
		newField.AsName = projOutNames[i].ColName
	}

	if _, ok := field.Expr.(*ast.ColumnNameExpr); !ok && field.AsName.L == "" {
		anonymousFields[i] = true
	} else {
		anonymousFields[i] = false
		// dedupMap should be inited with all non-anonymous fields before renaming other duplicated anonymous fields
		dedupMap[newField.AsName.L] = 0
	}

	resultList = append(resultList, &newField)
}

// We should rename duplicated anonymous fields in the first SelectStmt of CreateViewStmt
// See: https://github.com/pingcap/tidb/issues/29326
if selectStmt.AsViewSchema {
	for i, field := range resultList {
		if !anonymousFields[i] {
			continue
		}

		oldName := field.AsName
		if dup, ok := dedupMap[field.AsName.L]; ok {
			if dup == 0 {
				field.AsName = ast.NewCIStr(fmt.Sprintf("Name_exp_%s", field.AsName.O))
			} else {
				field.AsName = ast.NewCIStr(fmt.Sprintf("Name_exp_%d_%s", dup, field.AsName.O))
			}
			dedupMap[oldName.L] = dup + 1
		} else {
			dedupMap[oldName.L] = 0
		}
	}
}

return resultList, nil
```
