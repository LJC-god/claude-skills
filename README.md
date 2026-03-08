# Claude Skills

Claude AI 自定义 Skill 合集，可直接上传到 [claude.ai](https://claude.ai) 的 Skills 设置中使用。

## 包含 Skills

| Skill | 功能 | 环境变量 |
|-------|------|----------|
| [mcp-jzl](./mcp-jzl/) | 微信公众号文章提取（极致了 API） | `JZL_API_KEY` |
| [tavily](./tavily/) | LLM 优化网络搜索与内容提取 | `TAVILY_API_KEY` |
| [brave](./brave/) | Brave Search 隐私优先网络搜索 | `BRAVE_API_KEY` |
| [notion](./notion/) | Notion 工作区完整操作（MCP） | Notion MCP 已连接 |

## 安装方法

1. 下载对应的 `.skill` 文件
2. 打开 Claude.ai → 设置 → Skills → 上传 `.skill` 文件
3. 配置对应的环境变量

## 文件结构

每个 Skill 包含：
- `SKILL.md` — 触发逻辑与使用指南
- `references/` — 详细 API 文档和实现代码
- `*.skill` — 打包好的安装文件
