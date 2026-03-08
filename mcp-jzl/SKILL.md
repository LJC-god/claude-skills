---
name: mcp-jzl
description: 使用极致了 API 提取微信公众号文章内容。当用户提到公众号文章链接、想获取公众号历史文章、分析某个公众号内容、批量提取文章、研究公众号写作风格，或任何涉及 mp.weixin.qq.com 链接的任务时，都应使用此 skill。支持单篇提取、获取文章列表、批量提取三种模式。
---

# MCP 极致了（mcp-jzl）

封装极致了 API，提供微信公众号文章提取能力。

## 可用 Tools

| Tool | 功能 | 费用 |
|------|------|------|
| `get_article` | 获取单篇文章完整内容（标题、正文、作者等） | ¥0.04/次 |
| `get_article_list` | 获取公众号历史文章列表 | ¥0.06-0.08/次 |
| `batch_get_articles` | 批量获取多篇文章内容 | ¥0.04×N + 列表费用 |

> API Key 通过环境变量 `JZL_API_KEY` 注入，调用前确认已配置。

---

## 使用流程

### 场景判断

- 用户提供单篇文章链接 → 用 `get_article`
- 用户想了解某公众号发了什么 → 用 `get_article_list`
- 用户要批量分析/提取内容 → 用 `batch_get_articles`
- 用户要分析写作风格 → 先 `get_article_list` 再 `batch_get_articles`（只取原创）

### 参数选择优先级

获取文章列表时，优先使用 **biz**（最便宜 ¥0.06），其次 **url**，最后 **name**（¥0.08）。

---

## Tool 详细说明

### get_article

```typescript
{ url: string }  // 微信公众号文章链接

// 返回
{ title, nickname, author, html, text, post_time, cover_url, copyright, signature, biz }
```

**API**：`POST https://www.dajiala.com/fbmain/monitor/v3/article_html`
```json
{ "url": "文章链接", "key": "JZL_API_KEY", "v": 2 }
```

---

### get_article_list

```typescript
// 输入（三选一）
{ url?, biz?, name?, page?: 1, max_pages?: 1 }

// 返回
{ nickname, total_num, total_page, articles: [{title, url, digest, cover_url, post_time, original, is_deleted}], cost }
```

**API**：`POST https://www.dajiala.com/fbmain/monitor/v3/post_history`
每页返回约 5 次发文，每次发文可包含 1-8 篇文章。

---

### batch_get_articles

```typescript
{
  urls?: string[],       // 方式一：直接提供 URL 列表
  source_url?: string,   // 方式二：从公众号自动获取
  count?: 20,
  output_format?: 'html' | 'text' | 'both',
  only_original?: false,
  skip_deleted?: true,
}
```

**QPS 限制**：5次/秒，每次调用后等待 200ms。

---

## 错误处理

| code | 含义 | 处理 |
|------|------|------|
| 0 | 成功 | — |
| -1 | QPS 超限 | 等待 5 秒重试 |
| 101 | 文章已删除/违规 | 跳过 |
| 105 | 找不到公众号 | 改用 biz 或 url |
| 110 | 翻页无文章 | 停止翻页 |
| 20001 | 余额不足 | 提示充值 |

---

## 成本提示

| 场景 | 预估成本 |
|------|----------|
| 单篇分析 | ¥0.04 |
| 获取列表（1页） | ¥0.06 |
| 批量提取 10 篇 | ≈¥0.46 |
| 批量提取 30 篇 | ≈¥1.32 |
| 批量提取 100 篇 | ≈¥4.24 |

完整 TypeScript 实现见 `references/implementation.md`。
