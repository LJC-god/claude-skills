# Tavily 实现参考

## src/tools/search.ts

```typescript
export async function search(input: {
  query: string; search_depth?: 'basic' | 'advanced';
  topic?: 'general' | 'news' | 'finance'; max_results?: number;
  include_answer?: boolean | 'basic' | 'advanced';
  include_raw_content?: boolean | 'markdown';
  time_range?: 'day' | 'week' | 'month' | 'year';
  days?: number; include_domains?: string[]; exclude_domains?: string[];
}) {
  const apiKey = process.env.TAVILY_API_KEY;
  if (!apiKey) throw new Error('TAVILY_API_KEY 未设置');
  const response = await fetch('https://api.tavily.com/search', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json', 'Authorization': `Bearer ${apiKey}` },
    body: JSON.stringify(input),
  });
  if (!response.ok) throw new Error(`Tavily search 失败 [${response.status}]`);
  return await response.json();
}
```

## src/tools/extract.ts

```typescript
export async function extract(input: { urls: string[]; include_images?: boolean }) {
  const apiKey = process.env.TAVILY_API_KEY;
  const response = await fetch('https://api.tavily.com/extract', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json', 'Authorization': `Bearer ${apiKey}` },
    body: JSON.stringify(input),
  });
  if (!response.ok) throw new Error(`Tavily extract 失败 [${response.status}]`);
  return await response.json();
}
```

## src/tools/research.ts

```typescript
export async function createResearch(input: {
  query: string; model?: 'mini' | 'pro' | 'auto';
  citation_format?: 'numbered' | 'mla' | 'apa' | 'chicago';
}) {
  const apiKey = process.env.TAVILY_API_KEY;
  const response = await fetch('https://api.tavily.com/research', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json', 'Authorization': `Bearer ${apiKey}` },
    body: JSON.stringify(input),
  });
  if (!response.ok) throw new Error(`创建研究任务失败 [${response.status}]`);
  return await response.json();
}

export async function waitForResearch(requestId: string, timeoutMs = 120000) {
  const apiKey = process.env.TAVILY_API_KEY;
  const start = Date.now();
  while (Date.now() - start < timeoutMs) {
    const res = await fetch(`https://api.tavily.com/research/${requestId}`, {
      headers: { 'Authorization': `Bearer ${apiKey}` },
    });
    const result = await res.json();
    if (result.status === 'completed') return result;
    if (result.status === 'failed') throw new Error(`研究任务失败: ${result.error}`);
    await new Promise(r => setTimeout(r, 3000));
  }
  throw new Error('研究任务超时');
}
```

## 环境变量

```bash
TAVILY_API_KEY=tvly-你的APIKey
```

## 费用参考

| 操作 | Credits |
|------|---------|
| search basic | 1/次 |
| search advanced | 2/次 |
| extract | 1/URL |
| research mini | ~10 |
| research pro | ~50 |

免费额度：每月 1,000 credits
