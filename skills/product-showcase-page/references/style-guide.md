# 视觉设计规范

## 色彩体系

### 淡雅绿主题（默认）

| 用途 | 色值 |
|------|------|
| 头部渐变起始 | `#a8d5a2` |
| 头部渐变中间 | `#7bc67e` |
| 头部渐变终止 | `#5ab55e` |
| 按钮/链接 | `#4a9e4e` |
| 关键词标签背景 | `#e8f5e9` |
| 关键词标签文字 | `#2e7d32` |
| 页面背景 | `#f5f7f5` |
| 卡片hover阴影 | `rgba(74,158,78,0.12)` |
| 筛选按钮边框 | `#c8dcc8` |

### 字号体系

| 元素 | 字号 |
|------|------|
| 页面标题 | 26px，letter-spacing 4px |
| 筛选按钮 | 13px |
| 商品名称 | 13px，font-weight 600 |
| 关键词标签 | 9px |
| 差异点文本 | 10px |
| 店铺badge | 9px |
| 查看商品按钮 | 10px |
| 手卡标签 | 9px |

### 卡片规范

- 圆角：10px
- 阴影：`0 1px 6px rgba(0,0,0,0.06)`
- hover上浮：`translateY(-3px)` + 加强阴影
- 网格间距：12px
- 网格最大宽：1200px
- 卡片最小宽：220px（auto-fill）

### 产品图

- 比例：1:1 正方形（`padding-top: 100%`）
- 裁切：`object-fit: cover`
- 店铺badge：左下角，半透明黑底白字，`backdrop-filter: blur(4px)`

### 手卡图

- 宽度：100%
- 圆角：6px
- 边框：`1px solid #eee`
- 加载：`loading="lazy"`
- 交互：点击放大（`window.open(this.src)`）

### 移动端适配

```css
@media (max-width: 600px) {
  .grid { grid-template-columns: repeat(2, 1fr); gap: 8px; padding: 8px; }
  .header h1 { font-size: 20px; }
  .filter-btn { font-size: 11px; padding: 5px 14px; }
}
```

## 配色方案替换

如需换色调，替换以下CSS位置：
1. `.header` → `background: linear-gradient(...)`
2. `.filter-btn.active` → `background: linear-gradient(...)`
3. `.tag` → `background` 和 `color`
4. `.btn` → `background: linear-gradient(...)`
5. `.card:hover` → `box-shadow` 颜色
6. `body` → `background`
7. `.filter-btn:hover` → `border-color` 和 `color`
