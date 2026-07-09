---
title: OA 审批流程页面标准化搭建指南
date: 2026-07-09 17:44:17
tags:
  - OA
  - 审批流
  - React
  - 前端架构
  - 最佳实践
categories:
  - 技术笔记
---

## 前言

在 OA 系统中，审批流程页面是最常见的业务场景之一。这类页面通常包含：申请表单、审批流展示、审批操作弹窗等功能。

经过多个流程页面的迭代，我们沉淀了一套**标准化搭建模式**，能将一个新流程页面的开发时间从 2-3 天压缩到 **半天以内**。

---

## 一、目录结构

每个流程页面遵循统一的目录规范：

```
src/pages/{processName}/
├── services/
│   └── index.ts          # API 服务层
├── components/
│   ├── FormSections.tsx   # 表单区域组件（申请信息/变更信息/附件）
│   └── Dialogs.tsx        # 审批弹窗（通过/驳回/转办）
├── constants.ts           # 节点操作按钮配置
└── index.tsx              # 主页面（组装布局+状态管理）
```

**设计理念**：按职责分离，每一层只关心一件事。

---

## 二、核心模式拆解

### 2.1 主页面 index.tsx — 状态编排中心

主页面扮演「**胶水层**」角色，负责串联各个模块：

```tsx
import OADetailLayout, { Content, Main, Sidebar } from '@/components/oaDetail';
import ApprovalFlow from '@/components/ApprovalFlow';
import ProForm from 'ls-pro-form';
import { nodeActions } from './constants';
import Services from './services';
import Dialogs from './components/Dialogs';
import { useCurrentNode } from '@/components/useFun/useCurrentNode';
import { useShouldForceShowAll } from '@/components/useFun/useShouldForceShowAll';

const services = new Services();

export default () => {
  const isMobile = useIsMobile();
  const dialogsRef = useRef<DialogsHandle>(null);
  const approvalFlowRef = useRef<ApprovalFlowHandle>(null);
  const formRef = useRef<any>(null);
  const [masterObject, setMasterObject] = useState<any>({
    billNo: getUrlQuery('billNo') || '',
  });

  // 当前审批节点信息
  const { show, taskName, taskDefKey, loadCurrentNode } = useCurrentNode();

  // 是否强制显示所有区域
  const shouldForceShowAll = useShouldForceShowAll(masterObject);

  // isViewMode 判断（核心逻辑之一）
  const isViewMode = useMemo(() => {
    if (!masterObject?.billNo) return false;       // 无单号 → 新建
    if (shouldForceShowAll) return true;             // 已完结 → 查看
    if (taskDefKey === 'Activity_submit' && show) return false; // 提交节点 → 编辑
    return true;                                     // 其他 → 查看
  }, [masterObject, taskDefKey, show, shouldForceShowAll]);

  // ...
};
```

#### isViewMode 判断逻辑图解

```
是否有 billNo？
  ├── 否 → 新建模式（编辑）
  └── 是 → 是否已完结/已撤回？
       ├── 是 → 强制查看模式
       └── 否 → 当前节点是否为提交节点 + show=true？
            ├── 是 → 编辑模式（发起人可以修改）
            └── 否 → 查看模式（审批人只能查看）
```

这个判断覆盖了流程页面的**全部状态**，后续所有表单的读写控制都依赖它。

---

### 2.2 services/index.ts — API 服务层

统一封装所有后端接口调用：

```tsx
import { BaseService, httpGet, httpPost } from 'ls-pro-common';

export default class CustomerExitService extends BaseService {
  api = {
    load: '/lesoon-oa-api/customerExit/page',
    edit: '/lesoon-oa-api/customerExit/update?_loading=加载中',
  };

  // 查询详情
  getDetail = (params: any) => {
    return httpGet('/lesoon-oa-api/customerExit/page?_loading=加载中', params);
  };

  // 提交/审批
  getSubmit = (params: any) => {
    return httpPost('/lesoon-oa-api/customerExit/saveAndSubmit?_loading=加载中', params);
  };

  // 保存草稿
  save = (params: any) => {
    return httpPost('/lesoon-oa-api/customerExit/save', params);
  };
}
```

> 注意：`_loading=加载中` 是 ls-pro-common 的约定写法，会在请求期间自动显示 loading 提示。

---

### 2.3 constants.ts — 节点操作配置

这是流程页面**按钮显示逻辑的核心**，用纯数据驱动：

```tsx
/** 节点 Activity ID → 操作按钮映射 */
export const nodeActions: Record<string, { key: string; label: string }[]> = {
  // 提交节点 — 无操作按钮
  Activity_customerExit_submit: [],

  // 业务审批节点
  Activity_customerExit_bizAudit: [
    { key: 'approve', label: '审批' },
    { key: 'reject', label: '驳回' },
    { key: 'transfer', label: '转办' },
  ],

  // 财务审核节点 — 不可转办
  Activity_customerExit_financeAudit: [
    { key: 'approve', label: '审批' },
    { key: 'reject', label: '驳回' },
  ],

  // 法务审核节点 — 只有通过
  Activity_customerExit_legalAudit: [
    { key: 'approve', label: '审批' },
  ],
};
```

**关键点**：`nodeActions` 是一个**只读常量**，不要在运行时修改它。按钮的显示/隐藏通过 `show` 和 `isViewMode` 等状态控制，而不是动态增删 `nodeActions`。

---

### 2.4 渲染按钮的逻辑

在 OADetailLayout 的 `renderButton` 中组装按钮：

```tsx
renderButton={(btns: any) => {
  const actions = nodeActions[taskDefKey] || [];
  return [
    // 审批模式且 show=true 时显示审批按钮
    ...(show && isViewMode
      ? actions.map((act) => (
          <Button
            key={act.key}
            shape="round"
            type={act.key === 'approve' ? 'primary' : 'default'}
            danger={act.key === 'reject'}
            onClick={() => handleNodeAction(act.key)}
          >
            {act.label}
          </Button>
        ))
      : []),

    // 取消按钮（常驻）
    <CancelButton key="cancel"
      masterObject={masterObject}
      afterSave={async () => {
        await update();
        loadNodeAndRefreshFlow(masterObject?.billNo);
      }}
    />,

    // 保存/提交按钮（由 OADetailLayout 根据 isEdit 控制）
    ...btns,
  ];
}}
```

**这里的条件判断**：
- `show` — useCurrentNode 返回的布尔值，当前用户是否是此节点的处理人
- `isViewMode` — 查看模式才显示审批按钮（编辑模式下显示的是保存/提交）
- `nodeActions[taskDefKey]` — 根据当前节点 ID 获取对应操作

---

### 2.5 FormSections.tsx — 表单区域组件

表单区域按业务语义拆分为多个独立组件，通过 `isViewMode` 切换查看/编辑模式：

```tsx
// ==================== 申请信息 ====================
export const ApplicationInfo = ({ isViewMode, masterObject }: any) => (
  <FormSection title="申请信息" icon={<UserDeleteOutlined />}>
    {isViewMode ? (
      <ProDescriptions column={{ xs: 1, sm: 1, md: 2 }}
        dataSource={masterObject}
        columns={[
          { dataIndex: 'customerName', title: '客户名称' },
          { dataIndex: 'reason', title: '清退原因' },
          { dataIndex: 'contractNo', title: '合同编号' },
        ]}
      />
    ) : (
      <>
        <ProFormSelect name="customerName" label="客户名称" .../>
        <ProFormTextArea name="reason" label="清退原因" .../>
        <ProFormText name="contractNo" label="合同编号" .../>
      </>
    )}
  </FormSection>
);
```

利用 `ProDescriptions`（查看）和 `ProForm*`（编辑）的声明式字段配置，表单区域的代码非常简洁且易于维护。

---

### 2.6 Dialogs.tsx — 审批弹窗

通过式、驳回、转办三个操作共用一个弹窗组件，用 `type` 区分：

```tsx
const Dialogs = forwardRef<DialogsHandle, DialogsProps>(
  ({ loadCurrentNode, masterObject }, ref) => {
    const [visible, setVisible] = useState(false);
    const [type, setType] = useState<string>('');

    useImperativeHandle(ref, () => ({
      openApprove: (masterObject: any) => {
        setType('approve');
        setVisible(true);
        formRef.current?.setFieldsValue(masterObject);
      },
      openReject: (masterObject: any) => {
        setType('reject');
        setVisible(true);
      },
      openTransfer: (masterObject: any) => {
        setType('transfer');
        setVisible(true);
      },
    }));

    const handleOk = async () => {
      const mstVal = await formRef.current?.validateFieldsReturnFormatValue();
      if (!mstVal) return;
      const params = { ...masterObject, ...mstVal, ifSubmitOrApprove: 0 };
      const res = await services.getSubmit(params);
      if (res?.flag?.retCode === '0') {
        showSuccess('操作成功');
        setVisible(false);
        loadCurrentNode(masterObject?.billNo);
      }
    };

    return (
      <Modal title={...} open={visible} onOk={handleOk} onCancel={...}>
        <ProForm layout="vertical" submitter={false} formRef={formRef}>
          {type === 'transfer' && (
            <ProFormText name="transferUser" label="转办人" rules={[{ required: true }]} />
          )}
          <ProFormTextArea name="reason" label={...} rules={[{ required: true }]} />
        </ProForm>
      </Modal>
    );
  },
);
```

**设计亮点**：
- 通过 `forwardRef + useImperativeHandle` 暴露方法，父组件直接调用
- 操作成功后自动刷新审批流节点状态
- 弹窗 `destroyOnClose` 避免表单残留

---

## 三、路由配置

### PC 端 (.umirc.ts)

```tsx
{
  path: '/customerExit',
  component: '@/pages/customerExit',
  title: '客户清退流程',
},
```

### 移动端 (router.ts)

```tsx
// 客户清退
customerExit: {
  url: '/lesoon-oa-web/customerExit',
  mobileAllowedTaskKeys: [
    'Activity_customerExit_submit',
    'Activity_customerExit_bizAudit',
    'Activity_customerExit_financeAudit',
    'Activity_customerExit_legalAudit',
  ],
},
```

---

## 四、新建流程页面的 Checklist

```
□ 1. 创建 pages/{processName} 目录及子目录
□ 2. 编写 services/index.ts（API 接口）
□ 3. 编写 constants.ts（节点映射 + 字典）
□ 4. 编写 components/FormSections.tsx（表单区域）
□ 5. 编写 components/Dialogs.tsx（审批弹窗）
□ 6. 编写 index.tsx（主页面组装）
□ 7. PC 端路由：修改 .umirc.ts
□ 8. 移动端路由：修改 router.ts
□ 9. 联调：根据后端接口调整字段映射
□ 10. 测试：新建/编辑/审批/驳回/转办全链路
```

---

## 五、模式总结

这套标准化模式的核心收益：

| 方面 | 传统方式 | 标准化模式 |
|------|---------|-----------|
| 开发效率 | 每个页面从零写起，2-3天 | 复制模板改字段，0.5天 |
| 可维护性 | 各页面按钮逻辑分散，bug 多 | 统一在 constants 中声明式配置 |
| 一致性 | 不同开发写的风格不同 | 统一目录/组件/Hook |
| 上手成本 | 新人需要理解整套业务逻辑 | 熟悉模板即可快速开发 |

核心依赖的自定义 Hook：

| Hook | 职责 |
|------|------|
| `useCurrentNode` | 查询当前审批节点信息（show/taskName/taskDefKey） |
| `useShouldForceShowAll` | 判断流程是否已完结，强制显示所有区域 |
| `useIsMobile` | 响应式检测

---

## 结语

OA 审批流程页面虽然涉及多个组件和状态，但经过抽象后核心模式非常清晰：

- **index.tsx** 负责状态编排
- **FormSections** 负责表单渲染
- **Dialogs** 负责审批操作
- **constants** 负责数据驱动按钮
- **OADetailLayout** 负责布局组合

这套模式已经在**客户清退、实体仓变更、仓储索赔确认、报销助手**等多个流程页面落地，证明是一个可复用的高效方案。
