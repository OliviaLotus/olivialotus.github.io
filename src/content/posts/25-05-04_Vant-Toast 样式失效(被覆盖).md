---
title: Vant-Toast 样式失效(被覆盖)
published: 2025-05-04
description: Vant-Toast 样式失效(被覆盖)
tags: [javascript, vue]
category: 前端
---

## 1. Toast 样式全白

项目里 **别的页面正常**，唯独 `某一个` 弹出的 Toast 是 **白背景 + 白色图标** ，还以为是渲染失败。

---

## 2. 排查过程

1. 控制台 DOM 已生成，但 `background: var(--van-toast-background);` 被覆盖
2. 全局搜 CSS 没有任何一行手写 `.van-popup { background:#fff }`
3. 删光文件 `<style>` 依旧白底,并且发现`.van-popup:var(--van-popup-background);`
4. 计算样式对比正常页面发现多了一个样式文件 vant/es/popup/index.css
5. ide 全局搜索发现这个文件设置的`background`为白色
6. 搜索发现是因为有页面用了`<van-popup>`标签,并且由于`unplugin-vue-components`插件会自动引入样式导致样式冲突
7. 设置插件参数

```js
   Components({
      resolvers: [VantResolver({ importStyle: false })],
      })
    //vant/es/popup/index.css没了,样式正常
```

## 3. 根因：两份 CSS 冲突

**Vant 源文件链路(伪代码)：**

```css
/* vant/lib/index.css（全量） */
.van-toast.van-popup {
  --van-popup-background: rgba(0, 0, 0, 0.8);
}

/* vant/es/popup/index.css（按需） */
.van-popup {
  background: var(--van-popup-background, #fff);
}
```

- 如果只加载 **lib/index.css** → 黑底生效。
- 如果 **es/popup/index.css 被插入到后面** → 变量仍是 `#fff` → **Toast 被染白**。

---

## 4. 为什么有的没问题

没用过 `<van-popup>`的页面仅有 `lib/index.css`
用过 `<van-popup>`（或组件库自动引入） `lib/index.css` + `es/popup/index.css`会出问题

---

## 5. 解决方案

### ① 关掉自动样式

```ts
// vite.config.ts
Components({
  resolvers: [VantResolver({ importStyle: false })], // 只自动引组件，不拉样式
});
```

**自己只保留一份全量：**

```ts
// main.ts
import "vant/lib/index.css";
```

### ② 继续按需，但确保覆盖

```css
.van-toast.van-popup {
  --van-popup-background: rgba(0, 0, 0, 0.8) !important;
  color: #fff;
}
```

因为 css 的权重和顺序有点难搞,在 main.ts 最后引入生效
