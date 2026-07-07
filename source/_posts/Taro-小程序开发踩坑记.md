---
title: Taro 小程序开发踩坑记
date: 2026-07-07 09:59:49
tags:
  - Taro
  - 小程序
  - 踩坑
categories:
  - 技术笔记
---

## 前言

最近用 Taro 4 + React 18 开发了一款家庭点菜小程序，从项目初始化到上线踩了不少坑。这篇笔记记录下来，希望能帮到有同样问题的同学。

---

## 1. box-sizing 问题

### 问题
小程序中 `box-sizing` 默认值为 `content-box`，且 WXSS 不支持 `*` 通配符选择器，无法全局设置 `border-box`。

### 解决
通过 PostCSS 插件自动注入样式，在编译后的每个 WXSS 文件头部添加：

```css
view, scroll-view, swiper, movable-view, cover-view, text {
  box-sizing: border-box;
}
```

```js
// postcss-plugin/auto-box-sizing.js
const postcss = require('postcss');

module.exports = postcss.plugin('auto-box-sizing', () => {
  return (root) => {
    root.prepend(
      `view, scroll-view, swiper, movable-view, cover-view, text {
        box-sizing: border-box;
      }`
    );
  };
});
```
然后在 `postcss.config.js` 中注册即可。

---

## 2. Input 组件 placeholder 偏移

### 问题
给 Input 组件设置 `height` 后，placeholder 文字会偏离垂直居中，因为小程序 Input 的 `padding` 与 `height` 存在冲突。

### 解决
不要同时设置 `height` 和 `padding`，改为使用 `line-height` 控制高度：

```less
.order-input {
  // ❌ 错误写法
  height: 72px;
  padding: 0 24px;

  // ✅ 正确写法
  padding: 24px;
  line-height: 1.5;
  // 或者用 min-height 代替 height
  min-height: 72px;
}
```

---

## 3. Text 组件不支持块级样式

### 问题
小程序的 Text 组件是行内元素，设置 `display: flex`、`width`、`padding` 等块级样式无效。

### 解决
用 View 替代 Text 做布局容器，Text 只用于纯文本展示：

```jsx
// ❌ 错误：Text 里做 flex 布局
<Text className="wrapper">
  <Text>名称</Text>
  <Text>数量</Text>
</Text>

// ✅ 正确：View 做布局，Text 放文本
<View className="wrapper">
  <Text>名称</Text>
  <Text>数量</Text>
</View>
```

---

## 4. 按钮出界问题

### 问题
删除按钮使用 `position: absolute` 定位时，偶尔会超出屏幕边界，尤其是在滚动容器内。

### 解决
给滚动容器添加 `overflow: hidden`，并确保按钮的父元素有正确的 `position: relative`：

```less
.scroll-container {
  overflow: hidden;
  position: relative;
}

.delete-btn {
  position: absolute;
  right: 0;
  top: 0;
}
```

---

## 5. Zustand 状态管理性能问题

### 问题
在 selector 中调用方法会导致无限渲染循环：

```jsx
// ❌ 错误：每次 render 都创建新引用
const count = useOrderStore(state => state.getCount());

// ✅ 正确：只取原始值，方法在外部调用
const count = useOrderStore(state => state.count);
const getCount = useOrderStore(state => state.getCount);
const value = getCount();
```

当需要多个派生状态时，可以使用 `useShallow` 避免不必要的重渲染：

```jsx
import { useShallow } from 'zustand/react/shallow';

const { total, selected } = useOrderStore(
  useShallow(state => ({
    total: state.total,
    selected: state.selected
  }))
);
```

---

## 总结

| 问题 | 根因 | 解决方案 |
|------|------|----------|
| box-sizing 无效 | WXSS 不支持通配符 | PostCSS 自动注入 |
| placeholder 偏移 | height + padding 冲突 | 用 line-height 代替 |
| Text 不响应布局 | 小程序行内元素限制 | View 做容器 |
| 按钮出界 | 滚动容器定位问题 | overflow: hidden |
| 无限渲染 | selector 调用方法 | 取原始值 + useShallow |

小程序开发虽然坑多，但摸清规律后还是挺顺手的。希望对你有帮助 🚀
