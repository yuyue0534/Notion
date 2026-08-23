# 浏览器渲染全流程详解

这是一个从字节流到屏幕像素的多阶段管线，涉及好几个独立又相互关联的"树"结构。分层讲一下。

## 一、整体管线概览

```
HTML字节流 → DOM树
CSS字节流 → CSSOM树
              ↓
         Render Tree（渲染树）
              ↓
           Layout（布局/回流）
              ↓
            Paint（绘制）
              ↓
         Composite（合成）
              ↓
            屏幕像素
```

同时，JS 引擎有自己独立的一套解析管线，会在特定时机打断/参与上面这条链。

---

## 二、HTML 解析 → DOM 树

浏览器的 HTML 解析器不是简单的 XML/树解析，而是一个**状态机**（tokenizer + tree construction，遵循 WHATWG HTML5 规范）：

1. **Tokenization（分词）**：字节流 → 字符流（按编码解码）→ Token（如 `StartTag`、`EndTag`、`Character`、`Comment`）
2. **Tree Construction（建树）**：根据 token 类型和当前"插入模式"（insertion mode，如 `in head`、`in body`），维护一个**开放元素栈**，把 token 转成 DOM 节点插入树中

关键点：HTML 解析是**容错的**——标签写错、没闭合、嵌套错误，浏览器都有一套明确规则去"纠正"（比如 `<table>` 里塞非法内容会被移到 table 外面）。这也是为什么"手写不规范 HTML 也能正常显示"。

**特别的中断点**：解析到 `<script>` 标签时，如果没有 `async`/`defer`，解析器会**同步阻塞**，等 JS 下载并执行完（因为 JS 可能用 `document.write` 修改后续 DOM）。这就是"JS 阻塞 HTML 解析"的由来。

---

## 三、CSS 解析 → CSSOM 树

CSS 解析相对简单很多，因为 CSS 语法本身规整（Selector + Declaration Block）：

1. Tokenizer 把 CSS 文本切成 token（selector、property、value…）
2. 按规则建立 **CSSOM（CSS Object Model）**——一棵和 DOM 结构类似但独立的树，每个节点带有计算出的样式规则

CSSOM 是**渲染阻塞**的：浏览器必须拿到完整的 CSSOM 才能开始构建渲染树，因为任何一条后加载的 CSS 规则都可能覆盖前面的样式（级联特性决定了不能"边解析边渲染"）。这也是为什么 CSS 要放 `<head>` 里尽早加载。

---

## 四、JS 解析：这里才是真正的 AST

你提到的 AST（抽象语法树）主要在 **JS 引擎**（V8、SpiderMonkey、JavaScriptCore）里，跟 DOM/CSSOM 是完全不同的概念，别混了：

1. **Scanner/Lexer（词法分析）**：源码字符流 → Token 流（关键字、标识符、运算符…）
2. **Parser（语法分析）**：Token 流 → **AST**（抽象语法树），每个节点代表一种语法结构（`VariableDeclaration`、`FunctionExpression`、`BinaryExpression`…）
   - V8 用的是**预解析（Pre-parsing）+ 完整解析（Full parsing）**两阶段策略：先快速扫一遍只找变量作用域和语法错误（不生成完整 AST），真正要执行的函数才做完整解析生成 AST，来减少启动时间
3. **Ignition（V8 的解释器）**：AST → **字节码（Bytecode）**
4. **TurboFan（V8 的优化编译器）**：热点字节码 → 通过内联缓存（Inline Cache）收集类型信息 → 编译为**优化的机器码**；如果运行时假设被打破（比如变量类型突变），会"去优化"（Deoptimization）退回字节码

JS 的 AST 你可以用 [AST Explorer](https://astexplorer.net/) 直观看到——比如 `const x = 1 + 2` 会被解析成一棵 `VariableDeclarator` 下挂 `BinaryExpression` 的树。

**JS 和渲染的关系**：JS 执行时可以同步读写 DOM/CSSOM（比如 `getComputedStyle`），所以浏览器必须保证 DOM/CSSOM 在 JS 访问时是"确定"的，这也是脚本阻塞的另一个原因（避免时序竞争）。

---

## 五、Render Tree（渲染树）

DOM 树 + CSSOM 树 → **Render Tree**：

- 只包含**需要显示**的节点（`display: none` 的节点不进入渲染树；但 `visibility: hidden` 会进入，只是不可见，仍占位）
- `<head>`、`<script>` 等不可见标签也不进入
- 每个渲染树节点（在 Chromium 里叫 `LayoutObject`）都关联了它的最终计算样式

---

## 六、Layout（布局，又叫 Reflow 回流）

遍历渲染树，计算每个节点在视口中的**精确几何位置和尺寸**（x, y, width, height）。

- 这一步是**几何敏感**的：改变宽高、字体大小、增删 DOM 节点，都会触发重新布局
- 现代浏览器（如 Chromium 的 **LayoutNG**）用的是更精细化、支持约束传播的布局算法，取代早期简单的盒模型递归
- 布局是全局或局部的：改一个节点可能只需要局部重排（如果不影响兄弟/父节点），也可能触发整棵树重排（"回流地狱"的来源，频繁读写布局属性如 `offsetHeight` 会强制同步回流）

---

## 七、Paint（绘制）

把每个渲染树节点转换成实际的**绘制指令**（画背景、画边框、画文字、画阴影……），生成一系列"绘制记录"（Display List，类似录制下来的绘图命令）。

这一步还没有真正往屏幕上画像素，只是记录"该怎么画"。

---

## 八、Composite（合成）

这是现代浏览器提速的关键：

1. 把页面切分成若干 **合成层（Compositing Layers）**——通常触发条件包括 `transform`、`opacity`、`will-change`、`<video>`、`position: fixed` 等
2. 每层单独在 **GPU** 上栅格化（Rasterization：把绘制指令变成实际的位图像素）
3. 合成线程（Compositor Thread，与主线程分离）把这些层按顺序叠加合成最终画面

**为什么这很重要**：如果动画只涉及 `transform`/`opacity`，浏览器可以**跳过 Layout 和 Paint**，直接在合成线程重新合成图层——这就是"只用 transform 做动画更流畅"的根本原因，因为完全绕开了主线程的布局/绘制开销。

---

## 九、关键路径总结（Critical Rendering Path）

```
HTML → DOM ┐
           ├→ Render Tree → Layout → Paint → Composite → 屏幕
CSS → CSSOM┘
JS 可修改 DOM/CSSOM，可能触发上面任意阶段重做
```

浏览器优化的核心思路就是：**尽量减少重新触发 Layout（最贵）、其次是 Paint（次贵）、优先用 Composite-only 的属性变化（最便宜）**。

---

