# CSS 核心知识点手册

> 覆盖范围：布局 · 响应式 · 样式工程化 · 高频面试题  
> 适合：复习 / 面试准备 / 日常查阅

---

## 目录

1. [布局](#一布局)
   - [盒模型](#1-盒模型)
   - [BFC](#2-bfc-块级格式化上下文)
   - [Flex 布局](#3-flex-布局)
   - [Grid 布局](#4-grid-布局)
   - [定位 position](#5-定位-position)
   - [居中方案](#6-居中方案)
   - [两栏布局](#7-两栏布局)
   - [三栏 / 圣杯 / 双飞翼布局](#8-三栏--圣杯--双飞翼布局)
2. [响应式布局与移动端适配](#二响应式布局与移动端适配)
   - [rem / vw / vh 单位](#1-rem--vw--vh-单位)
   - [媒体查询](#2-媒体查询)
   - [安全区域 safe-area](#3-安全区域-safe-area)
   - [sticky 定位](#4-sticky-定位)
3. [样式工程化](#三样式工程化)
   - [CSS Modules](#1-css-modules)
   - [Sass / Less](#2-sass--less)
   - [PostCSS](#3-postcss)
   - [Tailwind CSS & 原子化 CSS](#4-tailwind-css--原子化-css)
   - [CSS-in-JS](#5-css-in-js)
   - [主题切换 & 暗黑模式 & 设计 Token](#6-主题切换--暗黑模式--设计-token)
4. [高频问题](#四高频问题)
   - [display: none vs visibility: hidden vs opacity: 0](#1-display-none-vs-visibility-hidden-vs-opacity-0)
   - [position 各值区别](#2-position-各值区别)
   - [flex: 1 到底代表什么](#3-flex-1-到底代表什么)
   - [Grid 适合什么场景](#4-grid-适合什么场景)
   - [BFC 触发方式与用途](#5-bfc-触发方式与用途)
   - [margin 合并](#6-margin-合并)
   - [z-index 与层叠上下文](#7-z-index-与层叠上下文)
   - [移动端 1px 问题](#8-移动端-1px-问题)
   - [图片自适应](#9-图片自适应)
   - [文本超出省略](#10-文本超出省略)

---

## 一、布局

### 1. 盒模型

每个元素都是一个矩形盒子，由四层构成：**content → padding → border → margin**

#### 两种盒模型

| 属性值 | width/height 包含范围 | 实际渲染宽度（width:200px, padding:10px, border:2px） |
|---|---|---|
| `content-box`（默认） | 仅 content | 224px（内容撑开） |
| `border-box`（推荐） | content + padding + border | 200px（符合直觉） |

```css
/* 全局推荐设置 */
*, *::before, *::after {
  box-sizing: border-box;
}
```

#### margin 合并（折叠）

- **触发条件**：相邻兄弟元素 / 父子元素（垂直方向），水平不合并
- **规则**：取两个 margin 的较大值
- **解决方案**：触发 BFC（给父元素加 `overflow: hidden` / `display: flow-root`）

```css
/* 父子 margin 穿透问题 */
.parent {
  overflow: hidden;  /* 触发 BFC，阻止子 margin 穿透 */
  /* 或 border-top: 1px solid transparent; */
  /* 或 padding-top: 1px; */
}
```

---

### 2. BFC 块级格式化上下文

BFC（Block Formatting Context）是一个**独立的渲染区域**，内部布局不影响外部，外部不影响内部。

#### 触发方式

```css
overflow: hidden | auto | scroll   /* 最常用 */
display: flow-root                 /* 最语义化，专门为此设计 */
display: flex | inline-flex
display: grid | inline-grid
position: absolute | fixed
float: left | right
contain: layout | content | paint
```

#### 三大用途

| 用途 | 说明 |
|---|---|
| 清除浮动 | 父容器触发 BFC，自动包含内部浮动子元素，解决高度塌陷 |
| 阻止 margin 合并 | 两元素处于不同 BFC，vertical margin 不再合并 |
| 防止被浮动覆盖 | BFC 元素会避开浮动，常用于两栏布局左右分隔 |

```css
/* 现代清除浮动方案 */
.parent { display: flow-root; }

/* 经典 clearfix */
.clearfix::after {
  content: '';
  display: block;
  clear: both;
}
```

> **面试记忆**：BFC 能解决三件事 —— 清浮动、防合并、避覆盖。

---

### 3. Flex 布局

Flex 是**一维**布局模型，分为主轴（默认横向）和交叉轴（默认纵向）。

#### 父容器属性

```css
.container {
  display: flex;
  flex-direction: row | column | row-reverse | column-reverse;
  flex-wrap: nowrap | wrap;
  justify-content: flex-start | center | space-between | space-around | space-evenly;
  align-items: stretch | center | flex-start | flex-end | baseline;
  align-content: ...;    /* 多行时控制整体对齐 */
  gap: 12px 16px;        /* 行间距 列间距 */
}
```

#### 子元素属性

```css
.item {
  flex: 1;               /* 简写，见下方详解 */
  flex-grow: 1;          /* 按比例分配剩余空间 */
  flex-shrink: 1;        /* 空间不足时收缩比例 */
  flex-basis: 0%;        /* 初始尺寸 */
  align-self: auto;      /* 单独覆盖 align-items */
  order: 0;              /* 排列顺序 */
}
```

#### flex: 1 详解（高频考点）

| 简写 | 展开 | 含义 |
|---|---|---|
| `flex: 1` | `flex: 1 1 0%` | 从 0 开始，能放大能缩小 |
| `flex: auto` | `flex: 1 1 auto` | 先占内容大小，再分配剩余空间 |
| `flex: none` | `flex: 0 0 auto` | 不伸缩，固定尺寸 |
| `flex: 0` | `flex: 0 1 0%` | 不放大，可缩小 |

```css
/* 常见陷阱：flex item 默认 min-width: auto，内容可能阻止收缩 */
.item {
  flex: 1;
  min-width: 0;  /* 修复内容撑开导致无法收缩的问题 */
  overflow: hidden;
}
```

---

### 4. Grid 布局

Grid 是**二维**布局模型，同时控制行和列。

#### 基础用法

```css
.container {
  display: grid;
  grid-template-columns: repeat(3, 1fr);       /* 3 等分列 */
  grid-template-columns: 200px auto 1fr;        /* 混合单位 */
  grid-template-rows: auto 1fr auto;
  gap: 16px;

  /* 命名区域（直观！）*/
  grid-template-areas:
    "header header header"
    "sidebar main   main"
    "footer  footer footer";
}

/* 子元素使用命名区域 */
.header  { grid-area: header; }
.sidebar { grid-area: sidebar; }

/* 子元素手动定位 */
.item {
  grid-column: 1 / 3;   /* 横跨第1到第3列 */
  grid-row: 2 / 4;      /* 竖跨第2到第4行 */
}
```

#### 响应式神器

```css
/* 不需要媒体查询，自动换行 */
.grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: 16px;
}
```

#### Flex vs Grid 选择

| 场景 | 推荐 |
|---|---|
| 导航栏、工具栏、按钮组（一维排列，内容决定尺寸） | Flex |
| 页面整体框架、仪表盘、图片网格（二维精确控制） | Grid |
| 卡片列表（等宽自适应列数） | Grid (auto-fit) |
| 表单元素对齐 | 均可 |

---

### 5. 定位 position

| 值 | 参照物 | 脱离文档流 | 典型用途 |
|---|---|---|---|
| `static` | 无（默认） | 否 | 正常流 |
| `relative` | 自身原始位置 | 否（保留空间） | 微调位置 / 作为子元素绝对定位的参照 |
| `absolute` | 最近的定位祖先（非 static） | 是 | 弹出层、角标、tooltip |
| `fixed` | 视口 viewport | 是 | 固定导航、返回顶部按钮 |
| `sticky` | 滚动容器 | 否（阈值前） | 吸顶表头、侧边栏跟随 |

```css
/* absolute 定位参照祖先 */
.parent { position: relative; }  /* 设置参照 */
.badge {
  position: absolute;
  top: -6px;
  right: -6px;
}

/* fixed 注意：父元素有 transform 时，fixed 会相对父元素而非视口定位！ */
```

---

### 6. 居中方案

#### 水平垂直居中（6 种）

```css
/* 方案1：Flex（推荐）*/
.parent {
  display: flex;
  justify-content: center;
  align-items: center;
}

/* 方案2：Grid（最简洁）*/
.parent {
  display: grid;
  place-items: center;
}

/* 方案3：absolute + transform（不需知道子元素尺寸）*/
.parent { position: relative; }
.child {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
}

/* 方案4：absolute + margin: auto（需要子元素有固定尺寸）*/
.child {
  position: absolute;
  inset: 0;          /* top/right/bottom/left: 0 */
  margin: auto;
  width: 200px;
  height: 100px;
}

/* 方案5：table-cell（老方法，了解即可）*/
.parent {
  display: table-cell;
  vertical-align: middle;
  text-align: center;
}

/* 方案6：line-height（仅适用于单行文本）*/
.parent { height: 40px; line-height: 40px; text-align: center; }
```

---

### 7. 两栏布局

左侧固定，右侧自适应。

```css
/* 方案1：Flex（推荐）*/
.container { display: flex; }
.left  { width: 200px; flex-shrink: 0; }
.right { flex: 1; min-width: 0; }

/* 方案2：float + BFC */
.left  { float: left; width: 200px; }
.right { overflow: hidden; }  /* 触发 BFC，自动避开浮动 */

/* 方案3：Grid */
.container { display: grid; grid-template-columns: 200px 1fr; }
```

---

### 8. 三栏 / 圣杯 / 双飞翼布局

**目标**：左右固定宽度，中间自适应，且中间列 HTML 优先渲染（SEO 友好）。

#### 现代方案（日常开发）

```css
/* Flex */
.container { display: flex; }
.left  { width: 200px; flex-shrink: 0; }
.main  { flex: 1; min-width: 0; }
.right { width: 200px; flex-shrink: 0; }

/* Grid */
.container { display: grid; grid-template-columns: 200px 1fr 200px; }
```

#### 圣杯布局（面试考点）

```html
<!-- HTML 结构：main 在最前 -->
<div class="container">
  <div class="main">中间</div>
  <div class="left">左侧</div>
  <div class="right">右侧</div>
</div>
```

```css
.container { padding: 0 200px; overflow: hidden; }
.main  { float: left; width: 100%; }
.left  {
  float: left; width: 200px;
  margin-left: -100%;          /* 移到 main 左侧 */
  position: relative; left: -200px;  /* 移出 padding 区域 */
}
.right {
  float: left; width: 200px;
  margin-left: -200px;         /* 移到 main 右侧 */
  position: relative; right: -200px;
}
```

#### 双飞翼布局（圣杯变体）

```html
<div class="main-wrap">
  <div class="main-inner">中间</div>  <!-- 多一层 wrapper -->
</div>
<div class="left">左侧</div>
<div class="right">右侧</div>
```

```css
.main-wrap  { float: left; width: 100%; }
.main-inner { margin: 0 200px; }      /* 用 margin 代替圣杯的 padding + position */
.left  { float: left; width: 200px; margin-left: -100%; }
.right { float: left; width: 200px; margin-left: -200px; }
```

> **圣杯 vs 双飞翼**：圣杯用父容器 `padding` + 子元素 `position: relative` 偏移；双飞翼用 inner wrapper 的 `margin` 留出空间，避免了 relative 定位，代码更清晰。

---

## 二、响应式布局与移动端适配

### 1. rem / vw / vh 单位

| 单位 | 含义 | 典型用途 |
|---|---|---|
| `px` | CSS 像素 | 边框、固定阴影 |
| `em` | 相对父元素字号 | padding 与字号联动 |
| `rem` | 相对根元素（html）字号 | 字号、间距的等比缩放 |
| `vw` | 视口宽度的 1/100 | 宽度流体布局 |
| `vh` | 视口高度的 1/100 | 全屏高度（注意移动端坑） |
| `dvh` | 动态视口高度 | 移动端替代 vh（推荐） |
| `clamp()` | 响应式值范围 | 流体字号 / 间距 |

```css
/* rem 适配：根字号随屏宽变化 */
html { font-size: calc(100vw / 375 * 16); }  /* 设计稿 375px 基准 */

/* clamp() 流体字号（推荐现代写法）*/
h1 { font-size: clamp(1.5rem, 4vw, 3rem); }
/* 最小 1.5rem，随视口宽度线性变化，最大 3rem */

/* dvh：解决移动端地址栏遮盖问题 */
.hero { height: 100dvh; }  /* 推荐替代 100vh */
```

> **100vh 的坑**：移动端浏览器地址栏可以收起/展开，`vh` 以展开状态计算，滚动后地址栏收起会导致内容溢出。`dvh`（dynamic viewport height）实时跟随地址栏变化。

---

### 2. 媒体查询

```css
/* Mobile First（推荐）：从小屏往大屏写 */
.container { padding: 0 16px; }

@media (min-width: 768px) {    /* tablet */
  .container { max-width: 720px; margin: 0 auto; }
}

@media (min-width: 1024px) {   /* desktop */
  .container { max-width: 1200px; }
}

/* 其他常用查询条件 */
@media (prefers-color-scheme: dark) { /* 系统暗黑模式 */ }
@media (prefers-reduced-motion: reduce) { /* 用户要求减少动画 */ }
@media (hover: hover) { /* 设备支持悬停（有鼠标）*/ }
@media (orientation: landscape) { /* 横屏 */ }
@media print { /* 打印样式 */ }
```

> **Mobile First 原则**：默认样式写手机，用 `min-width` 逐步增强大屏。比 Desktop First（用 `max-width` 降级）更符合渐进增强原则，性能也更好。

---

### 3. 安全区域 safe-area

iPhone X 及以后机型有刘海（顶部）和 Home 指示条（底部），内容可能被遮挡。

```html
<!-- 第一步：viewport meta 加 viewport-fit=cover -->
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
```

```css
/* 第二步：用 env() 获取安全区域尺寸 */
.bottom-nav {
  /* max() 保证最小间距，safe-area 更大时自动取安全区域值 */
  padding-bottom: max(16px, env(safe-area-inset-bottom));
}

.header {
  padding-top: env(safe-area-inset-top);
}

/* 四个方向 */
env(safe-area-inset-top)
env(safe-area-inset-right)
env(safe-area-inset-bottom)
env(safe-area-inset-left)
```

---

### 4. sticky 定位

`position: sticky` 是 `relative` + `fixed` 的混合：滚动到阈值前表现为 `relative`，超过后固定在设定位置，直到父容器边缘离开视口。

```css
.sticky-header {
  position: sticky;
  top: 0;           /* 必须设置 top / bottom / left / right 之一 */
  z-index: 100;
}
```

#### sticky 失效的常见原因

| 原因 | 解决方案 |
|---|---|
| 未设置 `top` / `bottom` | 添加 `top: 0` |
| 祖先元素有 `overflow: hidden / auto / scroll` | 移除该属性，或重新设计结构 |
| 父容器高度等于 sticky 元素本身高度 | 父容器需有足够高度提供"滑动空间" |
| sticky 元素高度超过视口 | 元素太高导致无法固定 |

---

## 三、样式工程化

### 1. CSS Modules

构建时将类名编译为唯一 hash 值，实现**局部作用域**，彻底解决全局命名污染。

```css
/* Button.module.css */
.btn { padding: 8px 16px; background: blue; color: white; }
.btn:hover { opacity: 0.8; }
```

```jsx
// Button.jsx
import styles from './Button.module.css';

function Button() {
  // 编译后 → class="Button_btn__xK2d9"
  return <button className={styles.btn}>Click</button>;
}
```

```css
/* :global 跳过 hash 处理（覆盖第三方库）*/
:global(.ant-btn) { border-radius: 4px; }

/* composes 复用其他类 */
.base { padding: 8px; border-radius: 4px; }
.primary {
  composes: base;          /* 继承 base */
  background: blue;
}
```

> Vue 的 `<style scoped>` 原理类似：自动给所有选择器加 `[data-v-hash]` 属性选择器实现隔离。

---

### 2. Sass / Less

CSS 预处理器，增强 CSS 语法，编译为标准 CSS。

#### Sass（SCSS 语法）核心特性

```scss
// 变量
$primary: #3498db;
$font-size-base: 16px;

// 嵌套（支持 BEM）
.nav {
  background: $primary;
  &:hover { opacity: 0.8; }
  &__item { font-size: $font-size-base; }
  &--active { font-weight: 500; }
}

// Mixin（带参数的代码复用）
@mixin flex-center($direction: row) {
  display: flex;
  align-items: center;
  justify-content: center;
  flex-direction: $direction;
}
.hero { @include flex-center(column); }

// %placeholder（选择器合并，比 mixin 更省体积）
%btn-base { padding: 8px 16px; border-radius: 4px; cursor: pointer; }
.btn-primary { @extend %btn-base; background: $primary; }
.btn-danger  { @extend %btn-base; background: red; }
// 编译为：.btn-primary, .btn-danger { padding: 8px 16px; ... }

// 循环
@each $color in red, blue, green {
  .text-#{$color} { color: $color; }
}

@for $i from 1 through 5 {
  .col-#{$i} { flex: $i; }
}

// 函数
@function rem($px, $base: 16) {
  @return #{$px / $base}rem;
}
.title { font-size: rem(24); }  /* → 1.5rem */

// 条件
@mixin theme($mode) {
  @if $mode == dark {
    background: #000; color: #fff;
  } @else {
    background: #fff; color: #000;
  }
}
```

> 现代项目推荐 Sass（SCSS 语法）。Less 使用率逐年下降。

---

### 3. PostCSS

PostCSS 是 CSS 的 **Babel**，通过插件平台对 CSS 做 AST 级别的转换，不是预处理器。

```js
// postcss.config.js
module.exports = {
  plugins: [
    require('autoprefixer'),           // 自动加浏览器前缀（最常用）
    require('postcss-preset-env'),     // 使用未来 CSS 语法
    require('postcss-nested'),         // 支持嵌套语法（类 Sass）
    require('cssnano'),                // 压缩 CSS
    require('postcss-px-to-viewport'), // px 转 vw（移动端适配）
  ]
};
```

```css
/* 输入 */
.box { display: flex; user-select: none; }

/* autoprefixer 输出 */
.box {
  display: -webkit-box;
  display: -ms-flexbox;
  display: flex;
  -webkit-user-select: none;
  -moz-user-select: none;
  user-select: none;
}
```

> Vite 和 CRA 内置 PostCSS，只需配置插件。`autoprefixer` + `browserslist` 是最常用组合。

---

### 4. Tailwind CSS & 原子化 CSS

**原子化 CSS**：每个类只做一件事，直接在 HTML 上组合类名，无需自定义命名。

```html
<!-- 传统 CSS：需要命名、切换文件 -->
<button class="submit-btn">提交</button>

<!-- Tailwind：直接组合工具类 -->
<button class="flex items-center px-4 py-2 bg-blue-500 text-white rounded-md hover:bg-blue-600 transition-colors">
  提交
</button>
```

#### 工作机制

- **JIT（Just-in-Time）**：构建时扫描源码类名，按需生成 CSS，最终产物极小
- **任意值**：`w-[137px]`、`text-[#ff6b6b]`、`mt-[calc(100vh-64px)]`
- **@apply**：在 CSS 中复用工具类

```css
/* @apply：避免 HTML 中类名过长 */
.btn {
  @apply flex items-center px-4 py-2 bg-blue-500 text-white rounded transition-colors;
}
.btn:hover {
  @apply bg-blue-600;
}
```

```js
// tailwind.config.js（v3）
module.exports = {
  content: ['./src/**/*.{html,js,jsx,tsx}'],  // 扫描路径
  theme: {
    extend: {
      colors: { brand: '#ff6b6b' },
      spacing: { '18': '4.5rem' },
    }
  }
};
```

> Tailwind v4 改用 CSS-first 配置（`@theme`），抛弃了 `tailwind.config.js`。

---

### 5. CSS-in-JS

将 CSS 写在 JS/TS 文件中，与组件逻辑放在一起，代表库：`styled-components`、`emotion`。

```js
// styled-components
import styled from 'styled-components';

const Button = styled.button`
  padding: 8px 16px;
  background: ${props => props.$primary ? 'blue' : 'white'};
  color: ${props => props.$primary ? 'white' : 'blue'};
  border: 2px solid blue;
  border-radius: 4px;

  &:hover { opacity: 0.8; }
`;

// 使用
<Button $primary>主按钮</Button>
<Button>次要按钮</Button>
```

```js
// emotion（更轻量，支持 css prop）
/** @jsxImportSource @emotion/react */
import { css } from '@emotion/react';

const cardStyle = css`
  padding: 1rem;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.1);
`;

<div css={cardStyle}>卡片内容</div>
```

#### 优缺点

| | 优点 | 缺点 |
|---|---|---|
| CSS-in-JS | 动态样式、自动作用域、TypeScript 友好 | 运行时开销、RSC 不兼容、包体积较大 |
| 零运行时方案 | 无运行时性能损耗、RSC 兼容 | 动态能力受限 |

> React 生态正从运行时 CSS-in-JS 转向零运行时方案：`Linaria`、`vanilla-extract`、Meta 的 `StyleX`。

---

### 6. 主题切换 & 暗黑模式 & 设计 Token

#### 设计 Token

将颜色、字号、间距等设计决策存为变量，是设计系统的基础。

```css
/* tokens.css：定义所有设计变量 */
:root {
  /* 颜色 */
  --color-primary: #3b82f6;
  --color-primary-hover: #2563eb;
  --color-bg: #ffffff;
  --color-surface: #f9fafb;
  --color-text: #111827;
  --color-text-muted: #6b7280;
  --color-border: #e5e7eb;

  /* 字号 */
  --font-size-sm: 0.875rem;
  --font-size-base: 1rem;
  --font-size-lg: 1.125rem;
  --font-size-xl: 1.25rem;

  /* 间距 */
  --space-1: 0.25rem;
  --space-2: 0.5rem;
  --space-4: 1rem;
  --space-8: 2rem;

  /* 圆角 */
  --radius-sm: 4px;
  --radius-md: 8px;
  --radius-lg: 12px;
}
```

#### 暗黑模式

```css
/* 方式1：跟随系统 */
@media (prefers-color-scheme: dark) {
  :root {
    --color-bg: #0f172a;
    --color-surface: #1e293b;
    --color-text: #f1f5f9;
    --color-text-muted: #94a3b8;
    --color-border: #334155;
  }
}

/* 方式2：手动切换（data-theme 属性，可覆盖系统）*/
[data-theme="dark"] {
  --color-bg: #0f172a;
  --color-surface: #1e293b;
  --color-text: #f1f5f9;
}
```

```js
// JS 切换主题
const toggleTheme = () => {
  const current = document.documentElement.getAttribute('data-theme');
  document.documentElement.setAttribute('data-theme', current === 'dark' ? 'light' : 'dark');
};

// 初始化：优先读用户偏好，再跟随系统
const savedTheme = localStorage.getItem('theme');
const prefersDark = window.matchMedia('(prefers-color-scheme: dark)').matches;
document.documentElement.setAttribute('data-theme', savedTheme || (prefersDark ? 'dark' : 'light'));

// 监听系统主题变化
window.matchMedia('(prefers-color-scheme: dark)')
  .addEventListener('change', e => {
    if (!localStorage.getItem('theme')) {
      document.documentElement.setAttribute('data-theme', e.matches ? 'dark' : 'light');
    }
  });
```

---

## 四、高频问题

### 1. display: none vs visibility: hidden vs opacity: 0

| 属性 | 占据空间 | 是否渲染 | 响应事件 | 子元素可见 | 性能影响 |
|---|---|---|---|---|---|
| `display: none` | ❌ | 不渲染 | ❌ | 不可（全部隐藏） | 触发回流（最重） |
| `visibility: hidden` | ✅ | 渲染 | ❌ | 可单独设 visible | 只重绘 |
| `opacity: 0` | ✅ | 渲染 | ✅ 仍可点击！ | 全部透明 | GPU 合成（最轻） |

```css
/* visibility 子元素单独覆盖（display: none 做不到）*/
.parent { visibility: hidden; }
.child  { visibility: visible; }  /* 子元素仍可见 */

/* opacity: 0 仍然可以被点击（常见坑）*/
.hidden { opacity: 0; pointer-events: none; }  /* 加 pointer-events 禁用交互 */
```

> **动画性能**：opacity + transform 由 GPU 合成，不触发回流重绘，是性能最好的动画属性。

---

### 2. position 各值区别

见[定位 position](#5-定位-position) 章节。

```css
/* 常见面试题：relative vs absolute */
/* relative：相对自身移动，原空间保留，不脱流 */
.box { position: relative; top: 10px; }  /* 下移 10px，但原位置仍占据 */

/* absolute：脱离文档流，相对最近定位祖先 */
.parent { position: relative; }
.child  { position: absolute; top: 0; right: 0; }  /* 右上角 */
```

---

### 3. flex: 1 到底代表什么

```css
flex: 1  →  flex-grow: 1; flex-shrink: 1; flex-basis: 0%;
```

- `flex-grow: 1`：有剩余空间时，按比例放大
- `flex-shrink: 1`：空间不足时，按比例缩小
- `flex-basis: 0%`：初始尺寸从 0 开始计算（与 `flex: auto` 的 `basis: auto` 不同）

```css
/* 等分三列：每列都从 0 开始平分容器宽度 */
.col { flex: 1; }

/* 常见坑：内容撑开导致无法等分 */
.col { flex: 1; min-width: 0; }  /* 加 min-width: 0 修复 */

/* flex: 1 vs flex: auto 的差异 */
/* flex: 1（basis: 0%）：三列内容不等，宽度仍等分 */
/* flex: auto（basis: auto）：三列内容不等，宽度按内容比例分配剩余空间 */
```

---

### 4. Grid 适合什么场景

- **页面整体布局**：header / sidebar / main / footer
- **卡片网格**：图片墙、商品列表（配合 `auto-fit`）
- **仪表盘 Dashboard**：不规则跨列/跨行的组件排列
- **表单布局**：标签和输入框的精确对齐
- **不适合**：简单的一维排列（用 Flex 更简洁）

---

### 5. BFC 触发方式与用途

见 [BFC 章节](#2-bfc-块级格式化上下文)。

---

### 6. margin 合并

**条件**：块级元素在垂直方向相邻（兄弟或父子），margin 取较大值而非相加。

```css
/* 场景1：相邻兄弟 */
.a { margin-bottom: 20px; }
.b { margin-top: 30px; }
/* 实际间距：30px（非 50px）*/

/* 场景2：父子（无 border/padding 隔离时）*/
.parent { margin-top: 0; }
.child  { margin-top: 20px; }
/* 结果：.parent 的 margin-top 变为 20px（穿透）*/

/* 解决方案 */
.parent {
  overflow: hidden;      /* 方案1：触发 BFC */
  /* 或 */
  border-top: 1px solid transparent;  /* 方案2：建立隔离 */
  /* 或 */
  padding-top: 1px;      /* 方案3：padding 隔离 */
  /* 或 */
  display: flow-root;    /* 方案4：最语义化 */
}
```

---

### 7. z-index 与层叠上下文

#### 层叠顺序（同一层叠上下文内，从低到高）

```
background / border
→ 负 z-index
→ block 块级元素
→ float 浮动元素
→ inline / inline-block 内联元素
→ z-index: 0 / auto
→ 正 z-index
```

#### 触发层叠上下文的条件

```css
position: relative/absolute/fixed/sticky + z-index 非 auto
opacity < 1
transform 非 none
filter 非 none
will-change: transform | opacity
isolation: isolate            /* 专门创建，推荐 */
display: flex/grid 的子元素且 z-index 非 auto
```

#### 常见坑

```css
/* 坑1：父有 transform，子 fixed 失效 */
.parent { transform: translateX(0); }  /* 创建了新层叠上下文 */
.child  { position: fixed; top: 0; }  /* 相对父元素而非视口！*/

/* 修复：使用 isolation */
.parent { isolation: isolate; }  /* 创建层叠上下文但不影响 fixed */

/* 坑2：z-index 大不代表在上面 */
/* 不同父层叠上下文中的元素，比的是父上下文的层级，不是子元素的 z-index */
```

---

### 8. 移动端 1px 问题

**原因**：Retina 屏 DPR（Device Pixel Ratio）= 2 或 3，CSS 的 1px border 在物理屏幕上显示为 2-3 个物理像素，看起来偏粗。

```css
/* 方案1：transform 缩放（推荐，最灵活）*/
.border-bottom::after {
  content: '';
  position: absolute;
  left: 0; bottom: 0;
  width: 100%;
  height: 1px;
  background: #ddd;
  transform: scaleY(0.5);     /* DPR=2 */
  transform-origin: bottom;
}

/* 方案2：box-shadow 模拟（简单场景）*/
.item { box-shadow: 0 0.5px 0 #ddd; }  /* 底部 0.5px */
.item { box-shadow: 0 0 0 0.5px #ddd; } /* 四边 0.5px */

/* 方案3：根据 DPR 动态设置（JS 配合）*/
const dpr = window.devicePixelRatio;
const scale = 1 / dpr;
// <meta name="viewport" content="...,initial-scale=0.5"> 影响全局，慎用

/* 方案4：SVG border（清晰度最好，成本最高）*/
.item {
  border-image: url("data:image/svg+xml,...") 2 stretch;
}
```

---

### 9. 图片自适应

```css
/* 基础：防止图片溢出容器 */
img { max-width: 100%; height: auto; }

/* object-fit：控制图片填充方式 */
.thumbnail {
  width: 200px; height: 150px;
  object-fit: cover;         /* 裁剪填满（最常用）*/
  object-position: center;   /* 裁剪锚点 */
}
/* cover：等比缩放填满，超出裁剪 */
/* contain：等比缩放放入，不裁剪，可能留白 */
/* fill：拉伸填满（不保持比例，慎用）*/

/* aspect-ratio：保持宽高比 */
.video-wrapper {
  width: 100%;
  aspect-ratio: 16 / 9;
  overflow: hidden;
}
img { width: 100%; height: 100%; object-fit: cover; }
```

```html
<!-- 响应式图片：srcset 让浏览器选最合适的 -->
<img
  src="img-800.jpg"
  srcset="img-400.jpg 400w, img-800.jpg 800w, img-1600.jpg 1600w"
  sizes="(max-width: 600px) 400px, (max-width: 1200px) 800px, 1600px"
  loading="lazy"
  decoding="async"
  alt="描述"
>

<!-- picture：不同格式 + 不同尺寸 -->
<picture>
  <source type="image/avif" srcset="img.avif">
  <source type="image/webp" srcset="img.webp">
  <img src="img.jpg" alt="描述">
</picture>
```

---

### 10. 文本超出省略

```css
/* 单行省略（三件套）*/
.single-line {
  white-space: nowrap;      /* 禁止换行 */
  overflow: hidden;         /* 裁剪超出 */
  text-overflow: ellipsis;  /* 显示 ... */
}

/* 多行省略（WebKit，已广泛支持）*/
.multi-line {
  display: -webkit-box;
  -webkit-box-orient: vertical;
  -webkit-line-clamp: 3;    /* 显示行数 */
  overflow: hidden;
  /* 注意：不需要 text-overflow: ellipsis */
}

/* 配合 JS 动态设置行数 */
el.style.webkitLineClamp = lines;

/* 渐变遮罩方案（跨浏览器，无省略号但效果更优雅）*/
.fade-clamp {
  position: relative;
  max-height: calc(1.6em * 3);  /* 行高 × 行数 */
  overflow: hidden;
}
.fade-clamp::after {
  content: '';
  position: absolute;
  bottom: 0; right: 0;
  width: 50%; height: 1.6em;
  background: linear-gradient(to right, transparent, white);
}
```

> `-webkit-line-clamp` 虽有 `-webkit` 前缀，但 Chrome / Firefox / Safari 均已支持，可放心在生产环境使用。

---

## 速查备忘

| 需求 | 方案 |
|---|---|
| 元素水平垂直居中 | `display: grid; place-items: center` |
| 清除浮动 | `display: flow-root` |
| 阻止 margin 合并 | 父元素触发 BFC |
| 移动端等比缩放 | `clamp()` 或 rem + 根字号 |
| 安全区域适配 | `env(safe-area-inset-*)` + `viewport-fit=cover` |
| 1px 细线 | `::after` + `scaleY(0.5)` |
| 图片填满容器不变形 | `object-fit: cover` |
| 单行省略 | `white-space: nowrap; overflow: hidden; text-overflow: ellipsis` |
| 多行省略 | `-webkit-line-clamp: N` |
| sticky 失效排查 | 检查祖先是否有 `overflow: hidden` |
| z-index 不生效 | 检查是否在同一层叠上下文，父是否触发了新上下文 |
| flex item 无法收缩 | 加 `min-width: 0` |
| 暗黑模式 | CSS 变量 + `[data-theme="dark"]` 或 `prefers-color-scheme` |

---

*最后更新：2025年*
