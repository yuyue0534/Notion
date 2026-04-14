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
