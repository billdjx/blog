---
title: Hermes Agent 实战：OA 审批流程自动化
date: 2026-07-09 09:34:16
tags:
  - Hermes Agent
  - OA
  - 审批流
  - 自动化
  - 实战
categories:
  - 实战教程
---

## 前言

上一篇介绍了 Hermes Agent 的基本配置，这篇用**真实业务场景**演示如何用它来自动化 OA 审批相关工作流。

以一个典型的**客户清退流程**（customerExit）为例，包含：

- 提单页数据填充
- 审批节点状态跟踪
- 待办任务批量处理
- 异常流程自动预警

---

## 场景一：自动填充提单页

### 痛点

每次提单都需要手动选择客户、填写合同信息、上传附件，重复性高且容易漏填。

### 配置上下文文件

在项目根目录创建 `.hermes_context.md`：

```markdown
# OA 项目上下文

## 项目路径

/Users/bill/company/lesoon-oa-web

## 技术栈

- 框架: Umi.js 4 + React 18
- UI: Ant Design 4
- 微前端: qiankun
- 请求库: ls-pro-common

## 关键页面路由

| 页面       | 路由                        | 组件                                       |
| ---------- | --------------------------- | ------------------------------------------ |
| 客户清退   | /customerExit               | pages/customerExit/index.tsx               |
| 清退执行   | /customerClearanceExecution | pages/customerClearanceExecution/index.tsx |
| 实体仓变更 | /storeInfoChange            | pages/storeInfoChange/index.tsx            |
| 报销助手   | /reimbursementAssistant     | pages/reimbursementAssistant/index.tsx     |

## API 前缀

- /petrel/ — 业务 API
- /lesoon/ — 流程 API
```

### 下达指令

配置好后，直接对 Hermes Agent 说：

```
帮我提单客户清退流程，客户是"XX科技有限公司"，
清退原因是"业务调整"，合同编号是"HT-2026-001"
```

Hermes Agent 会自动：

1. 读取 FormSections 组件的表单字段定义
2. 定位到对应的输入框和下拉选择器
3. 使用 Playwright 驱动浏览器自动填写
4. 提交表单并返回流程 ID

---

## 场景二：审批流状态看板

### 痛点

每天要打开多个页面查看待办流程走到哪一步了，效率低。

### 创建定时技能

让 Hermes Agent 在项目目录下自动生成一个审批流跟踪技能：

```bash
hermes "创建一个审批流跟踪技能：
  每天早上 9 点和下午 5 点，
  自动查询我的待办列表，
  汇总每个流程的状态和当前节点，
  发送到我的 Telegram"
```

Hermes Agent 会自动生成以下技能文件：

````markdown
# SKILL.md

## Name

oa-approval-tracker

## Description

定时跟踪 OA 审批流程状态并推送汇总报告。

## API Endpoints

- GET /petrel/flow/todoList — 查询待办列表
- GET /petrel/flow/detail?flowId={id} — 查询流程详情
- GET /petrel/flow/nodeStatus?flowId={id} — 查询当前节点

## Cron Schedule

```yaml
# 自动注册的定时任务
- schedule: "0 9,17 * * 1-5"
  task: "查询待办并推送摘要"
```
````

## 输出格式

```markdown
## 📋 待办流程汇总 (2026-07-07 09:00)

| 流程名称           | 发起人 | 当前节点 | 等待时间 |
| ------------------ | ------ | -------- | -------- |
| 客户清退-XX 科技   | 张三   | 财务审核 | 2 天     |
| 实体仓变更-YY 仓库 | 李四   | 法务审批 | 5 天     |
| 报销单-ZS202607    | 王五   | 部门经理 | 1 天     |

⚠️ 超时提醒：

- 实体仓变更 已等待 5 天，建议催办
```

```

### 查看结果

配置完成后，每次定时任务执行都会收到类似这样的推送：

```

📋 待办流程汇总 (2026-07-07 09:00)

| 流程名称           | 发起人 | 当前节点 | 等待时间 |
| ------------------ | ------ | -------- | -------- |
| 客户清退-XX 科技   | 张三   | 财务审核 | 2 天     |
| 实体仓变更-YY 仓库 | 李四   | 法务审批 | 5 天 ⚠️  |

共 2 个待办，1 个超时

````

---

## 场景三：批量审批与自动处理

### 痛点
某些标准化流程（如低风险清退）每次都要手动点审批，重复劳动。

### 配置自动审批规则

在 `.hermes_context.md` 或单独创建审批规则文件 `~/.hermes/approval-rules.yaml`：

```yaml
# 自动审批规则
rules:
  - name: "低风险客户清退"
    condition:
      page: customerExit
      field:
        - key: riskLevel
          value: low
        - key: amount
          max: 100000
    action: auto_approve
    reason: "低风险小额清退，自动通过"

  - name: "标准报销单"
    condition:
      page: reimbursementAssistant
      field:
        - key: totalAmount
          max: 5000
        - key: category
          value: travel
    action: auto_approve
    reason: "差旅费标准额度内，自动通过"

  - name: "超时催办"
    condition:
      currentNodeWaitHours: 72
    action: send_reminder
    channel: telegram
    message: "流程 {{flowName}} 已等待超过 72 小时，请及时处理"
````

### 自动审批流程

```bash
hermes "帮我处理今天的待办，
  符合自动审批规则的直接通过，
  不符合的列出让我手动处理"
```

Hermes Agent 会：

1. 查询待办列表
2. 逐项匹配自动审批规则
3. 符合规则的调用审批接口自动通过
4. 不符合的整理成表格让你手动处理

---

## 场景四：异常流程预警

### 痛点

流程卡在某节点好几天没人处理，等发现时已经延误了业务。

### 自动预警技能

直接在对话中让 Hermes Agent 建立预警：

```
帮我创建一个流程预警技能：
  每 2 小时检查一次所有进行中的流程，
  如果某个节点超过 3 天没有动作就发出预警，
  超过 5 天升级为紧急预警通知到钉钉群。
  预警信息包括：流程名称、当前节点、负责人、已等待时间
```

生成后的技能伪代码逻辑：

```python
# 自动生成的预警检查逻辑
def check_overdue_flows():
    # 1. 查询所有进行中的流程
    active_flows = api.get("/petrel/flow/activeList")

    for flow in active_flows:
        # 2. 获取当前节点
        node = api.get(f"/petrel/flow/nodeStatus?flowId={flow['id']}")
        wait_hours = calculate_wait_hours(node['startTime'])

        # 3. 分级预警
        if wait_hours > 120:  # 5天
            send_alert(
                level="critical",
                title=f"🚨 紧急: {flow['name']}",
                message=f"已等待 {wait_hours} 小时，负责人: {node['assignee']}",
                channel="dingtalk_group"
            )
        elif wait_hours > 72:  # 3天
            send_alert(
                level="warning",
                title=f"⚠️ 超时: {flow['name']}",
                channel="telegram"
            )
```

---

## 场景五：智能催办

### 痛点

遇到紧急流程卡在某审批人手上，需要催办但不好意思太频繁。

### 自然语言催办

```
帮我催办一下"XX科技客户清退"这个流程，
当前在财务审核节点，
催办语气不要太生硬，礼貌一些
```

Hermes Agent 会：

```
已找到流程 "XX科技客户清退" (ID: FL-2026-0715)
当前节点: 财务审核
负责人: 李四 (li.si@company.com)

正在生成催办消息...

---

李四你好，

打扰了，想跟进一下 "XX科技客户清退" 流程的审批进度。
目前该流程在财务审核节点已等待 3 天，
客户那边比较着急，方便的话麻烦抽空看下？

谢谢！
---

是否发送？
```

---

## 场景六：流程数据分析

### 痛点

想统计一周内的流程处理效率，手动翻数据非常痛苦。

### 对话式查询

```
帮我统计这周的 OA 流程数据：
  1. 各类流程的发起数量
  2. 平均审批耗时
  3. 超时流程占比
  4. 各部门处理效率排名
  用表格展示结果
```

Hermes Agent 返回：

```
📊 本周 OA 流程数据统计 (2026-06-30 ~ 2026-07-06)

## 1. 流程发起数量
| 流程类型 | 发起数 |
|---------|-------|
| 客户清退 | 23 |
| 实体仓变更 | 15 |
| 报销单 | 42 |
| 合计 | 80 |

## 2. 平均审批耗时
| 流程类型 | 平均耗时 |
|---------|---------|
| 客户清退 | 1.8天 |
| 实体仓变更 | 3.2天 |
| 报销单 | 0.5天 |

## 3. 超时流程占比
- 超时流程: 12 个 (15%)
- 其中超 5 天: 3 个

## 4. 部门处理效率
| 部门 | 平均处理时长 | 排名 |
|-----|------------|------|
| 财务部 | 4小时 | 🥇 |
| 业务部 | 8小时 | 🥈 |
| 法务部 | 2天 | 🥉 |
```

---

## 场景七：生成代码片段

Hermes Agent 还能直接帮你写 OA 相关的代码。

### 示例：审批按钮组件

```
帮我写一个审批流操作按钮组件，
包含：通过、驳回、转办三个按钮，
用 Ant Design 的 Button，
按钮的显示逻辑根据 nodeActions 配置控制
```

Hermes Agent 会根据上下文直接生成：

```tsx
import { Button, Space, Modal } from "antd";
import {
  CheckCircleFilled,
  CloseCircleFilled,
  SwapOutlined,
} from "@ant-design/icons";

interface ActionButtonsProps {
  nodeActions?: string[]; // 当前节点可用操作
  onApprove?: () => void;
  onReject?: () => void;
  onTransfer?: () => void;
}

const ActionButtons = ({
  nodeActions = [],
  onApprove,
  onReject,
  onTransfer,
}: ActionButtonsProps) => {
  const actions = new Set(nodeActions);

  return (
    <Space>
      {actions.has("approve") && (
        <Button type="primary" icon={<CheckCircleFilled />} onClick={onApprove}>
          通过
        </Button>
      )}
      {actions.has("reject") && (
        <Button danger icon={<CloseCircleFilled />} onClick={onReject}>
          驳回
        </Button>
      )}
      {actions.has("transfer") && (
        <Button icon={<SwapOutlined />} onClick={onTransfer}>
          转办
        </Button>
      )}
    </Space>
  );
};
```

---

## 配置总结

将以上场景整合到 Hermes Agent 的完整配置：

```bash
# 1. 创建 OA 项目上下文
cat > ~/company/lesoon-oa-web/.hermes_context.md << 'EOF'
# OA 项目上下文

## 项目路径
/Users/bill/company/lesoon-oa-web

## API Endpoints
- 待办查询: GET /petrel/flow/todoList
- 流程详情: GET /petrel/flow/detail?flowId={id}
- 提交审批: POST /petrel/flow/approve
- 驳回: POST /petrel/flow/reject
- 转办: POST /petrel/flow/transfer
EOF

# 2. 设置定时跟踪
hermes cron add \
  --schedule "0 9,17 * * 1-5" \
  --task "查询 OA 待办并推送汇总" \
  --channel telegram

# 3. 设置超时预警
hermes cron add \
  --schedule "0 */2 * * *" \
  --task "检查超时流程并预警" \
  --channel dingtalk

# 4. 创建审批规则
cat > ~/.hermes/approval-rules.yaml << 'EOF'
rules:
  - name: "低风险自动审批"
    condition:
      page: customerExit
      field:
        - key: riskLevel
          value: low
    action: auto_approve
EOF
```

---

## 总结

通过 Hermes Agent，OA 审批流程的日常操作可以大幅自动化：

| 场景     | 之前          | 之后         |
| -------- | ------------- | ------------ |
| 提单填写 | 手动 5 分钟   | 一句话 30 秒 |
| 待办跟踪 | 每天查 3-5 次 | 定时自动推送 |
| 批量审批 | 逐条手动点    | 规则自动处理 |
| 超时监控 | 遗漏风险高    | 自动分级预警 |
| 数据统计 | 翻表半小时    | 对话即查     |

Hermes Agent 的优势在于**持久记忆**——它记得你的项目结构、API 地址、审批规则，不需要每次重复配置。用得越久，它越懂你的业务。
