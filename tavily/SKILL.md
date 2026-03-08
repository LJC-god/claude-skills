---
name: tavily
description: 使用 Tavily API 进行网络搜索和内容提取，专为 LLM 优化。当用户需要搜索最新信息、查询新闻、提取网页内容、做实时研究、事实核查，或任何需要获取训练数据之外的最新网络内容时，都应使用此 skill。支持 search（网页搜索）、extract（网页内容提取）、research（深度研究报告）三种模式。需要环境变量 TAVILY_API_KEY。
---

# Tavily — LLM 专用搜索与提取 API

Tavily 是专为 LLM 优化的搜索引擎，提供实时、结构化的网络数据。

## 可用 Tools

| Tool | 功能 | 费用 | 适用场景 |
|------|------|------|----------|
| `search` | 网页搜索，返回结构化结果 | 1 credit（basic）/ 2 credits（advanced） | 实时信息、新闻、通用查询 |
| `extract` | 从指定 URL 提取完整内容 | 1 credit/URL | 读取特定网页全文 |
| `research` | 深度研究，生成完整报告 | 视模型而定 | 深度分析、综合报告 |

> API Key 通过环境变量 `TAVILY_API_KEY` 注入，Base URL：`https://api.tavily.com`

---

## 场景判断

- 用户问"最新…"、"今天…"、"现在…" → `search`（加 `time_range`）
- 用户问新闻/财经动态 → `search`，设 `topic: "news"` 或 `"finance"`
- 用户要读某个具体网页 → `extract`
- 用户要深度研究一个话题、生成报告 → `research`
- 一般信息查询 → `search`（`topic: "general"`，默认）

---

## Tool 1: search

**端点**：`POST https://api.tavily.com/search`

### 关键参数

| 参数 | 类型 | 默认 | 说明 |
|------|------|------|------|
| `query` | string | **必填** | 搜索关键词 |
| `search_depth` | `"basic"` \| `"advanced"` | `"basic"` | advanced 更深入但费 2 credits |
| `topic` | `"general"` \| `"news"` \| `"finance"` | `"general"` | 搜索类别 |
| `max_results` | int | 5 | 最多返回结果数（建议 5-10） |
| `include_answer` | bool \| `"basic"` \| `"advanced"` | false | 是否附带 LLM 生成的直接答案 |
| `include_raw_content` | bool \| `"markdown"` | false | 是否返回网页全文 |
| `time_range` | `"day"` \| `"week"` \| `"month"` \| `"year"` | null | 时间过滤 |
| `days` | int | 3 | 仅 `topic=news` 时有效，最近 N 天 |
| `include_domains` | string[] | [] | 只搜索这些域名 |
| `exclude_domains` | string[] | [] | 排除这些域名 |

### 请求示例

```json
POST https://api.tavily.com/search
Authorization: Bearer tvly-YOUR_API_KEY

{
  "query": "2025年AI大模型最新进展",
  "search_depth": "advanced",
  "topic": "news",
  "max_results": 5,
  "include_answer": "basic",
  "time_range": "week"
}
```

### 参数选择建议

- **一般查询**：`search_depth=basic`，省 credits
- **需要精确内容**：`search_depth=advanced` + `include_raw_content=true`
- **最新新闻**：`topic=news` + `days=1` 或 `time_range=day`
- **需要直接答案**：`include_answer="basic"`（快速）或 `"advanced"`（详细）

---

## Tool 2: extract

从指定 URL 提取完整网页内容。

**端点**：`POST https://api.tavily.com/extract`

```json
{ "urls": ["https://example.com/article1"], "include_images": false }
```

---

## Tool 3: research（异步，需轮询）

**创建任务**：`POST https://api.tavily.com/research`

```json
{ "query": "量子计算在密码学中的应用", "model": "pro", "citation_format": "numbered" }
```

| `model` | 说明 |
|---------|------|
| `"mini"` | 快速，表层分析 |
| `"pro"` | 全面深入（推荐） |
| `"auto"` | 自动选择（默认） |

**轮询结果**：`GET https://api.tavily.com/research/{request_id}`

`status` 字段：`"pending"` → `"running"` → `"completed"`

---

完整实现见 `references/implementation.md`。
