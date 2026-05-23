# 视图字段清单模板

**这是必做产出物，不是可选项。** 读完 PC 端代码后、写移动端代码前，必须按此模板输出字段清单，作为后续实现和审查的契约。

**跳过此步骤的后果**：历史上多次出现因为跳过字段清单直接写代码，导致移动端字段与 PC 端不一致——详情页缺少或多余字段、编辑表单标签写错、表格列顺序错乱。这些错误到用户手动验收时才发现，返工成本极高。**先贴清单，再写代码，不要偷懒。**

---

## 如何从 PC 端 Vue 代码提取字段

### 列表视图字段

搜索 PC 端列表组件中的 `<el-table-column>` 或 `<th>`：

```html
<!-- Vue 信号 -->
<el-table-column prop="code" label="单号" width="150" />
<el-table-column prop="status" label="状态" />
<el-table-column prop="ccName" label="主体" />
```

**提取规则**：
- `prop` 的值就是字段名（API 返回的字段名）
- `label` 的值是显示名
- 列的顺序就是字段顺序（这是用户看到的顺序，移动端必须一致）
- 如果有 `v-if` 条件列，记录条件
- 如果有 `formatter` 或 `slot`，说明有二次处理，记录处理逻辑

### 详情视图字段

搜索 PC 端详情/查看组件。可能是 `<el-descriptions>`、普通 `<div>` 布局、或弹窗：

```html
<el-descriptions-item label="单号">{{ detail.code }}</el-descriptions-item>
<el-descriptions-item label="状态">{{ statusMap[detail.status] }}</el-descriptions-item>
```

**提取规则**：
- 记录每个字段的 API 字段名、显示标签、展示顺序
- 记录分组（如"基本信息"、"财务信息"、"商品明细"）
- 商品明细表：记录所有列（和列表视图一样的方法）

### 编辑视图字段

搜索 PC 端编辑/新增组件。通常是 `<el-form>` + `<el-form-item>`：

```html
<el-form-item label="单据日期" prop="dateOrder" :rules="[{ required: true }]">
  <el-date-picker v-model="form.dateOrder" />
</el-form-item>
<el-form-item label="主体" prop="ccId" :rules="[{ required: true }]">
  <el-select v-model="form.ccId" />
</el-form-item>
```

**提取规则**：
- `prop` = API 字段名
- `label` = 显示标签
- `rules` 中有 `required: true` = 必填项
- `v-model` 绑定的变量名（注意可能是 `form.xxx` 或直接 `xxx`）
- 如果有 `:disabled="status !== 'B'"` 之类的条件，记录可编辑条件
- 注意默认值（如 `form.type: 31`）

---

## 模板：列表视图字段清单

```markdown
## 列表视图字段清单

### 列表项展示字段（按顺序）
| 序号 | API字段名 | 显示标签 | 数据来源 | 二次处理 | 备注 |
|------|-----------|----------|----------|----------|------|
| 1 | code | 单号 | API selectPage → list[].code | 无 | |
| 2 | status | 状态 | API selectPage → list[].status | statusMap 映射 | B→编辑中, C→已入账, O/P→已关闭, Q→已冲销 |
| 3 | ccName | 主体 | API selectPage → list[].companyChildName | 无 | 注意字段名是 companyChildName 不是 ccName |
| 4 | deptName | 部门 | API selectPage → list[].deptName | 无 | |
| 5 | dateOrder | 单据日期 | API selectPage → list[].dateOrder | 截取日期部分 | PC端可能带时分秒 |
| ... | ... | ... | ... | ... | |

### 列表项操作按钮
| 按钮 | 显示条件 | 触发动作 |
|------|----------|----------|
| 入账 | status === 'B' | POST /pc/sbill/post |
| 编辑 | status === 'B' | 跳转 edit.html |
| 删除 | status === 'B' | POST /pc/sbill/delete |
| 复制 | 始终显示 | 跳转 edit.html?type=copy |

### 列表页筛选条件
| 筛选字段 | 组件类型 | API参数名 | 默认值 |
|----------|----------|-----------|--------|
| 单号 | input text | code | "" |
| 主体 | select | ccId | ""（全部） |
| 部门 | select | deptId | ""（全部） |
| 状态 | 多选按钮 | status | "B,C" |
| 日期范围 | date picker ×2 | dateOrderRange | "" |
```

---

## 模板：详情视图字段清单

```markdown
## 详情视图字段清单

### 分组1：基本信息
| 序号 | API字段名 | 显示标签 | data-field | 数据来源 | 二次处理 | 备注 |
|------|-----------|----------|------------|----------|----------|------|
| 1 | code | 单号 | code | API select → result.code | 无 | |
| 2 | status | 状态 | status | API select → result.status | statusMap 映射 | |
| 3 | dateOrder | 单据日期 | dateOrder | API select → result.dateOrder | 截取日期 | |
| ... | ... | ... | ... | ... | ... | |

### 分组2：人员/组织信息（如果有）
| 序号 | API字段名 | 显示标签 | data-field | 二次处理 | 备注 |
|------|-----------|----------|------------|----------|------|
| ... | ... | ... | ... | ... | |

### 分组3：商品明细表
| 序号 | API字段名 | 表头 | 数据来源 | 二次处理 | 备注 |
|------|-----------|------|----------|----------|------|
| 1 | lineNo | 序号 | detailList[].lineNo | 无 | |
| 2 | productName | 商品名称 | detailList[].productName | 无 | |
| 3 | productCode | 商品编码 | detailList[].productCode | 无 | |
| 4 | spec | 规格型号 | detailList[].spec | 无 | |
| 5 | inNum | 数量 | detailList[].inNum | 无 | |
| 6 | unitName | 单位 | detailList[].unit | 无 | |
| 7 | recordNo | 注册证号 | detailList[].recordNo | 无 | |
| 8 | dateBegin | 生产日期 | detailList[].dateBegin | 截取日期 | |
| 9 | dateEnd | 有效期至 | detailList[].dateEnd | 截取日期 | |
| 10 | batNo | 批号 | detailList[].batNo | 无 | |
| 11 | storageCondition | 存储条件 | detailList[].storeCond | 无 | 注意字段名是 storeCond |
| ... | ... | ... | ... | ... | |

数据来源：API 响应的 `detailList` 数组

**合计行检查**：如果 PC 端详情表格有 `show-summary` 或 `getSummaries()` 方法，移动端必须在表格底部渲染合计行。尤其当表格列包含"单价"/"金额"等财务字段时，几乎必定有合计行。不要遗漏此项。
```

---

## 模板：编辑视图字段清单

```markdown
## 编辑视图字段清单

### 表头字段（第一步）
| 序号 | API字段名 | 显示标签 | 数据来源 | 组件 | 必填 | 默认值/写死值 | 可编辑条件 | 备注 |
|------|-----------|----------|----------|------|------|---------------|------------|------|
| 1 | code | 单号 | 后端生成 | 纯文本 | - | "自动生成" | 新增时显示此文本 | |
| 2 | dateOrder | 单据日期 | 用户选择 / API回填 | 日期选择 | 是 | 当天 | 始终 | |
| 3 | ccId | 主体 | API ccompany/selectAll → kimi-select | kimi-select | 是 | 无 | 始终 | 切换后重新加载商品列表 |
| 4 | deptId | 部门 | API dept/selectAll → kimi-select | kimi-select | 是 | 无 | 始终 | |
| 5 | stockUser | 库管员 | API user/selectAll → kimi-select | kimi-select | 是 | 无 | 始终 | |
| 6 | handingUser | 经手人 | API user/selectAll → kimi-select | kimi-select | 否 | localStorage userid | 始终 | |
| 7 | memo | 备注 | 用户输入 | textarea | 否 | "" | 始终 | |
| ... | ... | ... | ... | ... | ... | ... | ... | |

### 写死的常量（doSave 时传入）
| 字段名 | 值 | 说明 |
|--------|-----|------|
| type | 31 | 业务类型（不同模块不同） |
| isRed | 0 | 红字标志 |
| currencyId | 1 | 币种，默认人民币 |

### 商品行字段（第二步，makeLine 对象）
| 序号 | API字段名 | 显示标签 | 数据来源 | 输入方式 | 必填 | 备注 |
|------|-----------|----------|----------|----------|------|------|
| 1 | lineNo | 序号 | 自动 | 自动编号 | - | |
| 2 | productId | 商品ID | porder/selectAllDetail → productId | kimi-select | 是 | 提交时传给 API |
| 3 | productName | 商品名称 | porder/selectAllDetail → productName | 自动填入 | - | 选择商品后回填 |
| 4 | productCode | 产品编码 | porder/selectAllDetail → productCode | 自动填入 | - | 选择商品后回填 |
| 5 | spec | 规格 | porder/selectAllDetail → spec | 自动填入 | - | 选择商品后回填 |
| 6 | packingSize | 包装 | porder/selectAllDetail → packingSize | 自动填入 | - | 选择商品后回填 |
| 7 | unit | 单位 | porder/selectAllDetail → unit | 自动填入 | - | 选择商品后回填 |
| 8 | batNo | 批号 | 用户输入 | input | 是 | 输入后300ms防抖查询 |
| 9 | recordNo | 注册证号 | 批次查询 API 回填 | input(可编辑) | 否 | 查询后自动填充，也可手动修改 |
| 10 | dateBegin | 生产日期 | 批次查询 API 回填 | 日期选择 | 否 | 查询后自动填充并锁定 |
| 11 | dateEnd | 有效期 | 批次查询 API 回填 | 日期选择 | 否 | 查询后自动填充并锁定 |
| 12 | inNum | 入库数量 | 用户输入 | input number | 是 | |
| 13 | tax | 税率 | API product/selectAllTax → MobileSelect | MobileSelect 选择 | 否 | PC 端数据来源 |
| 14 | cost | 未税单价 | 用户输入 / 计算 | input number | 是 | 可手动输入，也可由含税单价反算 |
| 15 | price | 含税单价 | 选商品回填 / 计算 | input number | 是 | 选商品时回填，可手动修改，也可由未税单价正算 |
| ... | ... | ... | ... | ... | ... | |

### 校验规则清单
| 规则 | 触发位置 | 提示文案 |
|------|----------|----------|
| 日期必填 | 下一步/保存 | "请选择单据日期" |
| 主体必填 | 下一步/保存 | "请选择主体" |
| 至少1行商品 | 保存 | "请至少添加一行商品" |
| 每行必选商品 | 保存 | "请选择第X行商品" |
| 每行必填批次号 | 保存 | "请填写第X行批次号" |
| 每行必填数量 | 保存 | "请填写第X行数量" |
| ... | ... | ... |
```

---

## 使用方式

1. **FE Agent**：读完 PC 代码后，按此模板输出三份视图清单 → 贴到对话中给用户确认 → 按清单实现 → 实现完后逐项自查
2. **CV Agent**：拿到这三份清单后，逐项对比移动端代码，标记不一致项
3. **用户**：可以直接查看清单确认字段是否完整，不需要读 PC 代码
