# 工具 & 组件速查

## 工具函数（tool.js）

| 函数 | 用途 | 用法 |
|------|------|------|
| `$.request(url, method, data, success, error)` | AJAX 请求 | 自动带 `Access-Token-PC` header，601→login，自动管理 `$.me.locked` |
| `$.myAlert(message)` | 提示弹窗 | 或 `$.myAlert({title, message, callback})` |
| `$.myConfirm({title, message, callback})` | 确认弹窗 | callback 在用户点"确定"时执行 |
| `$.myToast(message)` | Toast | 居中，1秒自动消失 |
| `$.myLoadding(message?)` | 加载遮罩 | `$.request` 完成后自动关闭 |
| `$.getUrlParam(url, name)` | URL 参数 | 如 `$.getUrlParam(url, "id")` |
| `$.isEmpty(val)` | 判空 | null / undefined / 全空格 → true |
| `$.formatMoney(value)` | 格式化金额 | `1234.5` → `"1,234.50"` |
| `$.showFilter()` | 显示筛选面板 | 显示 `.screen-box` |
| `$.closeFilter()` | 关闭筛选面板 | 隐藏 `.screen-box` |
| `$.addVerToUrl(url)` | 加版本参数 | 从当前 URL 取 `v` 参数追加 |
| `$.selectDate(el, config, callback)` | 日期选择 | config: `{start, end, select, title}`，callback 得 `{year, month, day}` |
| `$.selectDateSimple(el, config, callback)` | 年月选择 | 同上但只有 year+month |

---

## KimiSelect Web Component

**规则：编辑表单中的所有下拉选择（包括税率、币种等简单选项）都必须使用 `<kimi-select>`，禁止使用原生 `<select>` 元素。** 原因：原生 `<select>` 在不同移动浏览器上表现不一致，且无法参与 Shadow DOM 样式穿透和统一的事件模型。

### HTML 属性

| 属性 | 类型 | 说明 |
|------|------|------|
| `placeholder` | string | 占位文字 |
| `title` | string | 弹窗标题 |
| `searchable` | boolean | 显示搜索框 |
| `clearable` | boolean | 显示清除按钮 |
| `multiple` | boolean | 多选模式 |
| `disabled` | boolean | 禁用状态 |
| `notrigger` | boolean | 隐藏输入框（由外部触发 open） |
| `direction` | string | `"bottom"`（默认）或 `"top"` |
| `value` | string/array | 当前选中值 |
| `value-field` | string | 选项值字段名，默认 `"value"`。**必须与 API 返回的字段名一致**，如 API 返回 `productId` 则设为 `"productId"` |
| `label-field` | string | 选项标签字段名，默认 `"label"`。**必须与 API 返回的字段名一致**，如 API 返回 `productName` 则设为 `"productName"` |

### JS API

```js
var el = $("#mySelect")[0];  // 获取 DOM 元素（不是 jQuery 对象）

// 选项
el.options = [{ value: "1", label: "选项1", code: "C001", spec: "..." }, ...];
var opts = el.getOptions();

// 值
el.value = "1";                // 设置
var val = el.value;            // 获取（单选=string，多选=array）
var selected = el.selectedOptions; // 完整选项对象（多选时是数组）

// 操作
el.open();
el.close();
el.clear();
el.setDisabled(true);
el.focus();
el.blur();
```

### 事件

```js
// change — 最重要的使用方式
$(document).on("change", "#mySelect", function (e) {
  var detail = e.originalEvent.detail;
  detail.value;    // 选中值（单选=string，多选=array）
  detail.option;   // 完整选项对象（仅单选）
  detail.options;  // 完整选项数组（仅多选）
  detail.label;    // 选中标签文字
});

// 其他事件
$(document).on("open", "#mySelect", fn);
$(document).on("close", "#mySelect", fn);
$(document).on("clear", "#mySelect", fn);
```

### CSS 穿透

```css
kimi-select::part(input-wrapper) { }
kimi-select::part(input) { }
kimi-select::part(suffix) { }
kimi-select::part(clear) { }
kimi-select::part(mask) { }
kimi-select::part(dropdown) { }
kimi-select::part(header) { }
kimi-select::part(clear-btn) { }
kimi-select::part(title) { }
kimi-select::part(confirm-btn) { }
kimi-select::part(search-box) { }
kimi-select::part(search-input) { }
kimi-select::part(options) { }
kimi-select::part(option) { }
kimi-select::part(empty) { }
```

---

## UdiScanner Web Component

```html
<udi-scanner id="udiScanner" label="扫描"></udi-scanner>
```

```js
// 扫码成功
$(document).on("scan-success", "#udiScanner", function (e) {
  var product = e.originalEvent.detail;
  // product.productId, product.productName, product.productCode,
  // product.batchNo, product.productionDate, product.expirationDate,
  // product.recordNo, product.spec, product.unit, ...
});

// 扫码失败
$(document).on("scan-error", "#udiScanner", function (e) {
  var error = e.originalEvent.detail;
});

// 手动触发
$("#udiScanner")[0].scan();
```

---

## MobileSelect（仅筛选面板用）

```js
var ms = new MobileSelect({
  trigger: "#triggerXx",
  wheels: [{ data: [{ id: "", value: "全部" }] }],
  callback: function (indexArr, data) {
    // data[0].id, data[0].value
  }
});

// 动态更新选项
ms.updateWheel([{ data: newData }]);
```

---

## Vue → jQuery 完整映射

| PC 端（Vue） | 移动端（jQuery） |
|---|---|
| `v-model="form.xxx"` | 手动 `$("#xxx").val()` + change 事件同步 |
| `<el-select v-model="form.xxId">` | `<kimi-select id="xxSelect">` + `e.originalEvent.detail.value` |
| `<el-date-picker v-model="form.date">` | `$.selectDate("#dateInput", {}, fn)` |
| `this.$refs.xxx` | `$("#xxx")` |
| `this.$route.query.id` | `$.getUrlParam(url, "id")` |
| `axios.post(url, data)` | `$.request(window.apiUrl + url, "POST", data, callback)` |
| `this.$message.success(msg)` | `$.myToast(msg)` |
| `this.$confirm(msg).then(...)` | `$.myConfirm({title, message, callback})` |
| `v-for="item in list" :key="item.id"` | `arr.forEach(fn)` 拼接 HTML 字符串 + `$().append()` |
| `v-if / v-show` | `$().show()/hide()` 或 CSS `display:none` |
| `computed: { total() { return ... } }` | `$.me.updateTotal()` 普通函数，数据变化后手动调 |
| `watch: { ccId(val) { ... } }` | `<kimi-select>` change 事件回调中处理 |
| `mounted() { ... }` | `$.me.init()` |
| `data() { return { x: 1 } }` | `$.me = { x: 1, ... }` 顶层属性 |
| `methods: { foo() {} }` | `$.me = { foo: function() {} }` |
| `this.$router.push("/path")` | `location.href = $.addVerToUrl("list.html")` |
| `this.$router.go(-1)` | `history.back()` |
| `localStorage.getItem("token")` | `$.request` 自动处理，不用手动 |
| `<el-table :data="list">` | 手动 `for` 循环拼 HTML `<table>` 或 `<ul>` |
| `<el-input-number v-model="num">` | `<input type="number">` + input 事件 + decimal.js 计算 |
| `el-select remote-method` | kimi-select 不支持远程搜索，用 `searchable` 本地过滤 |
| `this.$set(obj, key, val)` | 直接赋值 `obj[key] = val`（无响应式系统） |
| `<el-steps :active="step">` | 手动 div class `active` 切换 |
| `<el-form-item label="名称">` | `<div class="form-item"><span class="label"><span class="value">` |
| `el-form rules validation` | 手动 `if (!field) { $.myToast("..."); return false; }` |
| `<el-tag>` | `<span class="status-B">编辑中</span>` |
