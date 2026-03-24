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
