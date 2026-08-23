# Vue 3 源码级链路：从 `.vue` 文件到真实 DOM

这条链路横跨"编译时"（`@vue/compiler-sfc` + `@vue/compiler-core`）和"运行时"（`@vue/runtime-core` + `@vue/reactivity`）两个包，中间用 `render` 函数当接口对接起来。逐段拆。

---

## 一、`.vue` 文件：先过 SFC 编译器拆块

`.vue` 文件本身不是标准语法，第一步由 `@vue/compiler-sfc` 把它切成三块：

```
App.vue
  ├─ <template>...</template>  → 交给 compiler-core 编译成 render 函数
  ├─ <script setup>...</script> → 基本保留原样(做少量宏展开,如 defineProps)
  └─ <style>...</style>         → 走 CSS 处理(scoped 样式加 data-v-xxx 属性选择器)
```

这一步产出的是一个"伪模块"：

```js
import { render } from './App.vue?vue&type=template'
export default {
  setup() { /* script setup 编译后的内容 */ },
  render // 挂上模板编译出的 render 函数
}
```

真正硬核的部分在 `<template>` 那一路，下面细看。

---

## 二、Template → 模板 AST：`compiler-core` 的 `parse` 阶段

`compiler-core/src/parse.ts` 里的 `baseParse()` 是个手写的**状态机式解析器**（不依赖第三方 HTML parser），核心是维护一个"父节点栈" + 逐字符扫描：

```js
// 简化后的 AST 节点结构
{
  type: NodeTypes.ELEMENT,
  tag: 'div',
  props: [
    { type: NodeTypes.DIRECTIVE, name: 'if', exp: {...} },  // v-if
    { type: NodeTypes.ATTRIBUTE, name: 'class', value: 'red' }
  ],
  children: [
    { type: NodeTypes.INTERPOLATION, content: { content: 'msg' } }  // {{ msg }}
  ]
}
```

解析器遇到 `{{ }}` 会调用 `parseInterpolation()`，遇到 `v-xxx`/`:xxx`/`@xxx` 会调用 `parseAttribute()` 并识别成指令节点，遇到普通标签走 `parseElement()` 递归处理子节点。最终产出一棵**根节点为 `Root`** 的模板 AST。

---

## 三、Transform 阶段：AST 遍历 + 关键优化标记

这是整条链路里最值得深挖的一段。`transform()` 用**插件化的 AST 遍历（类似 Babel 的 Visitor 模式）**，对每个节点跑一遍注册好的 transform 函数（`transformIf`、`transformFor`、`transformElement`、`transformExpression`……）。

### 3.1 静态提升（Hoist Static）

`hoistStatic` transform 会检测"完全不依赖响应式数据的子树"，把它提到 render 函数外面，只创建一次：

```js
// 编译前
<div><span>纯文本</span><span>{{ msg }}</span></div>

// 编译后
const _hoisted_1 = /*#__PURE__*/createElementVNode("span", null, "纯文本")
// hoisted_1 在模块加载时创建一次，之后每次 render 都复用同一个 VNode 对象引用
function render() {
  return createElementVNode("div", null, [
    _hoisted_1,  // 直接复用，不重新创建
    createElementVNode("span", null, toDisplayString(msg), 1 /* TEXT */)
  ])
}
```

### 3.2 PatchFlag：给动态节点打"变化类型"标记

这是 Vue 3 相比 Vue 2 最核心的编译时优化。`transformElement` 在处理每个节点时，会分析它的动态绑定属于哪一类，编码成一个数字标记（在 `@vue/shared/src/patchFlags.ts` 定义）：

```js
export const enum PatchFlags {
  TEXT = 1,           // 动态文本内容
  CLASS = 1 << 1,     // 2,  动态 class
  STYLE = 1 << 2,     // 4,  动态 style
  PROPS = 1 << 3,     // 8,  动态属性（非 class/style）
  FULL_PROPS = 1 << 4,// 16, 有动态 key，需要全量 diff props
  // ... 还有 STABLE_FRAGMENT、KEYED_FRAGMENT 等
}
```

比如 `<div :class="cls">{{ msg }}</div>` 编译出来：

```js
createElementVNode("div", { class: cls }, toDisplayString(msg), 
  3 /* TEXT, CLASS */)  // 3 = 1 | 2，二进制标记的按位或
```

这个 `3` 会一路带到运行时的 diff 阶段（下面第六节讲），diff 时直接用位运算 `patchFlag & PatchFlags.CLASS` 判断"要不要比较 class"，跳过所有没被标记的属性比较——这就是所谓**编译时信息指导运行时行为**。

### 3.3 生成阶段：AST → 代码字符串

`generate()` 遍历打好标记的 AST，拼接出最终 render 函数的源码字符串（本质是字符串拼接 + 一点点 sourcemap 记录），大致长这样：

```js
export function render(_ctx, _cache) {
  return (_openBlock(), _createElementBlock("div", null, [
    _hoisted_1,
    _createElementVNode("span", null, _toDisplayString(_ctx.msg), 1 /* TEXT */)
  ]))
}
```

这个字符串最终被 `new Function(...)` 动态求值（运行时编译场景），或者在构建时直接由 Vite 插件写进 bundle（`vue-loader`/`@vitejs/plugin-vue` 场景）。

---

## 四、render 函数执行：产出 VNode 树

`render()` 函数本身只是一堆 `createElementVNode`（简写 `h` / `_createElementBlock`）调用的组合。VNode 的真实结构大致是：

```js
{
  type: 'div',              // 或组件对象、Symbol(Fragment/Text)
  props: { class: cls },
  children: [...],
  key: null,
  patchFlag: 3,
  dynamicChildren: [...],   // Block Tree 专用，见下节
  el: null,                 // 挂载后指向真实 DOM 节点
  component: null,          // 如果 type 是组件，指向组件实例
}
```

VNode 只是描述"应该长什么样"的**纯数据对象**，还没有跟真实 DOM 发生任何关系。

---

## 五、Block Tree：不止 PatchFlag，还有结构级优化

除了单节点的 PatchFlag，Vue 3 还引入了 **Block（块）** 的概念（`openBlock()`/`createElementBlock()`），这是比 PatchFlag 更进一步的优化，值得单独展开。

一个 Block 会在生成 VNode 树时，**顺带收集自己子树里所有带 PatchFlag 的动态节点**，拍平放进一个数组 `dynamicChildren`：

```js
// 即使 div 嵌套了 5 层静态 div，最终 dynamicChildren 也只放真正动态的那 1 个 span
{
  type: 'div',
  dynamicChildren: [
    { type: 'span', patchFlag: 1 /* TEXT */, ... }
  ]
}
```

这样运行时 diff 完全不需要**递归遍历整棵树**去找哪里变了，直接遍历这个拍平的 `dynamicChildren` 数组即可——这叫 **Block Tree 的靶向更新（Targeted Updates）**，是 Vue 3 比 Vue 2（全量递归 diff）快的另一半原因（另一半是 PatchFlag）。

---

## 六、响应式触发 render：`effect` 与组件更新函数

VNode 树不会自己重新生成，得靠响应式系统驱动。组件挂载时，`mountComponent()` 会创建一个 `ReactiveEffect`：

```js
// runtime-core/src/renderer.ts（简化）
const componentUpdateFn = () => {
  if (!instance.isMounted) {
    const subTree = instance.render.call(proxy)  // 首次执行 render，拿到 VNode 树
    patch(null, subTree, container)              // n1=null，走全量挂载
    instance.isMounted = true
  } else {
    const nextTree = instance.render.call(proxy) // 重新执行 render，拿到新 VNode 树
    patch(instance.subTree, nextTree, container)  // n1=旧树，n2=新树，走 diff
    instance.subTree = nextTree
  }
}
instance.update = () => effect.run()
const effect = new ReactiveEffect(componentUpdateFn, scheduler)
```

`@vue/reactivity` 里 `ref`/`reactive` 用 `Proxy` 拦截 `get`（调用 `track()` 收集依赖，把当前正在执行的 `effect` 存进这个属性对应的 `Dep` 集合）和 `set`（调用 `trigger()`，遍历 `Dep` 通知所有订阅的 `effect` 重新执行）。由于 `componentUpdateFn` 执行时读取了模板里用到的响应式数据，它自动成为这些数据的订阅者——数据一变，`componentUpdateFn` 被重新调度执行，进而重新跑一遍 render。

这里还有个关键优化：`scheduler` 不会同步立即重新执行，而是丢进一个**微任务队列（nextTick 的实现基础）**，同一个 tick 里多次数据变化只会触发一次组件更新——这是"批量更新"的来源。

---

## 七、patch：VNode diff 与真实 DOM 操作

`patch(n1, n2, container)` 是整个 renderer 的调度中枢（`runtime-core/src/renderer.ts`），根据新旧 VNode 的类型分发到不同处理函数：

```js
function patch(n1, n2, container) {
  if (n1 && !isSameVNodeType(n1, n2)) {
    unmount(n1)   // 类型都变了，直接卸载旧的
    n1 = null
  }
  const { type } = n2
  if (type === Text) processText(n1, n2, container)
  else if (type === Fragment) processFragment(n1, n2, container)
  else if (typeof type === 'object') processComponent(n1, n2, container) // 组件
  else processElement(n1, n2, container)  // 普通标签
}
```

对普通元素走到 `patchElement()` 时，才真正用到前面编译期打的 PatchFlag：

```js
function patchElement(n1, n2) {
  const el = n2.el = n1.el  // 复用旧的真实 DOM 节点引用
  const { patchFlag, dynamicChildren } = n2
  
  if (patchFlag & PatchFlags.CLASS) {
    if (n1.props.class !== n2.props.class) hostPatchProp(el, 'class', ..., n2.props.class)
  }
  if (patchFlag & PatchFlags.TEXT) {
    if (n1.children !== n2.children) hostSetElementText(el, n2.children)
  }
  
  if (dynamicChildren) {
    patchBlockChildren(n1.dynamicChildren, dynamicChildren, el) // 靶向更新，跳过静态子树
  }
}
```

`hostPatchProp`/`hostSetElementText` 这些"host"前缀的函数是**平台无关的抽象接口**——在浏览器环境里，它们最终调用的就是原生 DOM API：

```js
// runtime-dom/src/nodeOps.ts
export const nodeOps = {
  setElementText: (el, text) => { el.textContent = text },
  insert: (child, parent, anchor) => parent.insertBefore(child, anchor || null),
  createElement: (tag) => document.createElement(tag),
  patchProp: (el, key, prevValue, nextValue) => { /* 分发到 class/style/event/attr 各自的处理 */ }
}
```

这一层抽象也是 Vue 3 能同时支持 `runtime-dom`（浏览器）和 `runtime-test`（测试环境）、第三方能做 `runtime-core` 自定义渲染器（比如渲染到 Canvas、小程序）的原因——`patch` 算法本身完全不知道"DOM"是什么，只知道调用传进来的 `nodeOps`。

---

## 八、DOM 变化后：交给浏览器渲染管线接手

到 `el.textContent = ...` / `el.setAttribute(...)` 这一步，Vue 的工作就结束了，接力棒交还给浏览器：

```
Vue patch 阶段最后一步：真实 DOM 节点属性/内容被修改
    ↓
浏览器检测到 DOM/CSSOM 变化 → 标记对应 Render Tree 节点 "dirty"
    ↓
下一帧（requestAnimationFrame 时机）浏览器跑 Layout(如果需要) → Paint → Composite
    ↓
Pixel
```

这里能跟你上一条消息的重排重绘表联动一下：`hostPatchProp` 里如果改的是 `class`/`style` 中跟布局无关的属性（如 `color`），只触发 Paint；如果改的是会影响尺寸的属性，触发完整 Layout。Vue 自己无法控制这一层的代价，它能做的只是**用 PatchFlag + Block Tree 尽量减少"要不要碰 DOM"这个判断的开销**，碰了之后多贵是浏览器引擎决定的。

---

## 九、完整链路总览（带包名标注）

```
App.vue
  ↓ @vue/compiler-sfc: parse SFC
<template> 字符串
  ↓ @vue/compiler-core: baseParse()
模板 AST（ELEMENT/INTERPOLATION/DIRECTIVE 节点）
  ↓ @vue/compiler-core: transform()
      ├─ hoistStatic：静态子树提升
      ├─ transformElement：计算 PatchFlag
      └─ Block 收集：拍平 dynamicChildren
带标记的 AST
  ↓ @vue/compiler-core: generate()
render 函数源码字符串
  ↓ Vite/@vitejs/plugin-vue 写入 bundle（或 new Function 运行时编译）
render(ctx) 函数
  ↓ @vue/runtime-core: componentUpdateFn 执行 render()
VNode 树（含 patchFlag / dynamicChildren）
  ↓ @vue/reactivity: track/trigger 驱动 effect 重新调度
  ↓ @vue/runtime-core: patch(n1, n2, container)
      ├─ patchElement：位运算读 PatchFlag，精准比对
      └─ patchBlockChildren：只遍历 dynamicChildren，跳过静态子树
最小 DOM 操作集合
  ↓ @vue/runtime-dom: nodeOps（textContent / setAttribute / insertBefore...）
真实 DOM 变化
  ↓ 浏览器引擎接手
Render Tree → Layout → Paint → Composite → Pixel
```

