---
name: bilibili
description: 获取 B站（哔哩哔哩）视频信息、UP主数据、搜索视频、下载字幕/弹幕。当用户提到 B站链接、BV号、AV号、想搜索B站视频、获取UP主信息、下载视频字幕，或任何涉及哔哩哔哩内容的任务时，都应使用此 skill。支持通过 yt-dlp 下载视频信息，以及调用 B站公开 web 接口。无需 API Key，部分功能需要登录 Cookie。
---

# Bilibili Skill

获取 B站视频、UP主、搜索数据。B站没有官方开放 API（开放平台仅供企业），本 skill 使用 **B站 web 端公开接口** 和 **yt-dlp** 实现功能。

> ⚠️ **重要说明**：本 skill 仅使用浏览器正常访问时也会调用的公开 web 接口。请合理使用，勿高频请求。

---

## ID 格式说明

| 格式 | 示例 | 说明 |
|------|------|------|
| BV 号 | `BV1uT4y1P7CX` | 当前主要 ID 格式，URL 中可见 |
| AV 号 | `av170001` | 旧格式，仍可用 |
| UID | `12345678` | UP主用户 ID，在 space.bilibili.com/UID |
| 短链 | `b23.tv/xxxxx` | 重定向到完整链接 |

**从 URL 提取 ID**：
- `https://www.bilibili.com/video/BV1uT4y1P7CX` → `BV1uT4y1P7CX`
- `https://space.bilibili.com/12345678` → UID `12345678`

---

## 方式一：yt-dlp（推荐，功能最全）

```bash
# 安装
pip install yt-dlp

# 获取视频基本信息（不下载）
yt-dlp --dump-json "https://www.bilibili.com/video/BV1uT4y1P7CX"

# 列出所有分P
yt-dlp --flat-playlist "https://www.bilibili.com/video/BV1uT4y1P7CX"

# 下载字幕（CC字幕）
yt-dlp --write-sub --skip-download --sub-lang zh-Hans \
  "https://www.bilibili.com/video/BV1uT4y1P7CX"

# 解析关键信息
yt-dlp --dump-json "URL" | python3 -c "
import sys,json
d=json.load(sys.stdin)
print('标题:', d['title'])
print('UP主:', d['uploader'])
print('播放量:', d.get('view_count'))
print('点赞:', d.get('like_count'))
print('时长:', d['duration'], '秒')
print('简介:', d['description'][:200])
"

# 需要登录的视频（传入浏览器 Cookie）
yt-dlp --cookies-from-browser chrome --dump-json "URL"
```

---

## 方式二：B站公开 Web 接口

所有接口均为 GET 请求，**无需 API Key**。

```
公共 Header：
  User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64)
  Referer: https://www.bilibili.com
```

### 1. 获取视频详情

```bash
GET https://api.bilibili.com/x/web-interface/view?bvid=BV1uT4y1P7CX

# 响应关键字段：
# data.title          视频标题
# data.desc           视频简介
# data.owner.name     UP主名称
# data.owner.mid      UP主 UID
# data.stat.view      播放量
# data.stat.like      点赞数
# data.stat.danmaku   弹幕数
# data.stat.reply     评论数
# data.stat.coin      投币数
# data.stat.favorite  收藏数
# data.duration       时长（秒）
# data.pic            封面图 URL
# data.pages          分P列表
# data.cid            cid（下载弹幕用）
```

### 2. 搜索视频

```bash
GET https://api.bilibili.com/x/web-interface/search/type
  ?search_type=video
  &keyword=搜索词
  &page=1
  &pagesize=20
  &order=totalrank      # totalrank=综合 | click=播放量 | pubdate=最新 | dm=弹幕 | stow=收藏

# 响应：data.result 数组
```

### 3. 获取 UP 主信息

```bash
GET https://api.bilibili.com/x/space/acc/info?mid=UID

# 响应：data.name、data.sign（简介）、data.fans（粉丝数）、data.face（头像URL）
```

### 4. 获取 UP 主视频列表

```bash
GET https://api.bilibili.com/x/space/arc/search
  ?mid=UID&pn=1&ps=30&order=pubdate
```

### 5. 获取视频评论

```bash
GET https://api.bilibili.com/x/v2/reply
  ?type=1&oid=AV号数字&pn=1&ps=20&sort=2
# sort: 0=按时间 | 2=按热度
```

### 6. 获取弹幕（XML）

```bash
GET https://comment.bilibili.com/{cid}.xml
# cid 从 /view 接口的 data.cid 获取
# 每条 <d> 元素属性格式："时间偏移,类型,字体,颜色,时间戳,池,用户hash,ID"
```

### 7. 热门排行榜

```bash
# 全站排行榜
GET https://api.bilibili.com/x/web-interface/ranking/v2?rid=0&type=all

# 热门视频
GET https://api.bilibili.com/x/web-interface/popular?pn=1&ps=20
```

---

## 场景判断

| 用户需求 | 使用方式 |
|----------|----------|
| 提供链接/BV号，要视频信息 | web 接口 `/view` 或 yt-dlp `--dump-json` |
| 搜索视频关键词 | web 接口 `/search/type` |
| 获取 UP 主资料和视频 | web 接口 `/space/acc/info` + `/arc/search` |
| 获取字幕/文字稿 | yt-dlp `--write-sub` |
| 获取弹幕内容分析 | `comment.bilibili.com/{cid}.xml` |
| 需要登录才能访问 | yt-dlp `--cookies-from-browser chrome` |

---

## 注意事项

1. **不需要 API Key**，直接调用即可
2. 搜索接口有时需要 Cookie（`SESSDATA`）才能返回完整结果
3. 请勿高频请求，建议每次请求间隔 1 秒以上
4. 部分视频（大会员专属、地区限制）无法免登录访问
5. BV 号可以直接用，无需转换为 AV 号

---

完整实现代码见 `references/implementation.md`。
