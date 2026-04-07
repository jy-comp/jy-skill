# mode=plan：分析点位 + 规划代码骨架

## P1：读取点位配置

从沙盒读取当前点位配置，提取所有点位的 `key` 和类型：

```
GET /sandbox/sessions/:id/files?path=point.config.local-remote.json
```

或从对话上下文中获取已确认的点位列表。

## P2：各点位类型代码骨架规范

### 通用初始化模板（所有点位必须包含）

```typescript
import React from 'react';
import { createRoot } from 'react-dom/client';

export default async function main() {
  await window.JSSDK.shared.setSharedModules({ React, ReactDOM });
  const container = document.createElement('div');
  document.body.appendChild(container);
  createRoot(container).render(<App />);
}
```

### board（嵌入页面）

```typescript
// 全页面 React 应用，可调用 JSSDK 获取空间/工作项数据
const App = () => {
  const [data, setData] = React.useState(null);

  React.useEffect(() => {
    // 获取当前空间信息
    window.JSSDK.shared.getProjectInfo().then(setData);
  }, []);

  return <div className="board-container">{/* 业务内容 */}</div>;
};
```

### dashboard（详情 Tab）

```typescript
// 嵌入工作项详情页，接收 workItemId
const App = () => {
  const [workItem, setWorkItem] = React.useState(null);

  React.useEffect(() => {
    // 获取当前工作项详情
    window.JSSDK.shared.getWorkItemDetail().then(setWorkItem);
  }, []);

  return <div className="dashboard-container">{/* 业务内容 */}</div>;
};
```

### button（按钮，script 模式）

```typescript
// script 模式：执行操作后调用 trigger 通知
export default async function main() {
  // 获取当前上下文
  const context = await window.JSSDK.shared.getContext();

  // 执行业务逻辑...

  // 操作完成通知
  await window.JSSDK.shared.trigger({ success: true, message: '操作完成' });
}
```

### control（表单控件）

```typescript
// 实现受控输入接口
const App = () => {
  const [value, setValue] = React.useState('');

  const handleChange = (newValue: string) => {
    setValue(newValue);
    // 通知宿主值变更
    window.JSSDK.shared.onChange(newValue);
  };

  return <input value={value} onChange={e => handleChange(e.target.value)} />;
};
```

### config（插件配置页）

```typescript
// 插件配置界面，保存配置到插件存储
const App = () => {
  const handleSave = async (config: Record<string, any>) => {
    await window.JSSDK.shared.saveConfig(config);
  };

  return <div>{/* 配置表单 */}</div>;
};
```

### intercept（拦截器）

```typescript
// 服务端 webhook，不生成前端代码
// 生成服务端处理函数模板（Node.js/TypeScript）
export async function handler(req: Request, res: Response) {
  const { event_type, work_item } = req.body;

  // 拦截逻辑...

  // 返回是否允许操作
  res.json({ allow: true });
}
```

## P3：规划文件清单

根据点位列表，输出待生成的文件清单：

```
待生成文件：
  src/features/board/jira_board_main/index.tsx   （board 点位）
  src/features/dashboard/detail_tab/index.tsx    （dashboard 点位）
  plugin.config.json（更新 resources 字段）
```

向用户确认后，进入 apply 阶段。
