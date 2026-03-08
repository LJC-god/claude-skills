# Reddit API 实现参考

## OAuth 认证流程（script app，只读）

```typescript
const REDDIT_BASE = 'https://oauth.reddit.com';

async function getAccessToken(): Promise<string> {
  const clientId = process.env.REDDIT_CLIENT_ID;
  const clientSecret = process.env.REDDIT_CLIENT_SECRET;
  const userAgent = process.env.REDDIT_USER_AGENT || 'myapp/1.0';

  if (!clientId || !clientSecret) {
    throw new Error('REDDIT_CLIENT_ID 和 REDDIT_CLIENT_SECRET 环境变量未设置');
  }

  const credentials = Buffer.from(`${clientId}:${clientSecret}`).toString('base64');
  const res = await fetch('https://www.reddit.com/api/v1/access_token', {
    method: 'POST',
    headers: {
      'Authorization': `Basic ${credentials}`,
      'User-Agent': userAgent,
      'Content-Type': 'application/x-www-form-urlencoded',
    },
    body: 'grant_type=client_credentials',
  });

  const data = await res.json();
  return data.access_token; // 有效期 3600 秒
}

// 通用请求封装
async function redditGet(path: string, params: Record<string, any> = {}) {
  const token = await getAccessToken();
  const p = new URLSearchParams(params);
  const res = await fetch(`${REDDIT_BASE}${path}?${p}`, {
    headers: {
      'Authorization': `Bearer ${token}`,
      'User-Agent': process.env.REDDIT_USER_AGENT || 'myapp/1.0',
    },
  });
  if (!res.ok) throw new Error(`Reddit API 失败 [${res.status}]`);
  return res.json();
}
```

## 无需认证的公开访问

```typescript
// 直接在 URL 后加 .json，适合快速原型
async function redditPublic(url: string) {
  const apiUrl = url.endsWith('.json') ? url : `${url}.json`;
  const res = await fetch(apiUrl, {
    headers: { 'User-Agent': 'myapp/1.0' }
  });
  return res.json();
}

// 示例
const posts = await redditPublic('https://www.reddit.com/r/programming/hot');
```

## 搜索帖子

```typescript
export async function searchPosts(query: string, opts?: {
  subreddit?: string;
  sort?: 'relevance' | 'hot' | 'top' | 'new' | 'comments';
  time?: 'hour' | 'day' | 'week' | 'month' | 'year' | 'all';
  limit?: number;
  after?: string; // 翻页游标
}) {
  const path = opts?.subreddit ? `/r/${opts.subreddit}/search` : '/search';
  return redditGet(path, {
    q: query,
    restrict_sr: opts?.subreddit ? 'true' : 'false',
    sort: opts?.sort || 'relevance',
    t: opts?.time || 'all',
    limit: opts?.limit || 25,
    after: opts?.after,
  });
}
```

## 浏览 Subreddit

```typescript
export async function getSubredditPosts(
  subreddit: string,
  sort: 'hot' | 'new' | 'top' | 'rising' = 'hot',
  opts?: { limit?: number; t?: string }
) {
  return redditGet(`/r/${subreddit}/${sort}`, {
    limit: opts?.limit || 25,
    t: opts?.t || 'week', // 仅 top 有效
  });
}
```

## 获取帖子详情和评论

```typescript
export async function getPostComments(
  subreddit: string,
  postId: string,
  opts?: { sort?: 'top' | 'best' | 'new' | 'old'; limit?: number; depth?: number }
) {
  // 返回 [postData, commentsData]
  return redditGet(`/r/${subreddit}/comments/${postId}`, {
    sort: opts?.sort || 'top',
    limit: opts?.limit || 100,
    depth: opts?.depth || 3,
  });
}

// 从 URL 提取 subreddit 和 postId
export function parseRedditUrl(url: string) {
  const match = url.match(/reddit\.com\/r\/([^/]+)\/comments\/([^/]+)/);
  return match ? { subreddit: match[1], postId: match[2] } : null;
}
```

## Subreddit 信息

```typescript
export async function getSubredditInfo(subreddit: string) {
  return redditGet(`/r/${subreddit}/about`);
  // 返回：subscribers, active_user_count, description, created_utc, over18
}
```

## 用户信息

```typescript
export async function getUserInfo(username: string) {
  return redditGet(`/user/${username}/about`);
}

export async function getUserPosts(username: string, opts?: {
  sort?: 'top' | 'new' | 'hot'; t?: string; limit?: number;
}) {
  return redditGet(`/user/${username}/submitted`, {
    sort: opts?.sort || 'top',
    t: opts?.t || 'all',
    limit: opts?.limit || 25,
  });
}
```

## 响应数据解析

```typescript
// 解析帖子列表
function parsePosts(data: any) {
  return data.data.children.map((c: any) => ({
    id: c.data.id,
    title: c.data.title,
    author: c.data.author,
    subreddit: c.data.subreddit,
    score: c.data.score,
    upvoteRatio: c.data.upvote_ratio,
    numComments: c.data.num_comments,
    url: c.data.url,
    selftext: c.data.selftext,
    createdAt: new Date(c.data.created_utc * 1000).toISOString(),
    permalink: `https://www.reddit.com${c.data.permalink}`,
    isVideo: c.data.is_video,
    isSelf: c.data.is_self, // true=文字帖，false=链接帖
  }));
}

// 翻页
const nextPage = data.data.after; // 传给下次请求的 after 参数
```

## MCP Server 工具列表

```typescript
tools: [
  { name: 'reddit_search', description: '全站或指定 subreddit 搜索帖子' },
  { name: 'reddit_subreddit_posts', description: '浏览 subreddit 热门/最新帖子' },
  { name: 'reddit_post_comments', description: '获取帖子详情和评论' },
  { name: 'reddit_subreddit_info', description: '获取 subreddit 统计和描述' },
  { name: 'reddit_user_info', description: '获取用户信息和帖子历史' },
  { name: 'reddit_trending', description: '获取当日趋势 subreddits' },
]
```

## 环境变量

```bash
REDDIT_CLIENT_ID=你的client_id        # 必须
REDDIT_CLIENT_SECRET=你的client_secret  # 必须
REDDIT_USER_AGENT=myapp/1.0 (by /u/username)  # 必须，Reddit 要求标识应用
REDDIT_USERNAME=可选                    # 需要写操作时填写
REDDIT_PASSWORD=可选                    # 需要写操作时填写
```

## 获取 API 凭据步骤

1. 登录 Reddit → 访问 https://www.reddit.com/prefs/apps
2. 点击 "create another app"
3. 类型选 **script**
4. redirect_uri 填 `http://localhost:8080`
5. 创建后获得 `client_id`（应用名下方的短字符串）和 `client_secret`

## 速率限制

- OAuth 认证后：60 req/min
- 未认证：约 10 req/min
- 响应 Header：`X-Ratelimit-Remaining` / `X-Ratelimit-Reset`
