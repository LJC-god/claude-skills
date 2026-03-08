---
name: notion
description: 使用已连接的 Notion MCP 操作 Notion 工作区。当用户想要搜索、读取、创建、编辑、整理 Notion 页面或数据库，查看评论，管理任务，或任何涉及 Notion 内容的任务时，都应使用此 skill。支持页面/数据库的完整 CRUD、语义搜索、评论、团队管理。需要 Notion MCP 已连接（已在 Claude.ai 设置中启用）。
---

# Notion Skill

通过 Notion MCP 操作工作区中的页面、数据库、用户、团队和评论。

## 可用 Tools 总览

| Tool | 功能 |
|------|------|
| `notion-search` | 语义搜索工作区页面/数据库，或查找用户 |
| `notion-fetch` | 读取页面/数据库完整内容及 schema |
| `notion-create-pages` | 创建新页面（支持数据库条目、子页面、独立页面） |
| `notion-update-page` | 更新页面属性或内容（搜索替换/全量替换） |
| `notion-create-database` | 创建新数据库（用 SQL DDL 定义 schema） |
| `notion-update-data-source` | 修改数据库 schema（增删改列） |
| `notion-move-pages` | 移动页面/数据库到新的父级 |
| `notion-duplicate-page` | 复制页面（异步） |
| `notion-create-comment` | 在页面或特定内容处添加评论 |
| `notion-get-comments` | 获取页面讨论和评论 |
| `notion-get-users` | 列出/查找工作区成员 |
| `notion-get-teams` | 列出/查找团队（Teamspace） |

---

## 常见场景与推荐流程

### 读取页面内容
```
notion-fetch(id: "页面URL或ID")
```

### 搜索内容
```
notion-search(query: "关键词", query_type: "internal")
```

### 在数据库中新建条目
1. `notion-fetch(id: "数据库URL")` → 获取 schema 和 data_source_id
2. `notion-create-pages(parent: {data_source_id: "..."}, pages: [{properties: {...}}])`

### 修改页面内容（局部）
1. `notion-fetch(id: "页面ID")` → 获取当前内容
2. `notion-update-page(command: "update_content", content_updates: [{old_str: "...", new_str: "..."}])`

---

## Tool 详细说明

### notion-search
```
query: string          # 搜索词（必填）
query_type: "internal" | "user"
page_url?: string      # 限定在某页面内搜索
data_source_url?: string  # 限定在某数据库内（collection://... URL）
teamspace_id?: string
filters?: { created_date_range, created_by_user_ids }
```

### notion-fetch
```
id: string   # 页面URL、数据库URL、UUID 或 collection://... URL
include_discussions?: boolean
```
> **重要**：创建/更新数据库条目或修改页面内容前，**必须先 fetch**。

### notion-create-pages
```
parent?: { type: "page_id"|"data_source_id"|"database_id", ... }
pages: [{ properties: {...}, content?: string, template_id?: string }]
```

**属性格式特殊规则**：
- 日期：`"date:Due Date:start": "2024-12-25"`, `"date:Due Date:is_datetime": 0`
- 数字：JavaScript 数字（非字符串）
- 复选框：`"__YES__"` 或 `"__NO__"`
- 属性名为 `id`/`url`：加前缀 `"userDefined:URL"`

### notion-update-page
```
command: "update_properties" | "update_content" | "replace_content" | "apply_template" | "update_verification"
page_id: string
```
- **update_content**（推荐）：`content_updates: [{old_str, new_str}]` 精确局部修改
- **replace_content**：替换整页，会删除子页面，需用户确认
- **update_properties**：`null` 可清空属性

### notion-create-database（SQL DDL）
```sql
CREATE TABLE (
  "Name" TITLE,
  "Status" SELECT('To Do':red, 'In Progress':yellow, 'Done':green),
  "Tags" MULTI_SELECT('eng':blue, 'design':pink),
  "Due Date" DATE,
  "Priority" NUMBER,
  "Done" CHECKBOX,
  "Owner" PEOPLE,
  "ID" UNIQUE_ID PREFIX 'TASK'
)
```

### notion-update-data-source
```sql
ADD COLUMN "Priority" SELECT('High':red, 'Low':green);
DROP COLUMN "Old Field";
RENAME COLUMN "Name" TO "Task Name"
```
先 fetch 数据库，从 `<data-source url="collection://ID">` 获取 data_source_id。

### notion-create-comment
```
page_id: string
rich_text: [{ text: { content: "评论内容" } }]
selection_with_ellipsis?: string  # 开头~10字符 + ... + 结尾~10字符
discussion_id?: string            # 回复已有讨论
```

---

## 注意事项

- **创建数据库条目前必须 fetch 数据库**：获取准确属性名和 data_source_id
- **修改内容前必须 fetch 页面**：获取精确文本用于 update_content
- **replace_content 会删除子页面**：操作前需用户确认
- **多数据源数据库**：必须用 `data_source_id`，不能用 `database_id`
- **duplicate-page 是异步的**：内容稍后填充

---

完整用例见 `references/examples.md`。
