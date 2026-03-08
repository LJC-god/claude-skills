# Notion 常见用例参考

## 用例 1：读取并总结页面

```
notion-fetch(id: "https://notion.so/workspace/Page-abc123")
→ 直接从返回的 Markdown 内容中总结
```

## 用例 2：数据库新建任务

```
1. notion-fetch(id: "数据库URL")
   → 获取 schema 和 data_source_id

2. notion-create-pages(
     parent: { type: "data_source_id", data_source_id: "xxx" },
     pages: [{
       properties: {
         "Task Name": "实现登录功能",
         "Status": "To Do",
         "date:Due Date:start": "2025-04-01",
         "date:Due Date:is_datetime": 0
       }
     }]
   )
```

## 用例 3：更新任务状态

```
1. notion-search(query: "任务名称", query_type: "internal")
2. notion-update-page(command: "update_properties", page_id: "xxx",
     properties: { "Status": "Done", "Done": "__YES__" })
```

## 用例 4：搜索并整理页面

```
1. notion-search(query: "Q1 报告", query_type: "internal",
     filters: { created_date_range: { start_date: "2025-01-01" } })
2. notion-move-pages(
     page_or_database_ids: ["id1", "id2"],
     new_parent: { type: "page_id", page_id: "目标页面ID" })
```

## 用例 5：创建新数据库

```
notion-create-database(
  parent: { type: "page_id", page_id: "父页面ID" },
  title: "产品需求库",
  schema: "CREATE TABLE (
    \"Name\" TITLE,
    \"Status\" SELECT('规划':gray,'开发中':blue,'已完成':green),
    \"负责人\" PEOPLE,
    \"截止日期\" DATE,
    \"ID\" UNIQUE_ID PREFIX 'PRD'
  )"
)
```

## 用例 6：给数据库添加字段

```
1. notion-fetch(id: "数据库URL")  → 获取 collection://abc 中的 ID
2. notion-update-data-source(data_source_id: "abc",
     statements: "ADD COLUMN \"Reviewer\" PEOPLE; ADD COLUMN \"Review Date\" DATE")
```

## 用例 7：添加评论

```
1. notion-fetch(id: "页面ID", include_discussions: true)
2. notion-create-comment(page_id: "xxx",
     selection_with_ellipsis: "这里是要...评论的内容",
     rich_text: [{ text: { content: "需要确认一下" } }])
```

## 属性类型速查

| 类型 | 示例值 |
|------|--------|
| TITLE / RICH_TEXT | `"文本内容"` |
| SELECT / STATUS | `"选项名"` |
| MULTI_SELECT | `"选项1,选项2"` |
| NUMBER | `42`（JS 数字） |
| CHECKBOX | `"__YES__"` 或 `"__NO__"` |
| DATE | `"date:Due:start": "2025-01-01"`, `"date:Due:is_datetime": 0` |
| PEOPLE | `"user-uuid"` |
| URL（特殊命名） | key 用 `"userDefined:URL"` |
