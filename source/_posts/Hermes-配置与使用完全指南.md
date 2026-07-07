---
title: Hermes 配置与使用完全指南
date: 2026-07-07 10:29:49
tags:
  - Hermes
  - React Native
  - JavaScript 引擎
  - 性能优化
categories:
  - 技术笔记
---

## 前言

Hermes 是 Meta 为 React Native 量身打造的 JavaScript 引擎，核心目标是**提升启动速度**、**减少内存占用**、**缩小包体积**。自 RN 0.64 起成为可选引擎，0.70+ 开始默认开启。

---

## Hermes  vs  JSC 对比

| 特性 | Hermes | JSC（默认） |
|------|--------|-------------|
| 启动速度 | 快（预编译字节码） | 中等（需解析+编译） |
| 包体积 | 小（约 -10MB） | 较大 |
| 内存占用 | 低 | 一般 |
| 调试支持 | Chrome DevTools | Flipper |
| ES6+ 支持 | 大部分 | 完整 |
| Inspector | Hermes CLI 工具 | Flipper |

---

## 一、启用 Hermes

### React Native 0.70+

从 0.70 开始 Hermes 是默认引擎，无需额外配置。如果想显式指定，打开 `android/app/build.gradle`：

```gradle
project.ext.react = [
    enableHermes: true  // true 启用，false 关闭
]
```

iOS 端在 `ios/Podfile` 中已默认启用：

```ruby
:hermes_enabled => true
```

### React Native 0.64 - 0.69

需要手动开启：

**Android** - `android/app/build.gradle`：

```gradle
project.ext.react = [
    entryFile: "index.js",
    enableHermes: true  // 改为 true
]
```

**iOS** - `ios/Podfile`：

```ruby
use_react_native!(
  :path => config[:reactNativePath],
  :hermes_enabled => true
)
```

然后重新安装 pods：

```bash
cd ios && pod install
```

---

## 二、验证 Hermes 是否生效

### 方法 1：控制台打印

在 App 中执行：

```jsx
import { Platform } from 'react-native';

console.log('JavaScript Engine:', HermesInternal ? 'Hermes' : 'JSC');
```

### 方法 2：检测全局对象

```jsx
const isHermes = () => !!global.HermesInternal;
```

如果返回 `true`，说明 Hermes 已成功启用。

---

## 三、Hermes 特有 API

启用 Hermes 后可以使用以下特有 API：

```jsx
// 获取引擎版本号
const version = HermesInternal.getEngineVersion();

// 获取运行时内存信息
const memoryInfo = HermesInternal.getRuntimeMemoryInfo();
// { gc: {...}, js: {totalBytes, usedBytes}, native: {totalBytes, usedBytes} }

// 获取运行时间统计
const timeInfo = HermesInternal.getTTI();
```

> 注意：`HermesInternal` 在生产模式下会被移除，需在 debug 下使用或做容错处理。

---

## 四、Hermes 的局限性

### 1. 不支持 eval()

```jsx
// ❌ Hermes 不支持
eval('console.log("hello")');

// ✅ 替代方案：使用 Function 构造函数
new Function('console.log("hello")')();
```

### 2. 不支持 Symbol 的部分功能

```jsx
// 以下在 Hermes 中不可用：
Symbol.for('key');       // 全局 Symbol 注册
Symbol.keyFor(sym);      // 反向查找
Symbol.prototype.description;  // description 属性
```

### 3. Proxy 受限

Hermes 中的 Proxy 有性能开销，高频使用场景需注意。

### 4. 不支持 Array.prototype.flat 的 Infinity 参数

某些较旧版本不支持，建议使用 lodash 的 `_.flattenDeep` 替代。

---

## 五、Hermes 调试

### Chrome DevTools

```bash
# 启动 metro
npx react-native start

# 在新的终端中连接 Hermes 调试
npx react-native log-ios     # iOS
npx react-native log-android  # Android
```

在 Chrome 中打开 `chrome://inspect`，找到 Hermes 调试目标即可。

### 使用 Flipper

Hermes 支持 Flipper 调试，可以查看布局、网络请求、JS 堆快照等。

### Hermes Profiler

```bash
# 生成采样分析
npx react-native profile-hermes
```

会在 `android/app/src/main/assets/` 下生成 `.cpuprofile` 文件，可在 Chrome DevTools 的 Performance 面板中加载分析。

---

## 六、Hermes 字节码

Hermes 会将 JS 代码预编译为字节码（.hbc 文件），这是其启动快的关键原因。

### 手动编译字节码

```bash
# 安装 hermes CLI
npm install -g hermes-engine

# 编译 JS 文件为字节码
hermes -emit-binary -out app.hbc index.js

# 反编译字节码为可读格式
hermes -dump-bytecode app.hbc > bytecode.txt
```

---

## 七、最佳实践

### 1. 内存泄漏检测

利用 `HermesInternal.getRuntimeMemoryInfo()` 定期检测内存增长：

```jsx
setInterval(() => {
  const info = HermesInternal.getRuntimeMemoryInfo();
  const usedMB = info.js.usedBytes / 1024 / 1024;
  if (usedMB > 100) {
    console.warn(`内存偏高: ${usedMB.toFixed(2)}MB`);
  }
}, 30000);
```

### 2. 启动耗时监控

```jsx
if (__DEV__) {
  const tti = HermesInternal.getTTI();
  console.log('TTI:', tti, 'ms');
}
```

### 3. 兼容性检查

建议在入口文件做兼容检测，提前发现异常：

```jsx
if (global.HermesInternal) {
  // 检查关键 API 是否存在
  const featureCheck = {
    Proxy: typeof Proxy !== 'undefined',
    Symbol: typeof Symbol !== 'undefined',
    flat: typeof [].flat === 'function'
  };
  console.log('Hermes feature check:', featureCheck);
}
```

---

## 八、常见问题

### Q1: 启用 Hermes 后动画卡顿？

Hermes 的 GC 策略与 JSC 不同，建议：
- 减少高频触发的 `setState`
- 使用 `useNativeDriver: true`
- 用 `requestAnimationFrame` 替代 `setTimeout`

### Q2: Hermes 包的体积反而变大？

检查 `android/app/build.gradle` 中是否配置了 `hermes` 包压缩，确保 release 构建使用 `hermes-release.aar`。

### Q3: Hermes 报 "SyntaxError: Unexpected token"

某些较新的 JS 语法 Hermes 暂不支持，检查 Babel 配置是否包含了必要的 preset/plugin。

```bash
npm install @react-native/babel-preset --save-dev
```

并在 `babel.config.js` 中使用：

```js
module.exports = {
  presets: ['module:@react-native/babel-preset']
};
```

---

## 总结

Hermes 是目前 React Native 应用的**默认 JS 引擎**，在启动速度、内存和包体积上都有明显优势。虽然有一些语法限制，但对于大多数业务场景影响不大。如果你的项目还在用 JSC，升级到 Hermes 是一个低投入高收益的优化手段。
