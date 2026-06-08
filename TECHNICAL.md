# AI日报技术实现说明

> 文档生成日期：2026年4月8日 | 最后更新：2026年6月8日

---

## 📅 2026-06-08 重大技术升级

### ✅ 目录结构重构
- **新目录结构**：所有日报文件移至 `daily_report_file/` 目录
- **根目录**：只保留 README.md 和 TECHNICAL.md
- **优点**：更清晰的组织结构，便于扩展

### ✅ 新闻源扩展
- **新增SearXNG实时搜索**：聚合 Google + Bing 双引擎
- **时间过滤**：仅获取最近24小时新闻
- **JSON输出**：便于程序解析
- **多关键词策略**：不同主题分类搜索

### ✅ 定时任务配置更新
- **新Message格式**：详细的多步骤流程指引
- **自动聚合**：每次生成10-20条实时新闻
- **质量控制**：要求包含深度分析和来源标注

### ✅ 日报格式改进
- **新增章节**：深度分析、Hacker News讨论
- **新闻条数**：从5条提升至10条
- **内容深度**：每条新闻包含完整描述和影响分析

---

---

## 系统架构

```
┌─────────────────────────────────────────────────────────────┐
│                      OpenClaw Gateway                        │
│                    (systemd --user)                          │
└─────────────────────────────────────────────────────────────┘
                              │
            ┌─────────────────┼─────────────────┐
            ▼                 ▼                 ▼
    ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
    │  Cron调度器   │  │  会话管理    │  │  消息路由    │
    │  (内置)       │  │  (isolated) │  │  (钉钉)      │
    └──────────────┘  └──────────────┘  └──────────────┘
            │
            │ 定时触发 (14:00)
            ▼
    ┌──────────────────────────────────────────────────────┐
    │                   AI日报 Cron任务                      │
    │                 (sessionId: ce170b66)                 │
    └──────────────────────────────────────────────────────┘
            │
            ▼
    ┌──────────────────────────────────────────────────────┐
    │              独立Agent会话 (isolated)                  │
    │    - 加载AGENTS.md/SOUL.md等上下文                    │
    │    - 执行新闻搜索和内容生成                            │
    │    - 生成日报Markdown文件                             │
    └──────────────────────────────────────────────────────┘
            │
            ├──────────────────────┬───────────────────────┐
            ▼                      ▼                       ▼
    ┌──────────────┐      ┌──────────────┐      ┌──────────────┐
    │ SearXNG搜索  │      │ 内容生成     │      │ 钉钉通知    │
    │ (本地实例)   │      │ (LLM)        │      │ (delivery)  │
    └──────────────┘      └──────────────┘      └──────────────┘
```

---

## 定时任务配置

### Cron任务定义

**位置：** `~/.openclaw/cron/jobs.json`

```json
{
  "id": "ce170b66-885c-49b6-9242-2f2e5bf4fbc1",
  "name": "AI日报",
  "schedule": {
    "kind": "cron",
    "expr": "0 14 * * *",
    "tz": "Asia/Shanghai"
  },
  "sessionTarget": "isolated",
  "payload": {
    "kind": "agentTurn",
    "message": "搜索并整理AI新闻..."
  },
  "delivery": {
    "mode": "announce",
    "channel": "dingtalk",
    "to": "01431911541521448266"
  }
}
```

### 相关任务

| 任务名 | 时间 | 作用 |
|--------|------|------|
| AI日报 | 14:00 | 搜索新闻、生成日报内容 |
| AI日报GitHub推送 | 14:30 | 将日报推送到GitHub仓库 |

---

## 新闻获取流程

### 1. 数据源（2026-06-08升级）

| 来源 | 方式 | 用途 | 状态 |
|------|------|------|------|
| **SearXNG** | 本地实例 (localhost:8080) | 实时新闻聚合（Google+Bing） | ✅ 主要 |
| **Hacker News** | web_fetch + SearXNG | 技术社区讨论 | ✅ 新增 |
| web_fetch | HTTP抓取 | 深度获取文章内容 | ✅ 辅助 |
| Reddit | SearXNG索引（受限） | 社区讨论热点 | ⚠️ 部分可用 |
| Twitter/X | API受限（需手动链接） | 快速新闻源 | ⚠️ 需用户辅助 |

### 2. SearXNG实时搜索配置

```bash
# 实时新闻搜索（推荐）
curl "http://localhost:8080/search?q=artificial+intelligence+news+today&format=json&engines=google,bing&time_range=day&language=en"

# 多关键词组合搜索
curl "http://localhost:8080/search?q=OpenAI+Anthropic+Claude+GPT+news+2026&format=json&engines=google,bing&time_range=day"
```

### 3. 多关键词搜索策略（优化版）

```
AI日报搜索策略（2026-06-08）：
├── 实时新闻: "artificial intelligence news today June 2026"
├── 大模型进展: "OpenAI Claude GPT-5 Anthropic news"
├── AI工具更新: "AI tool coding assistant agent platform"
├── 行业动态:   "AI funding IPO acquisition valuation"
├── 技术突破:   "AI research breakthrough paper"
├── 市场影响:   "AI market stock backlash employment"
└── Hacker News: "AI machine learning LLM ycombinator"
```

### 4. 新闻聚合流程（新增）

```python
# 多源聚合示例
1. 搜索实时新闻（10-20条）
   → SearXNG聚合（Google + Bing）
   → 时间过滤（最近24小时）

2. 补充Hacker News讨论
   → web_fetch首页抓取
   → 提取高分帖子（>100分）

3. 深度获取关键新闻内容
   → web_fetch重要链接
   → 提取完整文章内容

4. 去重排序
   → URL去重
   → 时间排序（最新优先）
```

---

## 内容生成

### 生成流程

```
搜索新闻 → 去重筛选 → 提取摘要 → 生成日报 → 保存文件
```

### 日报结构

```markdown
# AI日报 - YYYY年MM月DD日

## 📰 今日要闻
1. 新闻标题 + 摘要 + 来源链接
2. ...
3. ...

## 💡 值得关注
深度分析1-2个重点事件

## 📊 趋势观察
简短总结行业趋势
```

### 文件存储（2026-06-08重构）

```
~/.openclaw/workspace/github/ai-dailyreport/
├── README.md              # 目录索引（保留在根目录）
├── TECHNICAL.md           # 技术文档（保留在根目录）
└── daily_report_file/     # 新目录：所有日报文件
    ├── 2026-06-08.md      # 日报文件（10条详细新闻）
    ├── 2026-06-07.md
    ├── 2026-03-18.md      # 首篇日报（82个历史文件）
    └── ...
```

**优点**：
- ✅ 根目录简洁（只保留2个核心文档）
- ✅ 日报文件集中管理
- ✅ 便于扩展更多文档类型

---

## GitHub推送

### 推送流程

```bash
cd ~/.openclaw/workspace/github/ai-dailyreport
git add .
git commit -m "docs: 添加AI日报 YYYY-MM-DD"
git push
```

### 仓库配置

```bash
# 远程仓库
git remote -v
origin  git@github.com:castlewuwu/ai-dailyreport.git

# 分支
main
```

### README更新

每次生成日报后，自动更新README.md的目录部分，新日期置顶。

---

## 通知机制

### Delivery配置

```json
{
  "delivery": {
    "mode": "announce",           // 公告模式
    "channel": "dingtalk",        // 钉钉通道
    "to": "01431911541521448266"  // 私聊ID
  }
}
```

### 通知流程

```
Cron任务完成
    │
    ▼
delivery.announce触发
    │
    ▼
消息发送到钉钉
    │
    ├── 成功: 用户收到通知
    └── 失败: 记录错误日志
```

---

## 关键依赖

| 组件 | 版本/配置 | 作用 |
|------|----------|------|
| OpenClaw | 2026.2.26 | 主框架 |
| Node.js | 24.14.0 | 运行时 |
| SearXNG | 本地实例 | 搜索聚合 |
| Git | - | 版本控制/推送 |
| DashScope | glm-5 | LLM模型 |

---

## 故障排查

### 常见问题

| 问题 | 原因 | 解决 |
|------|------|------|
| 任务不执行 | jobs.json损坏 | 恢复备份 |
| 消息不推送 | delivery配置错误 | 检查channel/to |
| 新闻获取失败 | SearXNG未启动 | 重启服务 |
| Git推送失败 | 仓库冲突 | git pull --rebase |

### 日志位置

```bash
# Cron运行记录
~/.openclaw/cron/runs/<session-id>.jsonl

# 任务配置
~/.openclaw/cron/jobs.json
```

---

## 改进方向（2026-06-08更新）

### 已完成 ✅
1. **新闻源扩展** - SearXNG实时搜索已集成（Google+Bing）
2. **目录重构** - daily_report_file目录已创建
3. **格式优化** - 添加深度分析和Hacker News讨论
4. **质量控制** - 新闻条数从5条提升至10条

### 进行中 🔄
1. **Reddit集成** - 部分可用，需解决API限制
2. **自动去重** - 多源合并后的URL去重机制
3. **内容深度化** - 自动提取关键新闻的完整内容

### 未来规划 📋
1. **RSS订阅** - 订阅OpenAI、Anthropic、Google AI官方博客
2. **Twitter集成** - 需用户手动提供重要推文链接
3. **自动分类** - 按主题自动打标签（资本、技术、产品等）
4. **多渠道推送** - 支持Telegram、邮件等
5. **历史分析** - 统计高频话题、趋势图表
6. **GitHub Trending** - 添加热门AI项目监控
7. **arXiv论文** - 集成最新AI学术研究

---

## 质量控制标准（2026-06-08新增）

### 日报内容标准

| 项目 | 标准 | 说明 |
|------|------|------|
| 新闻条数 | ≥10条 | 包含完整描述和来源 |
| 行数 | ≥100行 | 接近历史平均（121行）|
| 深度分析 | ≥3个章节 | 技术探讨、市场分析 |
| Hacker News | ≥3条热门讨论 | 高分帖子（>100分）|
| 数据源标注 | 必须标注 | SearXNG聚合说明 |

### 每日检查清单

```
□ 新闻条数是否达标（10条）
□ 每条新闻是否有完整描述
□ 深度分析章节是否完整
□ Hacker News讨论是否添加
□ README.md是否更新（完整摘要）
□ 文件路径是否正确（daily_report_file/）
□ GitHub推送是否成功
□ 钉钉通知是否发送
```

---

*本文档由噬金虫自动生成 🐛*