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


---

## 分析

固定前两列的核心方案：外层容器 `overflow-x: auto`，固定列用 `position: sticky; left: Npx`，关键是两列的 `left` 值要精确对齐，且固定列需要 `z-index` 压住滚动列，背景色也要显式设置否则透明会穿透。

列宽规划：
- 第1列（checkbox）：`32px`
- 第2列（Fund）：`110px`，`left: 32px`

---

## Diff

**CSS 部分：**

```diff
+/* ── 固定列表格容器 ── */
+.report_table_wrap {
+  overflow-x: auto;
+  -webkit-overflow-scrolling: touch;
+  border-radius: var(--radius);
+}

 .report_table >thead th:nth-child(2) { min-width: 80px;}
 .report_table >thead th:nth-child(3) { min-width: 80px;}
 .report_table >thead th:nth-child(5) { min-width: 55px;}
 .report_table >thead th:nth-child(6) { min-width: 45px;}
 .report_table >thead th:nth-child(7) { min-width: 45px;}
 .report_table >thead th:nth-child(8) { min-width: 45px;}

+/* 固定列通用 */
+.report_table th:nth-child(1),
+.report_table td:nth-child(1),
+.report_table th:nth-child(2),
+.report_table td:nth-child(2) {
+  position: sticky;
+  z-index: 2;
+  background: #fff;  /* 防透明穿透 */
+}
+#reportTableBody > tr:nth-child(2n+1) td:nth-child(1),
+#reportTableBody > tr:nth-child(2n+1) td:nth-child(2) {
+  background: #FFE4C4; /* 同斑马纹色 */
+}
+
+/* 第1列：checkbox，宽32px，left:0 */
+.report_table th:nth-child(1),
+.report_table td:nth-child(1) {
+  left: 0;
+  width: 32px;
+  min-width: 32px;
+  max-width: 32px;
+}
+
+/* 第2列：Fund，left = 第1列宽度 32px */
+.report_table th:nth-child(2),
+.report_table td:nth-child(2) {
+  left: 32px;
+  min-width: 110px;
+  max-width: 110px;
+  /* 右侧加一条分隔线，区分固定区与滚动区 */
+  box-shadow: 2px 0 4px -1px rgba(0,0,0,0.08);
+}
+
+/* thead 固定列 z-index 要高于 tbody，避免内容滚动时压过表头 */
+.report_table thead th:nth-child(1),
+.report_table thead th:nth-child(2) {
+  z-index: 3;
+}
+
+/* Fund 列文字截断，移动端列宽有限 */
+.report_table td:nth-child(2) {
+  overflow: hidden;
+  text-overflow: ellipsis;
+  white-space: nowrap;
+}
```

**HTML 部分（给表格加外层容器）：**

```diff
-<div class="table-scroll-wrap">
-  <table class="report_table">
+<div class="table-scroll-wrap">
+  <div class="report_table_wrap">
+  <table class="report_table" style="table-layout: fixed;">
     <thead>
       <tr>
         <th></th>
         <th>Fund</th>
         <th>Fund Type</th>
         <th>Scheme</th>
         <th>Risk</th>
         <th id="date1_price"></th>
         <th id="date2_price"></th>
         <th id="date3_price"></th>
       </tr>
     </thead>
     <tbody id="reportTableBody"></tbody>
   </table>
+  </div>
 </div>
```

---

## 注意事项

- `table-layout: fixed` 配合 `sticky` 在移动端 WebKit 兼容性更好，列宽以 CSS 定义为准不会被内容撑开
- 斑马纹的奇数行背景 `#FFE4C4` 需要在固定列 td 上显式重声明，否则 `sticky` 脱离普通流后背景透明会露出滚动内容
- `table-scroll-wrap` 已有 `overflow-y: auto`，外层不变；新增的 `report_table_wrap` 只负责横向滚动，两层职责分离
