---
name: youtube
description: 使用 YouTube Data API v3 搜索视频、获取视频/频道/播放列表详情、读取评论、提取字幕。当用户提到 YouTube 链接、想搜索视频、分析频道、获取视频文字稿/字幕、查看热门视频，或任何涉及 YouTube 内容的任务时，都应使用此 skill。需要环境变量 YOUTUBE_API_KEY。
---

# YouTube Data API v3 Skill

通过 YouTube Data API v3 搜索、分析和提取 YouTube 内容。

## 认证

```
Base URL: https://www.googleapis.com/youtube/v3
参数：key=YOUTUBE_API_KEY（所有请求必须附带）
```

> API Key 从环境变量 `YOUTUBE_API_KEY` 读取。
> 在 [Google Cloud Console](https://console.cloud.google.com/apis/credentials) 获取，启用 YouTube Data API v3。

---

## 可用 Endpoints 总览

| Endpoint | 功能 | 配额消耗 |
|----------|------|----------|
| `GET /search` | 搜索视频/频道/播放列表 | **100 units**（最贵） |
| `GET /videos` | 获取视频详情、统计、字幕信息 | 1 unit |
| `GET /channels` | 获取频道详情和统计 | 1 unit |
| `GET /playlists` | 获取播放列表信息 | 1 unit |
| `GET /playlistItems` | 获取播放列表内的视频 | 1 unit |
| `GET /commentThreads` | 获取视频评论列表 | 1 unit |
| `GET /captions` | 列出字幕轨道 | 50 units |

> **每日配额上限：10,000 units**。search 最贵，尽量用 videoId 直接查询替代反复搜索。

---

## 场景判断

- 用户提供 YouTube 链接 → 提取 videoId/channelId，直接调 `videos` 或 `channels`
- 用户搜索关键词 → `search`（注意消耗 100 units）
- 用户要看频道最近视频 → `channels` 获取 uploadsPlaylistId → `playlistItems`
- 用户要获取字幕/文字稿 → `captions` 列出轨道，再用 `yt-dlp` 下载
- 用户要看评论 → `commentThreads`
- 用户要看热门视频 → `videos?chart=mostPopular`

---

## Endpoint 1: search

```bash
GET https://www.googleapis.com/youtube/v3/search
  ?part=snippet
  &q=搜索词
  &type=video          # video | channel | playlist
  &maxResults=10       # 1-50，默认 5
  &order=relevance     # relevance | date | viewCount | rating
  &publishedAfter=2024-01-01T00:00:00Z   # ISO 8601
  &regionCode=CN       # 地区代码
  &videoDuration=medium  # short(<4min) | medium(4-20min) | long(>20min)
  &key=YOUTUBE_API_KEY
```

**响应关键字段**：
```json
{
  "items": [{
    "id": { "videoId": "xxx", "channelId": "UCxxx" },
    "snippet": {
      "title": "标题",
      "channelTitle": "频道名",
      "publishedAt": "2024-01-15T10:00:00Z",
      "description": "描述"
    }
  }],
  "nextPageToken": "用于翻页"
}
```

---

## Endpoint 2: videos（获取视频详情）

```bash
GET https://www.googleapis.com/youtube/v3/videos
  ?part=snippet,statistics,contentDetails
  &id=videoId1,videoId2    # 逗号分隔，最多 50 个
  &chart=mostPopular       # 获取热门视频（与 id 二选一）
  &regionCode=US
  &key=YOUTUBE_API_KEY
```

**part 参数说明**：
- `snippet`：标题、描述、发布时间、频道信息、缩略图
- `statistics`：viewCount、likeCount、commentCount
- `contentDetails`：duration（ISO 8601，如 `PT4M13S`）、caption（true/false）

**从 URL 提取 videoId**：
- `https://youtu.be/dQw4w9WgXcQ` → `dQw4w9WgXcQ`
- `https://www.youtube.com/watch?v=dQw4w9WgXcQ` → `dQw4w9WgXcQ`

---

## Endpoint 3: channels

```bash
GET https://www.googleapis.com/youtube/v3/channels
  ?part=snippet,statistics,contentDetails
  &id=UCxxxxxx              # channel ID
  &forHandle=@channelname   # 或用 handle
  &key=YOUTUBE_API_KEY
```

`contentDetails.relatedPlaylists.uploads` 返回该频道上传视频的 playlistId，配合 playlistItems 使用。

---

## Endpoint 4: playlistItems（频道最新视频）

```bash
GET https://www.googleapis.com/youtube/v3/playlistItems
  ?part=snippet,contentDetails
  &playlistId=UUxxxxxx      # 频道 uploads playlist ID
  &maxResults=20
  &pageToken=               # 翻页
  &key=YOUTUBE_API_KEY
```

---

## Endpoint 5: commentThreads

```bash
GET https://www.googleapis.com/youtube/v3/commentThreads
  ?part=snippet
  &videoId=视频ID
  &maxResults=20
  &order=relevance          # relevance | time
  &searchTerms=关键词        # 可选，过滤评论内容
  &key=YOUTUBE_API_KEY
```

---

## 字幕获取（两种方式）

### 方式一：API（需要字幕所有者授权，限制多）
```bash
GET /captions?part=snippet&videoId=xxx&key=API_KEY
# 列出字幕轨道，下载需要 OAuth
```

### 方式二：yt-dlp（推荐，无需认证）
```bash
# 下载自动生成字幕（中文）
yt-dlp --write-auto-sub --sub-lang zh-Hans --skip-download \
  --output "%(title)s" "https://www.youtube.com/watch?v=VIDEO_ID"

# 下载所有可用字幕
yt-dlp --list-subs "https://www.youtube.com/watch?v=VIDEO_ID"

# 提取为纯文本
yt-dlp --write-auto-sub --sub-lang en --skip-download \
  --convert-subs srt "URL"
```

---

## 常用参数速查

| 参数 | 说明 |
|------|------|
| `part` | 返回哪些字段，用逗号分隔 |
| `maxResults` | 每页数量（1-50） |
| `pageToken` | 翻页 token（来自上次响应的 `nextPageToken`） |
| `regionCode` | 两字母地区码，如 `US`, `CN`, `JP` |
| `order` | 排序：`date`=最新, `viewCount`=最多观看, `relevance`=相关性 |

---

完整实现代码见 `references/implementation.md`。
