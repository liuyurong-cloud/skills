# CSS 编码规范

## 实现前必读（强制规则）

1. **所有选择器必须以页面 ID 开头** — `#{module}-list .xxx` 而不是 `.xxx`，防止样式污染其他页面
2. **BEM 命名，不要用裸 class** — `.list-item__header` 而不是 `.header`
3. **布局用 rem，微调用 px** — `font-size: 0.14rem; padding: 0.1rem;` 但 `border: 1px solid #eee;`
4. **对照 PC 端 visual hierarchy** — PC 端一行展示几个字段、如何对齐，移动端保持一致的信息层级

---

## 命名约定

所有样式 scoped 到页面 ID，用 BEM 风格：

```css
/* 列表页 */
#{module}-list { }
#{module}-list__navbar { }
#{module}-list .list-item { }
#{module}-list .list-item__header { }
#{module}-list .list-item__info { }
#{module}-list .list-item__actions { }
#{module}-list .screen-box { }
#{module}-list .screen-box__status { }
#{module}-list .add-btn { }

/* 编辑页 */
#{module}-edit { }
#{module}-edit__step-bar { }
#{module}-edit .step-item { }
#{module}-edit .step-item.active { }
#{module}-edit .form-section { }
#{module}-edit .form-item { }
#{module}-edit .form-item__label { }
#{module}-edit .line-card { }
#{module}-edit .line-card__header { }
#{module}-edit #lineSummary { }
#{module}-edit .add-line-wrapper { }

/* 详情页 */
#{module}-detail { }
#{module}-detail .detail-card { }
#{module}-detail .detail-card__header { }
#{module}-detail .detail-card__body { }
#{module}-detail .detail-card__item { }
#{module}-detail .detail-card__label { }
#{module}-detail .detail-card__value { }
#{module}-detail .detail-table-wrapper { }
#{module}-detail .detail-table { }
```

---

## 尺寸单位

全局 `reset.css` 设置了 `html { font-size: 312.5% }`，即 `1rem ≈ 100px`。

- 大尺寸用 rem：`font-size: 0.15rem; padding: 0.1rem;`
- 微调用 px：`border: 1px solid #eee;`
- 水平滚动表格用 `min-width` + rem：`min-width: 12rem;`

---

## 状态颜色（全局约定）

```css
.status-B  { color: #0079fd; }  /* 编辑中 - 蓝色 */
.status-C  { color: #67c23a; }  /* 已入账 - 绿色 */
.status-O,
.status-P  { color: #909399; }  /* 已关闭/已终止 - 灰色 */
.status-Q  { color: #f56c6c; }  /* 已冲销 - 红色 */
```

PC 端使用什么颜色，移动端保持一致。通常在 PC 端的 `statusMap` 或类似配置中定义。

---

## KimiSelect Shadow DOM 样式穿透

使用 `::part()` CSS 选择器突破 Shadow DOM：

```css
kimi-select::part(input-wrapper) {
  border: 1px solid #ddd;
  border-radius: 4px;
}
kimi-select::part(input) {
  font-size: 0.14rem;
  padding: 0.08rem;
}
kimi-select::part(suffix) {
  color: #999;
}
kimi-select::part(dropdown) {
  max-height: 3rem;
}
kimi-select::part(options) {
  max-height: 2.5rem;
}
kimi-select::part(option) {
  padding: 0.1rem;
  font-size: 0.14rem;
}
kimi-select::part(mask) {
  /* 遮罩层 */
}
```

可用的 part 名称：`input-wrapper`, `tags`, `input`, `selected-text`, `suffix`, `clear`, `mask`, `dropdown`, `header`, `clear-btn`, `title`, `confirm-btn`, `search-box`, `search-input`, `options`, `empty`

---

## 常用布局模式

### 详情卡片：边到边布局（参考 instock）

**重要：详情页卡片使用边到边（edge-to-edge）布局，不要用 margin/border-radius/box-shadow 做卡片悬浮效果。** 这是 instock 验证过的移动端最佳实践。

```css
#{module}-detail .detail-card {
  background: #fff;
  margin-bottom: 0.1rem;  /* 仅卡片之间用 rem 间距 */
  /* ❌ 不要用：margin: 0.1rem; border-radius: 8px; box-shadow: 0 2px 8px rgba(0,0,0,0.1); */
}
#{module}-detail .detail-card__header {
  padding: 0.1rem 0.15rem;
  font-size: 0.15rem;
  font-weight: bold;
  color: #333;
  border-bottom: 1px solid #f5f5f5;
}
#{module}-detail .detail-card__body {
  /* 内容区左右撑满，不设 margin */
}
#{module}-detail .detail-card__item {
  display: flex;
  padding: 0.1rem 0.15rem;
  border-bottom: 1px solid #fafafa;
}
```

### 固定底部栏（编辑页汇总/操作栏）

```css
#{module}-edit #lineSummary {
  position: fixed;
  bottom: 0;
  left: 0;
  right: 0;
  z-index: 999;
  background: #fff;
  box-shadow: 0 -2px 8px rgba(0, 0, 0, 0.1);
  padding: 0.1rem 0.15rem;
  display: flex;
  align-items: center;
  justify-content: space-between;
}
/* 页面底部需要 padding-bottom 防止内容被遮挡 */
#{module}-edit #step2Content {
  padding-bottom: 0.6rem;
}
```

### 水平滚动表格（详情页多列表格）

```css
#{module}-detail .detail-table-wrapper {
  overflow-x: auto;
  -webkit-overflow-scrolling: touch;
}
#{module}-detail .detail-table {
  min-width: 12rem;   /* 根据列数调整，每列约 1~1.5rem */
  white-space: nowrap;
  border-collapse: collapse;
}
#{module}-detail .detail-table th,
#{module}-detail .detail-table td {
  padding: 0.08rem 0.1rem;
  text-align: left;
  font-size: 0.13rem;
  border-bottom: 1px solid #eee;
}
```

### 固定顶部导航栏

```css
#{module}-list__navbar {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  z-index: 998;
  background: #fff;
  display: flex;
  align-items: center;
  padding: 0.08rem 0.12rem;
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.08);
}
#{module}-list #list-content {
  padding-top: 0.5rem;  /* 避开导航栏 */
}
```

### 表单项布局（label + value 左右排列）

```css
#{module}-edit .form-item {
  display: flex;
  align-items: center;
  padding: 0.1rem 0.15rem;
  border-bottom: 1px solid #f5f5f5;
}
#{module}-edit .form-item__label {
  width: 0.8rem;
  font-size: 0.14rem;
  color: #333;
  flex-shrink: 0;
}
#{module}-edit .form-item.required .form-item__label::before {
  content: "*";
  color: #f56c6c;
  margin-right: 2px;
}
#{module}-edit .form-item kimi-select {
  flex: 1;
}
```

### 步骤条

```css
#{module}-edit__step-bar {
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 0.12rem 0;
  background: #fff;
}
#{module}-edit .step-item {
  font-size: 0.14rem;
  color: #999;
}
#{module}-edit .step-item.active {
  color: #0079fd;
  font-weight: bold;
}
#{module}-edit .step-arrow {
  margin: 0 0.2rem;
  color: #ccc;
}
```

---

## 常见错误

| 错误 | 为什么错了 | 正确做法 |
|------|-----------|----------|
| `.header { ... }` | 会污染所有 JQM 页面 | `#{module}-list .list-item__header { }` |
| `.active { color: red; }` | 太通用，影响其他页面的 `.active` | `#{module}-edit .step-item.active { }` |
| `.btn-edit { ... }` | 多个模块都有名为 btn-edit 的按钮，样式会互相污染 | `#{module}-list .btn-edit { }` |
| `.form-row { ... }` | 无前缀 class 会跨页面泄漏，不同模块的表单行布局可能不同 | `#{module}-edit .form-row { }` |
| `width: 300px;` | 不同屏幕尺寸不一致 | `width: 3rem;`（≈300px） |
| `font-size: 14px;` | px 不随 rem 基准缩放 | `font-size: 0.14rem;` |
| 忘记底部 padding | 固定底部栏遮挡最后一行内容 | 给内容区加 `padding-bottom: 0.6rem;` |
| 表格列挤在一起 | 没有设置 min-width | `min-width: 12rem;`（根据列数） |
| `kimi-select .input { }` | Shadow DOM 穿透不生效 | `kimi-select::part(input) { }` |
| 详情页样式写在 detail.html | JQM 不解析 detail.html 的 head | 在 list.html 中加载 detail.css |

---

## CSS 审查清单（写完后逐项检查）

- [ ] 每个 CSS 文件第一行都是页面 ID 选择器（`#{module}-list` / `#{module}-edit` / `#{module}-detail`）
- [ ] 所有选择器都以页面 ID 开头（或嵌套在其下）
- [ ] 没有裸的通用 class 选择器（如 `.header`、`.active`、`.item`）
- [ ] 布局尺寸使用 rem（不是 px）
- [ ] 固定定位元素有对应的 padding 防止遮挡
- [ ] 状态颜色（B/C/O/P/Q）与 PC 端一致
- [ ] 水平滚动表格配置了 `min-width` 和 `overflow-x: auto`
- [ ] KimiSelect 样式使用 `::part()` 穿透
- [ ] 表单 label 宽度一致（通常 `0.8rem`）
- [ ] 没有使用 `!important`（除非覆盖 JQM 默认）
- [ ] detail.css 在 list.html 中加载（不在 detail.html 中）
