# 如何高效阅读 PC 端 Vue 代码

阅读 PC 端代码是迁移的第一步也是最关键的一步。以下方法能帮你快速提取有效信息，避免被 Vue 模板代码干扰。

---

## 最重要的原则：按视图提取，不是按文件阅读

不要从头到尾读 Vue 文件。而是带着三个问题去搜索：

1. **列表页展示了哪些字段？**（顺序是什么？）
2. **详情页展示了哪些字段？**（怎么分组的？）
3. **编辑表单有哪些字段？**（哪些必填？哪些有条件？）

答案必须精确到字段名、字段顺序、显示标签。这是后续所有工作的基础。

---

## 一、提取列表视图字段

### 找到列表表格定义

搜索关键字：`<el-table`、`<el-table-column`、`<th`、`columns`

```html
<!-- 最常见的模式 -->
<el-table :data="tableData">
  <el-table-column prop="code" label="单号" width="150" />
  <el-table-column prop="status" label="状态">
    <template slot-scope="scope">
      {{ statusMap[scope.row.status] }}
    </template>
  </el-table-column>
  <el-table-column prop="ccName" label="主体" />
  <el-table-column prop="dateOrder" label="单据日期" />
</el-table>
```

**你要记录的是**：

| 序号 | prop (字段名) | label (显示名) | 有二次处理？ |
|------|---------------|----------------|-------------|
| 1 | code | 单号 | 无 |
| 2 | status | 状态 | statusMap映射 |
| 3 | ccName | 主体 | 无 |
| 4 | dateOrder | 单据日期 | 可能截取日期 |

### 找到操作按钮

搜索 `<el-table-column label="操作">` 或包含 `@click` 的按钮：

```html
<el-table-column label="操作">
  <template slot-scope="scope">
    <el-button v-if="scope.row.status === 'B'" @click="handlePost(scope.row)">入账</el-button>
    <el-button v-if="scope.row.status === 'B'" @click="handleEdit(scope.row)">编辑</el-button>
  </template>
</el-table-column>
```

**记录每个按钮的显示条件**（`v-if` 的值）和触发的方法名。

### 找到筛选条件

搜索 `searchForm`、`filterForm`、`queryParams`：

```html
<el-form :model="searchForm">
  <el-form-item label="主体">
    <el-select v-model="searchForm.ccId" />
  </el-form-item>
  <el-form-item label="状态">
    <el-select v-model="searchForm.status" />
  </el-form-item>
  <el-form-item label="日期">
    <el-date-picker v-model="searchForm.dateRange" />
  </el-form-item>
</el-form>
```

**记录每个筛选字段**：字段名（`v-model` 去掉前缀）、组件类型、默认值。

---

## 二、提取详情视图字段

### 找到详情展示

搜索包含 `detail`、`view`、`info` 的组件或 template：

```html
<!-- 可能是 el-descriptions -->
<el-descriptions title="基本信息">
  <el-descriptions-item label="单号">{{ detail.code }}</el-descriptions-item>
  <el-descriptions-item label="状态">{{ statusMap[detail.status] }}</el-descriptions-item>
</el-descriptions>

<!-- 也可能是普通 div -->
<div class="detail-info">
  <div class="info-item">
    <label>单号</label><span>{{ detail.code }}</span>
  </div>
</div>

<!-- 弹窗中查看详情 -->
<el-dialog title="详情">
  <el-descriptions>...</el-descriptions>
  <el-table :data="detail.detailList">...</el-table>
</el-dialog>
```

**你要记录的是**：
- 每个字段的 API 字段名（`detail.xxx` 中的 `xxx`）
- 显示标签
- 分组（如果有多个 `<el-descriptions>` 或分区 div）
- 顺序

### 找到明细表格列

详情页的 `<el-table>` 通常在弹窗或详情组件中：

```html
<el-table :data="detail.detailList">
  <el-table-column prop="lineNo" label="序号" />
  <el-table-column prop="productName" label="商品名称" />
  <el-table-column prop="spec" label="规格型号" />
  ...
</el-table>
```

**记录所有列**：prop、label、顺序。数据来源是 `detail.detailList`。

---

## 三、提取编辑视图字段

### 找到编辑表单

搜索 `<el-form`、`<el-form-item`、`rules`：

```html
<el-form :model="form" :rules="rules">
  <el-form-item label="单据日期" prop="dateOrder">
    <el-date-picker v-model="form.dateOrder" />
  </el-form-item>
  <el-form-item label="主体" prop="ccId">
    <el-select v-model="form.ccId" @change="onCcChange" />
  </el-form-item>
</el-form>
```

**逐项记录**：

| prop | label | 必填？ | 默认值 | disabled条件 | 联动 |
|------|-------|--------|--------|-------------|------|
| dateOrder | 单据日期 | 是 | 当天 | - | - |
| ccId | 主体 | 是 | - | status !== 'B' | @change 重新加载商品列表 |

必填判断方法：
- `rules` 中 `prop` 对应的规则有 `required: true`
- 或模板中有 `:required="true"`

默认值查找：
```js
data() {
  return {
    form: {
      type: 31,        // ← 写死的常量
      isRed: 0,        // ← 写死的常量
      currencyId: 1,   // ← 写死的常量
      dateOrder: new Date(), // ← 当天
      status: "B",     // ← 默认状态
    }
  }
}
```

### 找到校验规则

搜索 `rules`、`validator`、`validate`：

```js
rules: {
  dateOrder: [{ required: true, message: "请选择单据日期" }],
  ccId: [{ required: true, message: "请选择主体" }],
}
```

以及方法中的手动校验：
```js
if (!this.form.ccId) {
  this.$message.error("请选择主体");
  return;
}
```

---

## 四、提取 API 调用

搜索关键字：`axios`、`fetch`、`this.$http`、`request(`

**每个 API 记录完整信息**（参考下方 API 提取模板）。

重点关注：
- URL 路径（精确到每个字符）
- 请求方法（GET/POST）
- 参数对象的结构和字段名（不能自己编）
- 响应数据的字段名和嵌套结构
- `.then()` 回调中是否有 `map`/`filter`/字段重命名等二次处理
- 哪些参数是条件性的（如选了日期范围才传 `dateOrderRange`）

---

## Vue 代码中的信号 vs 噪声

### 信号（必须移植的）

- `axios.post/get/request` 调用 — API 接口
- `<el-table-column prop="xxx">` — 列表字段名和顺序
- `<el-descriptions-item label="xxx">` — 详情字段名和顺序
- `<el-form-item prop="xxx" label="xxx">` — 编辑字段名、顺序、标签
- `rules: { xxx: [{ required: true }] }` — 校验规则
- `v-if="status === 'B'"` — 条件渲染
- `:disabled="status !== 'B'"` — 禁用条件
- `new Decimal(a).mul(b)` — 计算逻辑
- `this.$confirm(...)` — 用户确认
- `data() { return { form: { type: 31 } } }` — 写死的常量
- `statusMap` / `statusOptions` — 状态枚举映射

### 噪声（可以忽略的）

- `<style scoped>` 中的样式 — 用移动端自己的 CSS
- `<template>` 中的 HTML 结构 — 改成移动端布局
- `@click="handleClick"` — 只关注函数做了什么，不关注 Vue 绑定方式
- `import ... from ...` — ES module 导入，移动端不用
- `components: { ... }` — 组件注册，移动端不用
- `props: { ... }`、`emits: [...]` — 组件通信，移动端是单文件
- `<el-*>` 组件标签名 — 关注它的 prop/label/v-if，不关注它是什么组件

---

## API 提取模板

在分析 PC 代码时，用这个格式整理每个 API：

```markdown
## API 清单

### 1. 列表查询
- **URL**: `/pc/sbill/selectPage`
- **方法**: POST
- **请求参数**:
  ```
  {
    page: number,           // 页码，从1开始
    pageSize: number,       // 每页条数
    code?: string,          // 单号模糊搜索
    status?: string,        // 状态，多个逗号分隔 "B,C"
    ccId?: string,          // 主体ID
    deptId?: string,        // 部门ID
    dateOrderRange?: string,// 日期范围 "2024-01-01,2024-12-31"
    sortField?: string,     // 排序字段
    sortOrder?: string,     // "desc" | "asc" | ""
  }
  ```
- **响应结构**:
  ```
  {
    data: [{
      id, code, status, ccName, deptName, supplierName,
      customerName, handingUserName, createUserName, createTime,
      dateOrder, postingTime, approvalStatus
    }]
  }
  ```
- **调用位置**: 页面加载 / 筛选 / 翻页 / 排序

### 2. 详情查询
- **URL**: `/pc/sbill/select`
- **方法**: POST
- **请求参数**: `{ id: string }`
- **响应结构**: `{ data: { ...所有字段, detailList: [...] } }`
...
```

---

## 常见陷阱

1. **PC 端用了 `this.$set` 或者直接读写 data，字段名可能不是请求的原始名称** — 以 API 返回的字段名为准
2. **PC 端的 `watch` 可能隐式做了数据转换** — 如日期格式转换 `"2024-01-01 00:00:00"` → `"2024-01-01"`
3. **PC 端的 `computed` 可能有副作用** — 如自动计算合计，移过来要改成显式调用 `updateTotal()`
4. **PC 端可能对 API 响应做了二次处理** — 在 `then()` 回调中留意 `map`、`filter`、`reduce` 等操作，特别是字段重命名
5. **PC 端可能有多个 Vue 组件** — 确保读完了所有子组件，特别是 dialog 弹窗、表单组件、表格组件
6. **列表页的字段可能分散在多个文件** — 搜索 `columns` 可能定义在单独的 config 文件中
7. **详情页可能藏在列表页的弹窗里** — 搜索 `dialog`、`visible`、`detail`，不要只找单独的 detail 文件
8. **详情字段以 PC 端实际展示为准，不要自行增减** — 逐字段对比 PC 端详情/预览组件（如 `xxxPreview.vue`）中展示的字段。PC 端展示了哪些字段，移动端就展示哪些字段；PC 端没展示的字段，移动端也不要加。不要在清单里夹带私货。
9. **编辑表单 label 必须与 PC 端逐字一致** — 每个 `<el-form-item label="xxx">` 的 label 文本就是移动端表单的显示标签。不要自己改写（如 PC 端写"经手人"你就不能写成"经办人"）。字段顺序也以 `<el-form-item>` 的出现顺序为准。
