# Bilibili API 实现参考

## 通用请求封装（TypeScript）

```typescript
const BILIBILI_BASE = 'https://api.bilibili.com';
const HEADERS = {
  'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36',
  'Referer': 'https://www.bilibili.com',
};

async function biliGet(path: string, params: Record<string, any> = {}) {
  const p = new URLSearchParams(params);
  const res = await fetch(`${BILIBILI_BASE}${path}?${p}`, { headers: HEADERS });
  const data = await res.json();
  if (data.code !== 0) throw new Error(`B站API失败 [${data.code}]: ${data.message}`);
  return data.data;
}
```

## 视频详情

```typescript
export async function getVideoInfo(bvid: string) {
  return biliGet('/x/web-interface/view', { bvid });
  // 返回：title, desc, duration, cid, owner, stat(view/like/danmaku/reply/coin/favorite)
}
```

## 搜索视频

```typescript
export async function searchVideos(keyword: string, opts?: {
  page?: number; pagesize?: number;
  order?: 'totalrank' | 'click' | 'pubdate' | 'dm' | 'stow' | 'scores';
  duration?: 0 | 1 | 2 | 3 | 4; // 0=全部,1=<10分,2=10-30分,3=30-60分,4=>60分
}) {
  return biliGet('/x/web-interface/search/type', {
    search_type: 'video', keyword,
    page: opts?.page || 1, pagesize: opts?.pagesize || 20,
    order: opts?.order || 'totalrank', duration: opts?.duration || 0,
  });
}
```

## UP 主信息和视频

```typescript
export async function getUserInfo(mid: number | string) {
  return biliGet('/x/space/acc/info', { mid });
  // 返回：name, sign(简介), face(头像URL), fans(粉丝数)
}

export async function getUserVideos(mid: number | string, page = 1, pagesize = 30) {
  return biliGet('/x/space/arc/search', { mid, pn: page, ps: pagesize, order: 'pubdate' });
}
```

## 弹幕获取与解析

```typescript
export async function getDanmaku(cid: number): Promise<string> {
  const res = await fetch(`https://comment.bilibili.com/${cid}.xml`, { headers: HEADERS });
  return res.text();
}

// 解析弹幕 XML -> 结构化数组
export function parseDanmaku(xml: string) {
  const matches = [...xml.matchAll(/<d p="([^"]+)">([^<]+)<\/d>/g)];
  return matches.map(m => {
    const [time, type] = m[1].split(',');
    return { time: parseFloat(time), type: parseInt(type), text: m[2] };
  }).sort((a, b) => a.time - b.time);
}
```

## yt-dlp 集成（Python）

```python
import subprocess, json

def get_video_info(url: str) -> dict:
    result = subprocess.run(['yt-dlp', '--dump-json', '--no-playlist', url],
                            capture_output=True, text=True)
    return json.loads(result.stdout)

def download_subtitles(url: str, lang='zh-Hans', output_dir='/tmp'):
    subprocess.run(['yt-dlp', '--write-sub', '--skip-download',
                    '--sub-lang', lang, '--output', f'{output_dir}/%(id)s', url], check=True)
    # 字幕文件：/tmp/{bvid}.{lang}.vtt

def get_playlist(url: str) -> list:
    result = subprocess.run(['yt-dlp', '--flat-playlist', '--dump-json', url],
                            capture_output=True, text=True)
    return [json.loads(line) for line in result.stdout.strip().split('\n') if line]
```

## URL 解析

```typescript
export function parseBilibiliUrl(url: string) {
  return {
    bvid: url.match(/bilibili\.com\/video\/(BV\w+)/)?.[1],
    aid: url.match(/bilibili\.com\/video\/av(\d+)/i)?.[1],
    uid: url.match(/space\.bilibili\.com\/(\d+)/)?.[1],
  };
}
```

## 主要分区 tids

| ID | 分区 | ID | 分区 |
|----|------|----|------|
| 0  | 全部 | 36 | 知识 |
| 1  | 动画 | 188| 科技 |
| 3  | 音乐 | 160| 生活 |
| 4  | 游戏 | 217| 美食 |
| 5  | 娱乐 | 234| 运动 |

## 注意事项

- 无需 API Key
- 搜索部分场景需要 `SESSDATA` Cookie
- 请求频率建议 ≤ 1 req/秒
- 大会员内容需要登录 Cookie：`yt-dlp --cookies-from-browser chrome`
