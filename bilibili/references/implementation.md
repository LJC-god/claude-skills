# B站 Skill 实现参考

## Python 封装

```python
import subprocess
import json
import re
from typing import Optional

def get_video_info(url: str) -> dict:
    """获取B站视频完整信息（通过 yt-dlp）"""
    result = subprocess.run(
        ['yt-dlp', '--dump-json', '--no-download', '--no-playlist', url],
        capture_output=True, text=True, timeout=30
    )
    if result.returncode != 0:
        raise Exception(f'yt-dlp 失败: {result.stderr}')
    return json.loads(result.stdout)

def get_subtitles(url: str, lang: str = 'zh-CN', output_dir: str = '/tmp') -> Optional[str]:
    """获取B站视频字幕，返回纯文本内容"""
    import tempfile, os
    
    output = os.path.join(output_dir, 'bili_sub_%(id)s')
    result = subprocess.run([
        'yt-dlp',
        '--write-auto-subs', '--write-subs',
        '--sub-langs', f'{lang},zh-Hans,zh,ai-zh',
        '--skip-download',
        '--output', output,
        url
    ], capture_output=True, text=True, timeout=60)
    
    # 找字幕文件
    for ext in ['vtt', 'srt', 'ass']:
        import glob
        files = glob.glob(f'{output_dir}/bili_sub_*.{ext}')
        if files:
            content = open(files[0]).read()
            # 清理时间轴，提取纯文本
            if ext == 'vtt':
                lines = [l for l in content.split('\n') 
                         if not l.startswith('WEBVTT') 
                         and '-->' not in l 
                         and not re.match(r'^\d+$', l.strip())
                         and l.strip()]
            else:
                lines = [l for l in content.split('\n')
                         if '-->' not in l
                         and not re.match(r'^\d+$', l.strip())
                         and l.strip()]
            return '\n'.join(dict.fromkeys(lines))  # 去重
    
    return None

def search_bilibili(keyword: str, limit: int = 10) -> list:
    """搜索B站视频"""
    result = subprocess.run(
        ['yt-dlp', '--dump-json', '--no-download', '--flat-playlist',
         f'bilisearch:{keyword}', '--playlist-end', str(limit)],
        capture_output=True, text=True, timeout=60
    )
    videos = []
    for line in result.stdout.strip().split('\n'):
        if line:
            try:
                videos.append(json.loads(line))
            except:
                pass
    return videos

def get_user_videos(uid: int, limit: int = 20) -> list:
    """获取UP主视频列表"""
    result = subprocess.run(
        ['yt-dlp', '--flat-playlist', '--dump-json', '--no-download',
         '--playlist-end', str(limit),
         f'https://space.bilibili.com/{uid}/video'],
        capture_output=True, text=True, timeout=60
    )
    videos = []
    for line in result.stdout.strip().split('\n'):
        if line:
            try:
                videos.append(json.loads(line))
            except:
                pass
    return videos
```

## 快速信息提取

```python
def format_video_summary(info: dict) -> str:
    """格式化视频信息摘要"""
    duration = info.get('duration', 0)
    mins, secs = divmod(duration, 60)
    
    date_str = info.get('upload_date', '')
    if date_str:
        date_str = f"{date_str[:4]}-{date_str[4:6]}-{date_str[6:]}"
    
    return f"""
📺 {info.get('title', '未知标题')}
👤 UP主：{info.get('uploader', '未知')} (uid:{info.get('uploader_id', '')})
📅 发布：{date_str}
⏱️ 时长：{mins}分{secs}秒
👁️ 播放：{info.get('view_count', 0):,}
👍 点赞：{info.get('like_count', 0):,}
🏷️ 标签：{' '.join(info.get('tags', [])[:6])}
🔗 链接：{info.get('webpage_url', '')}
📝 简介：{info.get('description', '')[:200]}
""".strip()
```

## 常用 URL 格式

```
视频页：
  https://www.bilibili.com/video/BV{id}
  https://www.bilibili.com/video/av{id}
  https://b23.tv/{短链}

UP主页：
  https://space.bilibili.com/{uid}/video

播放列表：
  https://www.bilibili.com/medialist/play/{uid}?business=space_collection&business_id={id}

直播间：
  https://live.bilibili.com/{room_id}

番剧：
  https://www.bilibili.com/bangumi/play/ss{id}
  https://www.bilibili.com/bangumi/play/ep{id}
```

## yt-dlp 常用选项速查

```bash
# 视频信息（不下载）
--dump-json --no-download

# 字幕
--write-subs --write-auto-subs --sub-langs "zh-CN"
--convert-subs srt     # 转 srt 格式
--skip-download        # 不下载视频本体

# 播放列表
--flat-playlist        # 只列出视频，不获取详情（快）
--playlist-start N     # 从第N个开始
--playlist-end N       # 到第N个结束

# 输出
--output "%(title)s.%(ext)s"    # 文件命名模板
--print "%(title)s"             # 打印特定字段

# 清晰度
-f "bestvideo+bestaudio"        # 最高清晰度
-f "worst"                      # 最低清晰度（快速测试）

# 登录（获取需要登录的内容）
--cookies-from-browser chrome   # 从浏览器读取 cookies
--cookies /path/to/cookies.txt  # 使用 cookies 文件
```

## 字幕语言代码（B站常见）

| 代码 | 说明 |
|------|------|
| `zh-CN` | 简体中文 |
| `zh-Hans` | 简体中文（备用） |
| `zh-TW` | 繁体中文 |
| `ai-zh` | AI 生成中文字幕 |
| `en` | 英文 |
| `ja` | 日文 |

## 注意事项

1. **版权合规**：yt-dlp 获取信息可用于个人学习分析，请勿大规模商业爬取
2. **反爬限制**：B站有速率限制，批量获取时建议加 `--sleep-interval 2` 延迟
3. **登录内容**：大会员视频需要提供 cookies，使用 `--cookies-from-browser chrome`
4. **API 稳定性**：B站非公开接口随时可能变更，yt-dlp 会定期更新适配

## 安装检查

```bash
# 检查 yt-dlp 版本
yt-dlp --version

# 更新到最新版
pip install yt-dlp --upgrade

# 快速测试
yt-dlp --dump-json --no-download "https://www.bilibili.com/video/BV1GJ411x7h7" | \
  python3 -c "import json,sys; d=json.load(sys.stdin); print(d['title'])"
```
