```
/* ① 关键修复：改为 separate 模式 */
#tableNode {
  border-collapse: separate;   /* ← 核心改动，原来是 collapse */
  border-spacing: 0;           /* 保持单元格紧贴 */
  width: max-content;
}

/* ② 所有单元格只保留 右边框 + 下边框，避免 2px 重叠 */
#tableNode th,
#tableNode td {
  border: none;
  border-right: 1px solid #eee;
  border-bottom: 1px solid #eee;
  padding: 6px;
  max-width: 120px;
  text-align: left;
  background: #fff;
  white-space: normal;
  word-break: break-word;
}

/* ③ 补上左边框（只给第一列）和上边框（只给第一行） */
#tableNode th:first-child,
#tableNode td:first-child {
  border-left: 1px solid #eee;
}
#tableNode thead th {
  border-top: 1px solid transparent; /* thead 背景已有橙色，不需要额外上边框 */
}

/* ④ 固定表头行 */
#tableNode thead th {
  position: sticky;
  top: 0;
  background: var(--primary);
  color: #fff;
  z-index: 2;
  border-right-color: rgba(255,255,255,0.25);
  border-bottom-color: rgba(255,255,255,0.25);
}

/* ⑤ 固定第一列 —— 用 box-shadow 替代 border-right，防止滚动时消失 */
#tableNode tbody td:first-child,
#tableNode thead th:first-child {
  position: sticky;
  left: 0;
  z-index: 3;
  border-right: none;                          /* 取消 border-right */
  box-shadow: 2px 0 0 0 #eee;                 /* 用 shadow 模拟右边框，跟随 sticky 层 */
}

/* ⑥ 左上角交叉点：同时 sticky top + left，层级最高 */
#tableNode thead th:first-child {
  z-index: 4;
  box-shadow: 2px 0 0 0 rgba(255,255,255,0.25); /* 匹配橙色表头上的阴影颜色 */
}
```
