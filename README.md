# Claude Skills

Claude AI 自定义 Skill 合集，可直接上传到 [claude.ai](https://claude.ai) 的 Skills 设置中使用。

## 包含 Skills

| Skill | 功能 | 环境变量 |
|-------|------|----------|
| [mcp-jzl](./mcp-jzl/) | 微信公众号文章提取（极致了 API） | `JZL_API_KEY` |
| [tavily](./tavily/) | LLM 优化网络搜索与内容提取 | `TAVILY_API_KEY` |
| [brave](./brave/) | Brave Search 隐私优先网络搜索 | `BRAVE_API_KEY` |
| [notion](./notion/) | Notion 工作区完整操作（MCP） | Notion MCP 已连接 |
| [youtube](./youtube/) | YouTube 视频搜索、频道分析、字幕提取 | `YOUTUBE_API_KEY` |
| [reddit](./reddit/) | Reddit 帖子搜索、社区浏览、评论分析 | `REDDIT_CLIENT_ID` + `REDDIT_CLIENT_SECRET` |
| [bilibili](./bilibili/) | B站视频信息、字幕提取、UP主分析 | 无需 API Key（使用 yt-dlp） |

## 安装方法

1. 下载对应的 `.skill` 文件
2. 打开 Claude.ai → 设置 → Skills → 上传 `.skill` 文件
3. 配置对应的环境变量（bilibili 无需配置）

## 获取 API Keys

- **YouTube**：[Google Cloud Console](https://console.cloud.google.com/apis/credentials)，启用 YouTube Data API v3
- **Reddit**：[Reddit App Preferences](https://www.reddit.com/prefs/apps)，创建 script 类型应用
- **Brave**：[Brave Search API](https://api.search.brave.com/)
- **Tavily**：[Tavily](https://app.tavily.com/)
- **极致了（mcp-jzl）**：[大家啦](https://www.dajiala.com/)
- **Bilibili**：无需 API Key，依赖 yt-dlp 工具

## 文件结构

每个 Skill 包含：
- `SKILL.md` — 触发逻辑与使用指南
- `references/` — 详细 API 文档和实现代码
- `*.skill` — 打包好的安装文件
