# YouTube Data API v3 实现参考

## 通用请求封装（TypeScript）

```typescript
const YT_BASE = 'https://www.googleapis.com/youtube/v3';

async function ytGet(endpoint: string, params: Record<string, any>) {
  const apiKey = process.env.YOUTUBE_API_KEY;
  if (!apiKey) throw new Error('YOUTUBE_API_KEY 环境变量未设置');

  const p = new URLSearchParams({ key: apiKey });
  Object.entries(params).forEach(([k, v]) => {
    if (v !== undefined && v !== null) p.set(k, String(v));
  });

  const res = await fetch(`${YT_BASE}/${endpoint}?${p}`);
  if (!res.ok) {
    const err = await res.json();
    throw new Error(`YouTube API 失败 [${res.status}]: ${err.error?.message}`);
  }
  return await res.json();
}
```

## 搜索视频

```typescript
export async function searchVideos(query: string, opts?: {
  maxResults?: number; order?: 'relevance'|'date'|'viewCount'|'rating';
  publishedAfter?: string; regionCode?: string;
  videoDuration?: 'short'|'medium'|'long'; channelId?: string;
}) {
  return ytGet('search', {
    part: 'snippet', q: query, type: 'video', maxResults: 10, ...opts
  });
}
```

## 获取视频详情

```typescript
export async function getVideoDetails(videoIds: string | string[]) {
  const ids = Array.isArray(videoIds) ? videoIds.join(',') : videoIds;
  return ytGet('videos', {
    part: 'snippet,statistics,contentDetails',
    id: ids,
  });
}

// 解析时长（ISO 8601 → 秒）
export function parseDuration(iso: string): number {
  const match = iso.match(/PT(?:(\d+)H)?(?:(\d+)M)?(?:(\d+)S)?/);
  if (!match) return 0;
  return (parseInt(match[1]||'0') * 3600) + (parseInt(match[2]||'0') * 60) + parseInt(match[3]||'0');
}
```

## 获取频道最新视频

```typescript
export async function getChannelVideos(channelId: string, maxResults = 20) {
  // Step 1: 获取 uploads playlist ID
  const channel = await ytGet('channels', {
    part: 'contentDetails', id: channelId
  });
  const uploadsId = channel.items[0]?.contentDetails?.relatedPlaylists?.uploads;
  if (!uploadsId) throw new Error('找不到频道上传列表');

  // Step 2: 获取视频列表
  return ytGet('playlistItems', {
    part: 'snippet,contentDetails',
    playlistId: uploadsId,
    maxResults,
  });
}
```

## 获取评论

```typescript
export async function getComments(videoId: string, opts?: {
  maxResults?: number; order?: 'relevance'|'time'; searchTerms?: string;
}) {
  return ytGet('commentThreads', {
    part: 'snippet', videoId, maxResults: 20, ...opts
  });
}
```

## 获取热门视频

```typescript
export async function getTrendingVideos(regionCode = 'US', maxResults = 10) {
  return ytGet('videos', {
    part: 'snippet,statistics',
    chart: 'mostPopular',
    regionCode,
    maxResults,
  });
}
```

## 从 URL 提取 videoId

```typescript
export function extractVideoId(url: string): string | null {
  const patterns = [
    /youtu\.be\/([^?&]+)/,
    /youtube\.com\/watch\?v=([^&]+)/,
    /youtube\.com\/embed\/([^?]+)/,
    /youtube\.com\/shorts\/([^?]+)/,
  ];
  for (const p of patterns) {
    const m = url.match(p);
    if (m) return m[1];
  }
  return null;
}
```

## MCP Server 工具列表

```typescript
tools: [
  { name: 'youtube_search', description: '搜索 YouTube 视频' },
  { name: 'youtube_video_details', description: '获取视频详情和统计数据' },
  { name: 'youtube_channel_details', description: '获取频道信息和最新视频' },
  { name: 'youtube_comments', description: '获取视频评论' },
  { name: 'youtube_trending', description: '获取地区热门视频' },
  { name: 'youtube_transcript', description: '用 yt-dlp 下载视频字幕' },
]
```

## 字幕下载（yt-dlp shell 命令）

```bash
# 安装
pip install yt-dlp

# 下载自动字幕
yt-dlp --write-auto-sub --sub-lang en --skip-download \
  --output "/tmp/%(id)s" "https://youtu.be/VIDEO_ID"

# 列出可用字幕
yt-dlp --list-subs "https://youtu.be/VIDEO_ID"
```

## 配额成本速查

| 操作 | 单次成本 | 每日可用次数 |
|------|----------|-------------|
| search | 100 units | 100 次 |
| videos / channels / playlists | 1 unit | 10,000 次 |
| commentThreads | 1 unit | 10,000 次 |
| captions.list | 50 units | 200 次 |

## 环境变量

```bash
YOUTUBE_API_KEY=AIzaSy...你的Key
```
