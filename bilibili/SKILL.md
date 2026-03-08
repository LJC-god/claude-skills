---
name: bilibili
description: 获取哔哩哔哩（B站）视频信息、字幕、UP主数据和搜索内容。当用户提到 B站/哔哩哔哩链接、想获取视频信息、下载字幕/文字稿、分析UP主、搜索B站视频，或任何涉及 bilibili.com 内容的任务时，都应使用此 skill。主要通过 yt-dlp 工具实现，无需 API Key。
---

# B站（哔哩哔哩）Skill

通过 yt-dlp 和 B站公开接口获取视频信息、字幕和UP主数据。

## ⚠️ 重要说明

B站于 2026 年 1 月向第三方 API 文档项目发律师函，**不欢迎直接调用非公开接口**。
本 skill 采用两种合规方式：

1. **yt-dlp**：业界标准的视频信息提取工具，合法使用
2. **B站官方开放平台**：[openhome.bilibili.com](https://openhome.bilibili.com/doc)（需申请 AppKey）

---

## 工具选择

| 场景 | 推荐方式 |
|------|----------|
| 获取视频信息、字幕 | yt-dlp（无需认证） |
| 批量分析、数据研究 | yt-dlp + jq 解析 |
| 商业级 UP主/创作者数据 | B站开放平台 API |
| 下载视频 | yt-dlp（遵守版权） |

---

## 方式一：yt-dlp（推荐）

### 安装
```bash
pip install yt-dlp
# 或
pip install yt-dlp --upgrade
```

### 获取视频信息（JSON）
```bash
# 通过 BV 号或 AV 号
yt-dlp --dump-json --no-download "https://www.bilibili.com/video/BVxxxxxx"

# 通过 av 号
yt-dlp --dump-json --no-download "https://www.bilibili.com/video/av170001"

# 只提取关键字段
yt-dlp --dump-json --no-download "URL" | python3 -c "
import json, sys
d = json.load(sys.stdin)
print(f'标题：{d[\"title\"]}')
print(f'UP主：{d[\"uploader\"]}')
print(f'播放量：{d.get(\"view_count\", \"N/A\")}')
print(f'点赞：{d.get(\"like_count\", \"N/A\")}')
print(f'弹幕：{d.get(\"comment_count\", \"N/A\")}')
print(f'时长：{d.get(\"duration\", 0)}秒')
print(f'发布时间：{d.get(\"upload_date\", \"\")}')
print(f'描述：{d.get(\"description\", \"\")[:200]}')
"
```

### 可获取的字段
```
title          - 视频标题
uploader       - UP主名称
uploader_id    - UP主 uid
upload_date    - 发布日期（YYYYMMDD格式）
duration       - 时长（秒）
view_count     - 播放量
like_count     - 点赞数
comment_count  - 评论/弹幕数
description    - 视频简介
tags           - 标签列表
thumbnail      - 封面图 URL
webpage_url    - 视频链接
formats        - 可用清晰度列表
```

### 获取字幕/字幕文字稿

```bash
# 列出所有可用字幕
yt-dlp --list-subs "https://www.bilibili.com/video/BVxxxxxx"

# 下载中文字幕（AI 生成）
yt-dlp --write-subs --write-auto-subs \
  --sub-langs "zh-CN,zh-Hans,ai-zh" \
  --skip-download \
  --output "/tmp/%(id)s.%(ext)s" \
  "https://www.bilibili.com/video/BVxxxxxx"

# 转换为 srt 格式
yt-dlp --write-auto-subs --sub-langs "zh-CN" \
  --convert-subs srt --skip-download \
  --output "/tmp/%(title)s" \
  "URL"

# 提取纯文本（去掉时间轴）
cat /tmp/subtitles.srt | grep -v "^[0-9]" | grep -v "^-->" | grep -v "^$"
```

### 获取播放列表/UP主视频列表

```bash
# UP主所有视频（只获取信息，不下载）
yt-dlp --dump-json --no-download --flat-playlist \
  "https://space.bilibili.com/UID/video"

# 播放列表
yt-dlp --dump-json --no-download --flat-playlist \
  "https://www.bilibili.com/medialist/play/UID?business=space_collection&business_id=LISTID"
```

### 搜索视频

B站搜索结果通过网页获取，yt-dlp 支持搜索：
```bash
# 搜索关键词，获取前10个视频信息
yt-dlp --dump-json --no-download "bilisearch:Python教程" --playlist-end 10
```

---

## 方式二：B站开放平台 API（商业/正式场景）

官网：https://openhome.bilibili.com/doc

需要注册开发者账号并申请 AppKey，适合需要大量调用或商业用途的场景。

### 主要开放接口
- 视频数据查询（播放量、点赞、投币等）
- UP主粉丝数、视频列表
- 直播数据

---

## 从 URL 提取视频 ID

```python
import re

def parse_bilibili_url(url: str) -> dict:
    """解析 B站各种格式的 URL"""
    result = {}

    # BV 号
    bv = re.search(r'BV([a-zA-Z0-9]+)', url)
    if bv:
        result['bvid'] = 'BV' + bv.group(1)

    # AV 号
    av = re.search(r'(?:av|/video/av)(\d+)', url, re.I)
    if av:
        result['aid'] = int(av.group(1))

    # UP主 UID
    uid = re.search(r'space\.bilibili\.com/(\d+)', url)
    if uid:
        result['uid'] = int(uid.group(1))

    return result

# 示例
# https://www.bilibili.com/video/BV1GJ411x7h7 → {'bvid': 'BV1GJ411x7h7'}
# https://www.bilibili.com/video/av170001 → {'aid': 170001}
# https://space.bilibili.com/123456/video → {'uid': 123456}
```

---

## 常用完整工作流

### 工作流1：分析单个视频

```bash
URL="https://www.bilibili.com/video/BVxxxxxx"

# 获取完整信息
INFO=$(yt-dlp --dump-json --no-download "$URL")

# 解析关键数据
echo $INFO | python3 -c "
import json, sys
d = json.load(sys.stdin)
print('=== 视频分析 ===')
print(f'标题：{d[\"title\"]}')
print(f'UP主：{d[\"uploader\"]} (uid:{d.get(\"uploader_id\")})')
print(f'发布：{d.get(\"upload_date\",\"\")[:4]}-{d.get(\"upload_date\",\"\")[4:6]}-{d.get(\"upload_date\",\"\")[6:]}')
print(f'时长：{d.get(\"duration\",0)//60}分{d.get(\"duration\",0)%60}秒')
print(f'播放：{d.get(\"view_count\",0):,}')
print(f'点赞：{d.get(\"like_count\",0):,}')
print(f'标签：{\" \".join(d.get(\"tags\",[])[:5])}')
print(f'简介：{d.get(\"description\",\"\")[:300]}')
"
```

### 工作流2：提取视频字幕做总结

```bash
URL="https://www.bilibili.com/video/BVxxxxxx"

# 获取字幕
yt-dlp --write-auto-subs --sub-langs "zh-CN,zh-Hans,zh" \
  --skip-download --output "/tmp/bili_sub" "$URL"

# 找到字幕文件并提取纯文本
find /tmp -name "bili_sub*.vtt" -o -name "bili_sub*.srt" 2>/dev/null | head -1 | \
  xargs cat | grep -v "^[0-9:\.]" | grep -v "^-->" | grep -v "^$" | \
  head -200
```

### 工作流3：获取UP主最新视频列表

```bash
UID=12345678  # UP主UID（从 space.bilibili.com/UID 获取）

yt-dlp --flat-playlist --dump-json --no-download \
  --playlist-end 20 \
  "https://space.bilibili.com/$UID/video" | \
  python3 -c "
import sys
for line in sys.stdin:
    import json
    v = json.loads(line)
    print(f'{v.get(\"id\",\"\")} | {v.get(\"title\",\"\")}')
"
```

---

完整实现代码见 `references/implementation.md`。
