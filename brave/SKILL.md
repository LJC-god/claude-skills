---
name: brave-search
description: 使用 Brave Search API 进行网络搜索，隐私优先、独立索引，专为 LLM 和 AI 应用优化。当用户需要搜索最新信息、查询新闻、搜索图片/视频/本地地点，或任何需要实时网络数据的任务时，都应使用此 skill。支持 web（通用搜索）、news（新闻搜索）、images（图片搜索）、videos（视频搜索）四种模式，以及 AI 摘要生成。需要环境变量 BRAVE_API_KEY。
---

# Brave Search API

Brave Search 拥有独立的 300 亿页网络索引，隐私优先，不追踪用户查询，专为 AI/LLM 应用设计。

## 认证

```
Base URL: https://api.search.brave.com/res/v1
Header: X-Subscription-Token: <BRAVE_API_KEY>
        Accept: application/json
```

---

## 可用 Endpoints

| Endpoint | 功能 | 适用场景 |
|----------|------|----------|
| `GET /web/search` | 通用网页搜索 | 通用查询、技术问题、最新信息 |
| `GET /news/search` | 专项新闻搜索 | 时事、新闻、舆情监控 |
| `GET /images/search` | 图片搜索 | 查找图片资源 |
| `GET /videos/search` | 视频搜索 | 查找视频内容 |
| `GET /summarizer/search` | AI 摘要（两步流程） | 需要直接答案时 |

---

## 场景判断

- 用户问"最新消息"、"今天发生了什么" → `/news/search` + `freshness=pd`
- 用户要找图片 → `/images/search`
- 用户要找视频/教程 → `/videos/search`
- 用户要一个直接答案 → `/web/search` + `summary=1`，再调 `/summarizer/search`
- 其他通用查询 → `/web/search`

---

## web_search 关键参数

| 参数 | 默认 | 说明 |
|------|------|------|
| `q` | **必填** | 搜索词，支持 `site:`, `"精确短语"`, `filetype:pdf` 等算符 |
| `count` | 20 | 返回数量（最多 20） |
| `offset` | 0 | 翻页偏移（最大 9） |
| `freshness` | — | `pd`=今天, `pw`=本周, `pm`=本月, `py`=今年, 或 `YYYY-MM-DDtoYYYY-MM-DD` |
| `country` | — | 两字母国家码，如 `US`, `CN` |
| `search_lang` | — | 内容语言，如 `en`, `zh` |
| `result_filter` | — | 逗号分隔：`web,news,videos,images` |
| `extra_snippets` | false | 每条结果附加最多 5 个额外摘要（需 AI/Data 套餐） |
| `summary` | false | 请求 AI 摘要 key |
| `safesearch` | `moderate` | `off` / `moderate` / `strict` |
| `goggles_id` | — | Goggle URL，自定义结果重排序 |

---

## AI 摘要两步流程

### Step 1 — 触发

```bash
GET /web/search?q=量子计算原理&summary=1
```

响应包含 `summarizer.key`（不透明字符串，原样传递）。

### Step 2 — 获取摘要

```bash
GET /summarizer/search?key=<KEY>&entity_info=1&inline_references=true
```

---

## 搜索算符（写入 q 参数）

| 算符 | 示例 | 效果 |
|------|------|------|
| `"短语"` | `"climate change"` | 精确短语 |
| `-词` | `tech -crypto` | 排除词 |
| `site:` | `site:github.com` | 限定域名 |
| `filetype:` | `filetype:pdf` | 限定格式 |

---

完整实现见 `references/implementation.md`。
