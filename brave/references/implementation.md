# Brave Search 实现参考

## 通用请求封装

```typescript
const BASE = 'https://api.search.brave.com/res/v1';

function headers() {
  const key = process.env.BRAVE_API_KEY;
  if (!key) throw new Error('BRAVE_API_KEY 未设置');
  return { 'Accept': 'application/json', 'X-Subscription-Token': key };
}

async function braveGet(path: string, params: Record<string, any>) {
  const p = new URLSearchParams();
  Object.entries(params).forEach(([k, v]) => { if (v !== undefined) p.set(k, String(v)); });
  const res = await fetch(`${BASE}${path}?${p}`, { headers: headers() });
  if (!res.ok) throw new Error(`Brave API 失败 [${res.status}]: ${path}`);
  return await res.json();
}
```

## Web Search

```typescript
export const webSearch = (q: string, opts?: {
  count?: number; offset?: number; freshness?: string;
  country?: string; search_lang?: string; result_filter?: string;
  extra_snippets?: boolean; summary?: boolean;
  safesearch?: 'off'|'moderate'|'strict'; goggles_id?: string;
}) => braveGet('/web/search', { q, ...opts });
```

## News Search

```typescript
export const newsSearch = (q: string, opts?: {
  count?: number; freshness?: string; country?: string;
  search_lang?: string; safesearch?: string;
}) => braveGet('/news/search', { q, ...opts });
```

## Images / Videos

```typescript
export const imagesSearch = (q: string, count = 20) => braveGet('/images/search', { q, count });
export const videosSearch = (q: string, opts?: { count?: number; freshness?: string }) =>
  braveGet('/videos/search', { q, ...opts });
```

## AI Summarizer

```typescript
export async function getSummary(key: string, opts?: { entity_info?: boolean; inline_references?: boolean }) {
  const p = new URLSearchParams({ key });
  if (opts?.entity_info) p.set('entity_info', '1');
  if (opts?.inline_references) p.set('inline_references', 'true');
  const res = await fetch(`${BASE}/summarizer/search?${p}`, { headers: headers() });
  if (!res.ok) throw new Error(`Brave summarizer 失败 [${res.status}]`);
  return await res.json();
}
```

## freshness 参数速查

| 值 | 含义 |
|----|------|
| `pd` | 过去 24 小时 |
| `pw` | 过去一周 |
| `pm` | 过去一个月 |
| `py` | 过去一年 |
| `2025-01-01to2025-03-01` | 自定义日期范围 |

## 环境变量

```bash
BRAVE_API_KEY=你的BraveAPIKey
```
