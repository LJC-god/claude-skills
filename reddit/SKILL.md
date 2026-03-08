---
name: reddit
description: 使用 Reddit API（PRAW）浏览、搜索和分析 Reddit 内容。当用户想搜索 Reddit 帖子、浏览某个 subreddit、获取帖子评论、分析社区讨论、做舆情研究、品牌监控，或任何涉及 Reddit 内容的任务时，都应使用此 skill。支持搜索、热门帖子、评论、用户信息、subreddit 统计等。需要环境变量 REDDIT_CLIENT_ID 和 REDDIT_CLIENT_SECRET。
---

# Reddit Skill

通过 Reddit OAuth API 浏览、搜索和分析 Reddit 内容。

## 认证配置

```bash
# Reddit OAuth（只读，无需用户名密码）
REDDIT_CLIENT_ID=你的client_id
REDDIT_CLIENT_SECRET=你的client_secret
REDDIT_USER_AGENT=myapp/1.0 (by /u/yourusername)

# 可选（用于发帖/评论等写操作）
REDDIT_USERNAME=你的用户名
REDDIT_PASSWORD=你的密码
```

> 在 [Reddit App Preferences](https://www.reddit.com/prefs/apps) 创建应用获取凭据。
> 类型选 **script**，redirect_uri 填 `http://localhost:8080`。

### OAuth Token 获取（只读）

```bash
curl -X POST https://www.reddit.com/api/v1/access_token \
  -u "CLIENT_ID:CLIENT_SECRET" \
  -d "grant_type=client_credentials" \
  -H "User-Agent: myapp/1.0"
# 返回 access_token，有效期 1 小时
```

所有 API 请求加 Header：
```
Authorization: Bearer ACCESS_TOKEN
User-Agent: myapp/1.0
```

---

## 可用功能

| 功能 | Endpoint | 说明 |
|------|----------|------|
| 搜索帖子 | `GET /search` | 全站或指定 subreddit 搜索 |
| 热门帖子 | `GET /r/{sub}/{sort}` | hot/new/top/rising |
| 帖子详情+评论 | `GET /r/{sub}/comments/{id}` | 帖子内容和评论树 |
| Subreddit 信息 | `GET /r/{sub}/about` | 描述、订阅数等 |
| 用户帖子/评论 | `GET /user/{name}/submitted` | 用户历史 |
| 趋势 subreddits | `GET /api/trending_subreddits` | 当日趋势 |

**API Base URL**：`https://oauth.reddit.com`（OAuth 认证后）或 `https://www.reddit.com`（只读公开内容加 `.json`）

---

## 场景判断

- 用户要搜索某话题 → `search`（可限定 subreddit）
- 用户提供 Reddit 链接 → 直接 fetch 链接 + `.json`
- 用户要看某 subreddit 热门 → `GET /r/{sub}/hot`
- 用户要做品牌监控 → `search` + `time_filter=week` + 多关键词
- 用户要分析某社区 → `about` + `top` 帖子
- 用户要看某帖子评论 → `GET /r/{sub}/comments/{id}`

---

## API 详细说明

### 1. 搜索帖子

```bash
GET https://oauth.reddit.com/search
  ?q=搜索词
  &restrict_sr=false        # true=限定在指定 subreddit
  &sr_detail=false
  &sort=relevance           # relevance | hot | top | new | comments
  &t=all                    # hour | day | week | month | year | all
  &limit=25                 # 最多 100
  &after=t3_postid          # 翻页游标

# 限定 subreddit 搜索
GET https://oauth.reddit.com/r/python/search
  ?q=async&restrict_sr=true&sort=top&t=month
```

### 2. 浏览 Subreddit

```bash
# 热门帖子
GET https://oauth.reddit.com/r/{subreddit}/hot?limit=25

# 最新帖子
GET https://oauth.reddit.com/r/{subreddit}/new?limit=25

# 时间段最热
GET https://oauth.reddit.com/r/{subreddit}/top?t=week&limit=25

# 上升趋势
GET https://oauth.reddit.com/r/{subreddit}/rising?limit=25
```

### 3. 获取帖子评论

```bash
GET https://oauth.reddit.com/r/{subreddit}/comments/{post_id}
  ?sort=top         # top | best | new | controversial | old
  &limit=100        # 顶层评论数
  &depth=3          # 评论嵌套深度

# 示例（从 URL 提取 post_id）
# https://www.reddit.com/r/python/comments/abc123/title/
# → GET /r/python/comments/abc123
```

**响应结构**：返回数组，`[0]` 是帖子本身，`[1]` 是评论树。

### 4. Subreddit 详情

```bash
GET https://oauth.reddit.com/r/{subreddit}/about

# 响应包含：
# subscribers、active_user_count、description、created_utc、over18
```

### 5. 用户信息

```bash
# 用户概览
GET https://oauth.reddit.com/user/{username}/about

# 用户帖子
GET https://oauth.reddit.com/user/{username}/submitted
  ?sort=top&t=all&limit=25

# 用户评论
GET https://oauth.reddit.com/user/{username}/comments
  ?sort=top&t=month&limit=25
```

### 6. 无需认证的只读访问

```bash
# 公开帖子直接加 .json 后缀
curl "https://www.reddit.com/r/programming/hot.json?limit=10" \
  -H "User-Agent: myapp/1.0"

# 搜索
curl "https://www.reddit.com/search.json?q=python&sort=top&t=week" \
  -H "User-Agent: myapp/1.0"
```

> **注意**：不带认证访问有更严格的速率限制（约 10 req/min），正式使用建议用 OAuth。

---

## 响应数据结构

```json
{
  "data": {
    "children": [
      {
        "kind": "t3",
        "data": {
          "id": "abc123",
          "title": "帖子标题",
          "author": "用户名",
          "subreddit": "python",
          "score": 1234,
          "upvote_ratio": 0.95,
          "num_comments": 56,
          "url": "https://...",
          "selftext": "帖子正文（文字帖）",
          "created_utc": 1709452800,
          "permalink": "/r/python/comments/abc123/title/"
        }
      }
    ],
    "after": "t3_nextid",   # 翻页游标
    "before": null
  }
}
```

**Kind 前缀说明**：
- `t1` = 评论
- `t2` = 用户
- `t3` = 帖子（Link/Submission）
- `t4` = 私信
- `t5` = Subreddit

---

## 速率限制

| 认证方式 | 限制 |
|----------|------|
| OAuth（script app） | 60 req/min |
| 无认证 | ~10 req/min |
| PRAW 库 | 自动处理，内置限速 |

响应 Header 中包含：`X-Ratelimit-Used`、`X-Ratelimit-Remaining`、`X-Ratelimit-Reset`

---

完整实现代码见 `references/implementation.md`。
