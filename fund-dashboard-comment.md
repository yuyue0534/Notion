```
yAxis: {
  type: 'value',
  min: value => {
    const range = value.max - value.min;
    const result = value.min - range * 0.3;
    // 根据数据范围动态取整，避免出现小数或超长数字
    const magnitude = Math.pow(10, Math.floor(Math.log10(Math.abs(range) || 1)));
    return Math.floor(result / magnitude) * magnitude;
  },
  max: value => {
    const range = value.max - value.min;
    const result = value.max + range * 0.3;
    const magnitude = Math.pow(10, Math.floor(Math.log10(Math.abs(range) || 1)));
    return Math.ceil(result / magnitude) * magnitude;
  },
  axisLine: {
    show: true,
    lineStyle: { color: '#ccc' }
  },
  // 格式化标签，超过一定长度时用 K/M 缩写
  axisLabel: {
    formatter: value => {
      if (Math.abs(value) >= 1_000_000) return (value / 1_000_000).toFixed(1) + 'M';
      if (Math.abs(value) >= 1_000) return (value / 1_000).toFixed(1) + 'K';
      return value;
    }
  }
},
```

---

```
#netconnode {
  appearance: none;
  width: 16px;
  height: 16px;
  border: 2px solid #ccc;
  border-radius: 3px;
  cursor: pointer;
}

#netconnode:checked {
  background-color: #4A90E2;
  border-color: #4A90E2;
}

#netconnode:checked::after {
  content: '';
  display: block;
  width: 4px;
  height: 8px;
  border: 2px solid #fff; /* 勾的颜色 */
  border-top: none;
  border-left: none;
  transform: rotate(45deg) translate(2px, -1px);
}
```

```
const configs = {
  '1M': { count: 30, labels: Array.from({ length: 30 }, (_, i) => `${i + 1}-Jan`) },
  '3M': { count: 12, labels: ["1-Jan","8-Jan","16-Jan","24-Jan","1-Feb","9-Feb","17-Feb","25-Feb","4-Mar","12-Mar","20-Mar","28-Mar"] },
  '6M': { count: 12, labels: ["1-Jan","16-Jan","31-Jan","15-Feb","2-Mar","17-Mar","1-Apr","16-Apr","1-May","16-May","31-May","15-Jun"] },
  '1Y': { count: 12, labels: ["1-Jan","1-Feb","1-Mar","1-Apr","1-May","1-Jun","1-Jul","1-Aug","1-Sep","1-Oct","1-Nov","1-Dec"] },
  '3Y': { count: 12, labels: ["1-Jan","1-Apr","1-Jul","1-Oct","1-Jan","1-Apr","1-Jul","1-Oct","1-Jan","1-Apr","1-Jul","1-Oct"] },
};
```
