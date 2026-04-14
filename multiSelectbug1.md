## 问题分析

`setValue` 失效的根本原因是 **`renderTags` 里把 `this.input.innerHTML = ''` 清空了，但 `this.placeholder` 是提前创建好的 DOM 节点，被移走后再 `appendChild` 没问题——真正的 bug 在于 `dropdown` 里的 checkbox 被 `render()` 绑定了 `onchange`，但 `setValue` 是直接用 JS 设置 `input.checked`，不会触发 `onchange` 事件**。

不过看你的代码，`setValue` 末尾确实调用了 `this.renderTags()`，所以这条路是通的。

**真正的 bug 是：`setValue` 调用时 `this.dropdown` 里根本查不到任何 checkbox。**

原因：`render()` 里用的是：
```js
this.el = document.querySelector(el); // el = '#schemeNode1'
```
但 `schemeOptionsList` 是外部变量，如果在 `new MultiSelect(...)` 执行时 `schemeOptionsList` 还是空数组或 undefined，`forEach` 就不会生成任何 `<input>`，后续 `setValue` 自然无效。

---

## 核心 Bug + 修复

有两处需要修复：

**Bug 1**：`options` 列表为空或异步加载时，`render()` 时没有 checkbox 可渲染。需要支持动态更新 options。

**Bug 2**：`setValue` 里 `querySelectorAll` 时序没问题，但缺少对 `options` 未就绪的保护。

```diff
  render() {
    this.el.innerHTML = `
      <div class="multi-select">
        <div class="select-input"></div>
        <div class="clear-btn">✕</div>
        <div class="dropdown"></div>
      </div>
    `;
    this.input = this.el.querySelector('.select-input');
    this.clearBtn = this.el.querySelector('.clear-btn');
    this.dropdown = this.el.querySelector('.dropdown');
    this.placeholder = document.createElement('span');
    this.placeholder.className = 'placeholder';
    this.placeholder.textContent = this.options.placeholder;
    this.input.appendChild(this.placeholder);
-   this.options.options.forEach(opt => {
+   (this.options.options || []).forEach(opt => {
      const label = document.createElement('label');
      label.className = 'option';
      label.innerHTML = `<input type="checkbox" value="${opt.value}"> ${opt.label}`;
      this.dropdown.appendChild(label);
    });
+   // 每次渲染 options 后重新绑定 checkbox 的 onChange
+   this._bindCheckboxes();
  }

+ // 新增：单独绑定 checkbox，供 render 和 setOptions 复用
+ _bindCheckboxes() {
+   this.dropdown.querySelectorAll('input').forEach(cb => {
+     cb.onchange = () => this.renderTags();
+   });
+ }

+ // 新增：支持动态更新 options 列表（解决 schemeOptionsList 异步赋值问题）
+ setOptions(optionsList) {
+   this.options.options = optionsList;
+   this.dropdown.innerHTML = '';
+   optionsList.forEach(opt => {
+     const label = document.createElement('label');
+     label.className = 'option';
+     label.innerHTML = `<input type="checkbox" value="${opt.value}"> ${opt.label}`;
+     this.dropdown.appendChild(label);
+   });
+   this._bindCheckboxes();
+   this.renderTags();
+ }

  bind() {
    this.input.onclick = () => {
      this.dropdown.classList.toggle('open');
    };
    this.clearBtn.onclick = (e) => {
      e.stopPropagation();
      this.dropdown.querySelectorAll('input').forEach(i => i.checked = false);
      this.renderTags();
    };
-   this.dropdown.querySelectorAll('input').forEach(cb => {
-     cb.onchange = () => this.renderTags();
-   });
+   this._bindCheckboxes();
    document.addEventListener('click', (e) => {
      if (!this.el.contains(e.target)) {
        this.dropdown.classList.remove('open');
      }
    });
  }
```

---

## 使用方式

如果 `schemeOptionsList` 是异步获取的，改成：

```js
// 初始化组件（options 可为空）
const schemeNode1Select = new MultiSelect('#schemeNode1', {
  placeholder: 'select scheme',
  options: [],
  onChange: (v) => { console.log('account-Select:', v) }
});

// 异步数据回来后再 setOptions
fetchSchemeOptions().then(list => {
  schemeNode1Select.setOptions(list);
});
```

这样 `searchTable1` 里调用 `setValue(['ALL'])` 就能正常生效了。

---

## 需求分析

两个独立约束需要同时满足：

1. **单个 tag 截断**：每个 tag 文本宽度限制 40px，超出用 `...` 省略
2. **容器行数限制**：选中 >3 项时，`.select-input` 最多展示两行，溢出隐藏（不是折叠 tag，是容器截断）

关键点：这两个约束叠加后，tag 本身已经很窄（40px 文本区），两行能容纳的 tag 数量是有限的，超出两行的 tag 视觉上不可见但 DOM 里仍存在，`getValue()` 逻辑不受影响。

---

## Diff

**CSS 部分：**

```diff
 .select-input {
   border: 1px solid var(--border);
   border-radius: var(--radius);
   padding: 6px 36px 6px 8px;
   display: flex;
   flex-wrap: wrap;
   gap: 6px;
   cursor: pointer;
+  /* 单 tag 截断依赖此上下文 */
+  min-height: 34px;
 }

+/* 选中 >3 项时，容器限制为两行 */
+.select-input.overflow-2line {
+  max-height: calc(34px * 2 + 6px + 12px); /* 两行tag高度 + gap + padding */
+  overflow: hidden;
+}

 .tag {
   background: var(--primary);
   color: #fff;
   border-radius: 4px;
   padding: 2px 6px;
   font-size: 12px;
   display: flex;
   align-items: center;
+  max-width: calc(40px + 6px + 4px + 1em); /* 文本40px + padding + gap + ✕ */
+  min-width: 0;  /* 允许flex子项收缩 */
 }

+.tag-text {
+  display: inline-block;
+  max-width: 40px;
+  overflow: hidden;
+  text-overflow: ellipsis;
+  white-space: nowrap;
+  vertical-align: middle;
+}
```

**JS 部分（`renderTags` 方法内）：**

```diff
 renderTags() {
   this.input.innerHTML = '';
   const checked = [...this.dropdown.querySelectorAll('input')].filter(i => i.checked);

   if (!checked.length) {
     this.input.appendChild(this.placeholder);
     this.clearBtn.style.display = 'none';
+    this.input.classList.remove('overflow-2line');
   } else {
     this.clearBtn.style.display = 'block';
+    this.input.classList.toggle('overflow-2line', checked.length > 3);
     checked.forEach(cb => {
       const tag = document.createElement('div');
       tag.className = 'tag';
-      tag.innerHTML = `${cb.value}<span>✕</span>`;
+      tag.innerHTML = `<span class="tag-text" title="${cb.value}">${cb.value}</span><span>✕</span>`;
       tag.querySelector('span').onclick = (e) => {
```

---

## 说明

- `tag-text` 加了 `title="${cb.value}"` 属性，移动端长按可以看到完整文本（部分浏览器支持），桌面端 hover tooltip 也生效
- `overflow-2line` 的 `max-height` 用 `calc` 精确计算：单行 tag 高度（`34px` = `font-size:12px` + `padding:2px*2` + `border` + 行高）× 2 行 + 1 个 gap(`6px`) + 容器上下 padding(`12px`)，避免硬编码出现半截 tag 的情况
- `>3` 的判断在 `renderTags` 里动态加减 class，不影响 `getValue()` 返回值，选中状态完整保留
