---
title: Hermes Agent 配置与使用完全指南
date: 2026-07-07 10:29:49
tags:
  - Hermes Agent
  - AI 智能体
  - Nous Research
  - 开源
categories:
  - 技术笔记
---

## 前言

Hermes Agent 是由 **Nous Research** 于 2026 年 2 月发布的开源自主 AI 智能体框架（MIT 协议）。它不是 IDE 代码补全工具，也不是套壳聊天机器人——它是一个**住在你服务器上、拥有持久记忆、能自主创建技能、越用越聪明的智能体**。

核心特性：
- **持久记忆** — 跨会话记住你的偏好、项目和环境
- **自动技能创建** — 解决问题后自动生成可复用的技能文档
- **多平台接入** — Telegram、Discord、Slack、微信、飞书等 20+ 平台
- **100% 自托管** — 所有数据存储在你本地，零遥测零追踪

---

## 一、快速安装

### macOS / Linux

一行命令安装，无需任何前置依赖：

```bash
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash
```

安装程序会自动完成：安装 `uv`、Python 3.11、克隆仓库并完成配置。

### 验证安装

```bash
hermes --version
```

### 更新

```bash
hermes update
```

---

## 二、基础配置

### 交互式配置向导

首次使用建议运行配置向导：

```bash
hermes setup
```

向导会逐步引导你完成：
1. 选择 LLM 提供商
2. 配置 API Key
3. 设置智能体名称和个性

### 选择模型提供商

Hermes Agent 支持多种模型后端：

| 提供商 | 配置方式 | 适用场景 |
|--------|---------|---------|
| Nous Portal | OAuth 登录 | 官方推荐，原生集成 |
| OpenRouter | API Key | 访问 200+ 模型 |
| OpenAI | API Key | 标准 OpenAI 兼容 |
| 自定义端点 | 任意 OpenAI 兼容 API | 自建模型或国内大模型 |
| 本地 vLLM | 本地地址 | 完全离线运行 |

```bash
# 切换模型提供商
hermes model
```

### 配置文件位置

所有配置存储在 `~/.hermes/` 目录下：

```
~/.hermes/
├── config.yaml        # 主配置文件
├── memory/            # 持久化记忆存储
├── skills/            # 自动生成的技能
└── gateway/           # 消息网关配置
```

### 核心配置项

```yaml
# ~/.hermes/config.yaml
agent:
  name: "Hermes"                # 智能体名称
  soul: "helpful and concise"   # 个性描述
  model:
    provider: openrouter        # 提供商
    model: nousresearch/hermes-3-pro  # 模型
    temperature: 0.7
  memory:
    enabled: true               # 开启持久记忆
    recall_threshold: 0.6       # 记忆召回阈值
  skills:
    auto_create: true           # 自动创建技能
    auto_improve: true          # 自动改进技能
```

---

## 三、CLI 使用入门

### 启动对话

```bash
# 直接开始对话
hermes

# 带上下文启动
hermes "帮我分析一下当前目录的项目结构"

# 指定工作目录
hermes --workdir /path/to/project
```

### 常用 CLI 命令

```bash
hermes setup          # 配置向导
hermes model          # 切换模型
hermes gateway        # 启动消息网关
hermes gateway setup  # 配置消息平台
hermes skill list     # 查看已创建的技能
hermes memory search  # 搜索记忆
hermes cron list      # 查看定时任务
hermes update         # 更新到最新版本
hermes logs           # 查看日志
```

### 上下文文件

在项目目录下创建 `.hermes_context.md`，智能体会自动读取：

```markdown
# .hermes_context.md

## 项目简介
这是一个基于 Taro 4 + React 18 的家庭点菜小程序。

## 技术栈
- 框架: Taro 4.2.0
- 语言: TypeScript 5
- UI: NutUI 3
- 状态管理: Zustand 5

## 编码规范
- 使用函数组件 + Hooks
- 样式使用 Less
- 状态管理优先使用 Zustand
```

---

## 四、多平台接入（消息网关）

Hermes Agent 支持通过单一网关连接 20+ 平台，可以在不同平台间无缝切换对话。

### 配置网关

```bash
# 启动交互式网关配置
hermes gateway setup
```

### 支持的平台

| 平台 | 状态 | 说明 |
|------|------|------|
| Telegram | ✅ 稳定 | 机器人 Token 接入 |
| Discord | ✅ 稳定 | Bot Token + Intents |
| Slack | ✅ 稳定 | App + OAuth |
| WhatsApp | ✅ 稳定 | WhatsApp Business API |
| Signal | ✅ 稳定 | Signal CLI |
| 钉钉 | ✅ 稳定 | 机器人 Webhook |
| 飞书 | ✅ 稳定 | 机器人 Webhook |
| 企业微信 | ✅ 稳定 | 机器人回调 |
| QQ 机器人 | ✅ 稳定 | QQ 官方 Bot |
| 微信 | ⚠️ 实验 | 第三方实现 |
| Matrix | ✅ 稳定 | 去中心化聊天 |
| Email | ✅ 稳定 | IMAP/SMTP |
| SMS | ✅ 稳定 | Twilio |

### 启动网关服务

```bash
# 前台运行
hermes gateway

# 安装为系统服务（后台驻留）
hermes gateway install
```

---

## 五、记忆系统

记忆是 Hermes Agent 最核心的能力——它跨会话记住你的信息，越用越懂你。

### 记忆类型

| 类型 | 说明 | 示例 |
|------|------|------|
| 用户偏好 | 你的习惯和偏好 | "用户喜欢 TypeScript 而不是 JavaScript" |
| 项目信息 | 项目相关上下文 | "这个项目使用 Taro 4 + NutUI 3" |
| 环境配置 | 开发环境信息 | "开发服务器在 192.168.1.100" |
| 决策记录 | 重要的技术决策 | "选择了 Zustand 而不是 Redux" |

### 记忆操作

```bash
# 搜索记忆
hermes memory search "项目技术栈"

# 查看所有记忆
hermes memory list

# 手动添加记忆
hermes memory add "用户 prefers TypeScript over JavaScript"

# 删除记忆
hermes memory delete <memory-id>
```

### 记忆配置

```yaml
memory:
  enabled: true
  # 基于 FTS5 的全文搜索
  search_backend: fts5
  # LLM 摘要增强召回
  llm_summary: true
  # 主动提示频率
  proactive_prompt_interval: 10
```

---

## 六、技能系统

当 Hermes Agent 解决了一个复杂问题，它会自动生成可复用的**技能文档**（SKILL.md），下次遇到类似问题直接调用。

### 技能生命周期

```
遇到新问题 → 解决问题 → 自动生成技能 → 下次复用 → 持续改进
```

### 技能管理

```bash
# 列出所有技能
hermes skill list

# 查看技能详情
hermes skill show <skill-name>

# 搜索技能
hermes skill search "react native"

# 禁用技能
hermes skill disable <skill-name>

# 从社区安装技能
hermes skill install <skill-url>
```

### 技能格式示例

```markdown
# SKILL.md

## Name
Taro 小程序项目初始化

## Description
创建基于 Taro 4 + TypeScript 的小程序项目，并配置 NutUI 和 Zustand。

## Steps
1. 使用 `npm create taro-app@latest` 创建项目
2. 选择 TypeScript 模板
3. 安装 NutUI: `npm install @nutui/nutui-react-taro`
4. 安装 Zustand: `npm install zustand`
5. 配置项目别名和编译选项

## Tags
taro, miniprogram, react, typescript

## Compatibility
- Platform: macOS, Linux
- Requirements: Node.js 18+
```

---

## 七、定时任务

内置 cron 调度器，支持无人值守自动执行。

### 配置定时任务

```bash
# 添加定时任务
hermes cron add --schedule "0 9 * * 1" --task "生成本周项目报告"

# 列出任务
hermes cron list

# 删除任务
hermes cron remove <task-id>
```

### 实用示例

```bash
# 每天早上 9 点发送日报
hermes cron add \
  --schedule "0 9 * * *" \
  --task "检查 GitHub Issues 更新并生成摘要" \
  --channel telegram

# 每周五下午 6 点周报
hermes cron add \
  --schedule "0 18 * * 5" \
  --task "汇总本周代码提交和项目进展" \
  --channel email
```

---

## 八、子智能体与并行任务

Hermes Agent 支持派生子智能体并行处理任务，每个子智能体有独立的对话和终端。

```bash
# 在对话中派生子智能体
hermes "请同时帮我做三件事：
  1. 审查这个 PR 的代码
  2. 检查服务器日志中的错误
  3. 更新项目依赖版本
  每个任务用独立的子智能体并行执行"
```

子智能体的特点：
- **隔离环境** — 每个子智能体有独立的上下文
- **并行执行** — 多任务同时处理，互不干扰
- **结果汇总** — 自动合并各子任务结果

---

## 九、安全配置

### 命令审批

敏感操作需要审批：

```yaml
# ~/.hermes/config.yaml
security:
  # 需要审批的命令模式
  require_approval:
    - "rm *"
    - "sudo *"
    - "docker * --force *"
  # 自动批准的安全命令
  auto_approve:
    - "git status"
    - "npm install *"
    - "ls *"
```

### 容器隔离

```yaml
# 启用 Docker 沙箱
terminal:
  backend: docker
  docker:
    image: hermes-sandbox:latest
    read_only_root: true
    drop_capabilities: true
    pids_limit: 100
```

---

## 十、SOUL.md — 自定义智能体个性

通过 `~/.hermes/soul.md` 定义智能体的行为风格：

```markdown
# SOUL.md

你是一个资深前端工程师助手。

## 沟通风格
- 回答简洁直接，优先给出可运行的代码
- 解释问题时先说结论，再展开细节
- 不确定时说"我不确定"，而不是编造答案

## 技术偏好
- 推荐 TypeScript 而不是 JavaScript
- 优先使用函数组件和 Hooks
- 状态管理推荐 Zustand

## 行为准则
- 修改文件前先向用户展示 diff
- 执行破坏性操作前必须确认
- 主动提示潜在的风险和注意事项
```

---

## 十一、最佳实践

### 1. 项目上下文文件

在每个项目根目录创建 `.hermes_context.md`，让智能体自动了解项目：

```markdown
# .hermes_context.md

## Project: family-meal-app
## Stack: Taro 4 + TypeScript + NutUI 3 + Zustand 5
## Build: npm run build:weapp
## Dev: npm run dev:weapp
```

### 2. 定期回顾记忆

```bash
# 查看智能体对自己的认知
hermes memory search "user"

# 查看项目相关记忆
hermes memory search "project"
```

### 3. 技能管理

定期清理不再需要的技能，保持技能库高质量：

```bash
hermes skill list --sort frequency
hermes skill disable outdated-skill
```

### 4. 多终端协同

在 Telegram 上发起任务，在 CLI 中继续：

```
Telegram: "帮我检查一下生产服务器状态"
→ 智能体开始执行
→ 回到电脑前，在 CLI 中输入 hermes 继续同一对话
→ "把刚才的检查结果整理成报告发我邮箱"
```

---

## 十二、常见问题

### Q1: Hermes Agent 和 Claude/ChatGPT 有什么区别？

Hermes Agent 是**自主智能体**，不是聊天机器人。区别在于：
- **持久记忆** — 跨会话记住上下文，不用每次重新解释
- **自动技能** — 解决问题的经验会沉淀为可复用技能
- **工具调用** — 可以直接操作你的终端、浏览器、文件系统
- **自主执行** — 支持定时任务、无人值守

### Q2: 数据安全吗？

所有数据存储在本地 `~/.hermes/` 目录，零遥测、零追踪、零数据收集。完全开源 MIT 协议，每行代码都可审计。

### Q3: 可以用国内大模型吗？

可以。配置自定义 OpenAI 兼容端点指向国内模型 API 即可：

```bash
hermes model set custom \
  --base-url https://api.xxx.com/v1 \
  --api-key your-key \
  --model deepseek-chat
```

### Q4: 提示 quota 耗尽或 API 不可用？

Hermes Agent 支持多模型自动切换，可以配置 fallback 模型：

```yaml
model:
  primary:
    provider: openrouter
    model: nousresearch/hermes-3-pro
  fallback:
    provider: openai
    model: gpt-4o-mini
```

### Q5: 如何迁移到新机器？

直接复制 `~/.hermes/` 目录到新机器即可，记忆和技能全部保留。

---

## 总结

Hermes Agent 是一个真正「越用越聪明」的 AI 智能体。与传统 AI 工具不同，它的**持久记忆**和**自动技能创建**让它能随着使用时间的增长不断进化。

| 场景 | 推荐用法 |
|------|---------|
| 日常开发 | 项目上下文 + CLI 交互 |
| 团队协作 | 接入 Telegram/Discord/飞书 |
| 自动化运维 | 定时任务 + 命令审批 |
| 研究实验 | 批处理 + 轨迹导出 + RL 训练 |

快速开始：

```bash
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash
hermes setup
hermes
```

> 官方文档: https://hermes-agent.nousresearch.com/docs/zh-Hans/
> GitHub: https://github.com/NousResearch/hermes-agent
