# mcp-jzl 实现参考

## 目录结构

```
mcp-jzl/
├── src/
│   ├── index.ts
│   ├── tools/
│   │   ├── get_article.ts
│   │   ├── get_article_list.ts
│   │   └── batch_get_articles.ts
│   └── utils/
│       ├── api.ts
│       ├── html_parser.ts
│       └── rate_limiter.ts
├── package.json
└── tsconfig.json
```

## src/utils/api.ts

```typescript
const JZL_BASE_URL = 'https://www.dajiala.com';

export async function callJzlApi(endpoint: string, params: Record<string, any>) {
  const response = await fetch(`${JZL_BASE_URL}${endpoint}`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ ...params, key: process.env.JZL_API_KEY }),
  });
  const result = await response.json();
  if (result.code !== 0) throw new Error(`极致了 API 错误 [${result.code}]: ${result.msg || '未知错误'}`);
  return result;
}

export const sleep = (ms: number) => new Promise(r => setTimeout(r, ms));
```

## src/tools/get_article.ts

```typescript
import { callJzlApi } from '../utils/api.js';
import { parse } from 'node-html-parser';

export async function getArticle({ url }: { url: string }) {
  const result = await callJzlApi('/fbmain/monitor/v3/article_html', { url, v: 2 });
  const d = result.data;
  const text = parse(d.html).text.replace(/\s+/g, ' ').trim();
  return { title: d.title, nickname: d.nickname, author: d.author, html: d.html, text,
    post_time: d.post_time_str, cover_url: d.cover_url, copyright: d.copyright, signature: d.signature, biz: d.biz };
}
```

## src/tools/get_article_list.ts

```typescript
import { callJzlApi, sleep } from '../utils/api.js';

export async function getArticleList(input: { url?: string; biz?: string; name?: string; page?: number; max_pages?: number }) {
  const { page = 1, max_pages = 1 } = input;
  const articles = [];
  let totalNum = 0, totalPage = 0, nickname = '', totalCost = 0;

  for (let p = page; p < page + max_pages; p++) {
    const result = await callJzlApi('/fbmain/monitor/v3/post_history', { url: input.url, biz: input.biz, name: input.name, page: p });
    if (p === page) { totalNum = result.total_num; totalPage = result.total_page; nickname = result.mp_nickname || ''; }
    totalCost += result.cost_money || 0;
    for (const item of result.data || []) {
      articles.push({ title: item.title, url: item.url, digest: item.digest, cover_url: item.cover_url,
        post_time: item.post_time_str, original: item.original, is_deleted: item.is_deleted === '1' });
    }
    if (p >= totalPage) break;
    await sleep(200);
  }
  return { nickname, total_num: totalNum, total_page: totalPage, articles, cost: totalCost };
}
```

## src/tools/batch_get_articles.ts

```typescript
import { getArticle } from './get_article.js';
import { getArticleList } from './get_article_list.js';
import { sleep } from '../utils/api.js';

export async function batchGetArticles(input: {
  urls?: string[]; source_url?: string; count?: number;
  output_format?: 'html' | 'text' | 'both'; only_original?: boolean; skip_deleted?: boolean;
}) {
  const { count = 20, output_format = 'both', only_original = false, skip_deleted = true } = input;
  let urls = input.urls || [];
  let nickname = '', listCost = 0;

  if (input.source_url && urls.length === 0) {
    const list = await getArticleList({ url: input.source_url, max_pages: Math.ceil(count / 5) });
    nickname = list.nickname; listCost = list.cost;
    let filtered = list.articles;
    if (only_original) filtered = filtered.filter(a => a.original === 1);
    if (skip_deleted) filtered = filtered.filter(a => !a.is_deleted);
    urls = filtered.slice(0, count).map(a => a.url);
  }

  const results = []; let successCount = 0, failedCount = 0;
  for (let i = 0; i < urls.length; i++) {
    try {
      const article = await getArticle({ url: urls[i] });
      if (!nickname) nickname = article.nickname;
      results.push({ title: article.title, url: urls[i],
        html: output_format !== 'text' ? article.html : undefined,
        text: output_format !== 'html' ? article.text : undefined,
        post_time: article.post_time, original: article.copyright, status: 'success' as const });
      successCount++;
    } catch (error: any) {
      results.push({ title: '', url: urls[i], status: 'failed' as const, error: error.message });
      failedCount++;
    }
    if (i < urls.length - 1) await sleep(200);
  }
  return { nickname, success_count: successCount, failed_count: failedCount,
    total_cost: listCost + successCount * 0.04, articles: results };
}
```

## 环境变量

```bash
JZL_API_KEY=你的极致了APIKey
```
