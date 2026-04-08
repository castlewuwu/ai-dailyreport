# AI日报技术实现说明

> 文档生成日期：2026年4月8日

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

### 1. 数据源

| 来源 | 方式 | 用途 |
|------|------|------|
| SearXNG | 本地实例 (localhost:8080) | 主要搜索引擎聚合 |
| web_fetch | HTTP抓取 | 深度获取文章内容 |
| 知乎/Reddit | SearXNG索引 | 社区讨论热点 |

### 2. SearXNG配置

```bash
# 搜索命令
SEARXNG_URL=http://localhost:8080 uv run searxng.py search "AI news GPT Claude" -n 15
```

### 3. 搜索策略

```
AI日报搜索关键词：
├── 大模型进展: "GPT Claude Gemini model release benchmark"
├── AI工具更新: "AI tool coding assistant agent"
├── 行业动态:   "OpenAI Anthropic funding acquisition"
├── 技术突破:   "AI research paper arxiv breakthrough"
└── 国产AI:     "DeepSeek Qwen Kimi 中国AI"
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

### 文件存储

```
~/.openclaw/workspace/github/ai-dailyreport/
├── 2026-04-08.md    # 日报文件
├── 2026-04-07.md
├── ...
└── README.md        # 目录索引
```

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

## 改进方向

1. **新闻源扩展** - 添加RSS订阅、Twitter API
2. **自动分类** - 按主题自动打标签
3. **多渠道推送** - 支持Telegram、邮件等
4. **历史分析** - 统计高频话题、趋势图表

---

*本文档由噬金虫自动生成 🐛*