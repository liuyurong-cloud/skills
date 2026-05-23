# 数据流分析模板

**这是必做产出物，在 API 契约清单和视图字段清单之前完成。** 如果搞不清楚数据从哪来、到哪去，字段清单就是无源之水。

---

## 核心问题

在提取任何字段之前，先回答三个问题：

1. **数据从哪来？** — API 响应？URL 参数？localStorage？用户输入？计算得出？
2. **数据到哪去？** — 展示在哪个视图？传参给哪个 API？存储到哪？
3. **什么交互触发数据变化？** — 用户点击？选择器联动？定时刷新？

**禁止的行为**：看到一个字段就加到移动端，不追溯它的来龙去脉。这是"多余字段"和"遗漏字段"的根因。

---

## 模板：数据流全景图

```markdown
## 数据流分析

### 1. 列表页数据流

**触发时机**：页面加载 / 筛选条件变化 / 排序 / 翻页

**数据来源**：
| 数据 | 来源 | 说明 |
|------|------|------|
| 列表数据 | POST /pc/sbill/selectPage | types=11, 含筛选/分页/排序参数 |
| 下拉选项-主体 | GET /pc/ccompany/selectAll | 页面加载时获取，缓存 |
| 下拉选项-供应商 | POST /pc/supplier/selectAll | 页面加载时获取，缓存 |

**数据消费**：
| 数据 | 消费位置 | 方式 |
|------|----------|------|
| list[].code + status | 列表卡片 header | 直接取值 + statusMap 映射 |
| list[].ccName, deptName... | 列表卡片 info grid | 直接取值 |
| ccArr, supplierArr | 筛选面板 | MobileSelect updateWheel |

**交互触发**：
| 用户操作 | 触发效果 |
|----------|----------|
| 输入搜索码 + 回车 | 重置 pageNum→1 → 重新请求 |
| 点击筛选图标 | 展开筛选面板 |
| 筛选面板点确定 | 收集筛选值 → 重置 pageNum→1 → 关闭面板 → 重新请求 |
| 点击排序图标 | 切换 sortOrder 三态 → 重新请求 |
| 滚动到底部 500px 内 | pageNum++ → 追加请求 |
| 点击列表项 | $.mobile.changePage 到 detail.html |
| 点击入账按钮(B状态) | e.stopPropagation → $.myConfirm → POST /pc/sbill/sendCheck |
| 点击编辑按钮(B状态) | e.stopPropagation → 跳转 edit.html?id=xxx |
| 点击删除按钮(B状态) | e.stopPropagation → $.myConfirm → POST /pc/sbill/delete |

### 2. 详情页数据流

**触发时机**：从列表页点击进入（JQM pageinit 事件）

**数据来源**：
| 数据 | 来源 | 说明 |
|------|------|------|
| id | URL 参数 | $.getUrlParam(url, "id") |
| 详情主体 | POST /pc/sbill/select {id} | res.result 包含所有字段 + detailList |
| 商品明细 | 同上 API 的 res.result.detailList | 数组，每个元素是一行商品数据 |

**数据消费**：
| 数据 | 消费位置 | 方式 |
|------|----------|------|
| result.code, status... | 详情信息卡片 | $("#detailXxx").text() 逐个填充 |
| result.detailList | 商品明细表格 | for 循环拼 HTML → $("#detailTableBody").html() |
| 合计金额 | 表格底部合计行 | reduce 累加 inNum × price |

**交互触发**：
| 用户操作 | 触发效果 |
|----------|----------|
| 直接访问 detail.html | 顶部 redirect script → 跳转到 list.html |

### 3. 编辑页数据流

**触发时机**：从列表页点击新增/编辑/复制 / 直接访问 edit.html

**数据来源**：
| 数据 | 来源 | 说明 |
|------|------|------|
| id | URL 参数 | $.getUrlParam(url, "id") |
| type | URL 参数 | $.getUrlParam(url, "type"), 决定 add/edit/copy |
| 已有单据数据 | POST /pc/sbill/select {id} | 编辑/复制模式时加载 |
| 下拉-主体 | GET /pc/ccompany/selectAll | 页面加载时获取 |
| 下拉-供应商 | POST /pc/supplier/selectAll | 页面加载时获取 |
| 下拉-用户 | GET /pc/user/selectAll | 页面加载时获取 |
| 下拉-仓库 | GET /pc/stock/selectAll | 页面加载时获取 |
| 下拉-税率 | GET /pc/product/selectAllTax | 页面加载时获取 |
| 商品列表 | POST /pc/porder/selectAllDetail {ccId, supplierId, productName, status:C} | goToStep2 时加载 |

**数据消费**：
| 导出数据 | 消费位置 | 方式 |
|----------|----------|------|
| POST /pc/sbill/add 或 edit | 保存按钮 | doSave 拼装全部 form 数据 |
| detailList 写入 form | 商品明细行 | makeLine 规范化为 25+ 字段的对象 |

**交互触发**：
| 用户操作 | 触发效果 |
|----------|----------|
| 选择到货日期 | $.selectDate 弹出 → 回调设置 $.me.dateOrder |
| 选择主体 | kimi-select change → $.me.ccId = value |
| 选择供应商 | kimi-select change → $.me.supplierId = value |
| 点击下一步 | validateStep1 → goToStep2 → loadProductList |
| 点击切换步骤 | 回到步骤1 |
| 点击添加商品 | addLine → renderLines → refreshSummary |
| 点击行内"选择" | 设置 curScanIdx → 打开 productSelect |
| 商品选中 | kimi-select change → makeLine 填充 → renderLines |
| 输入数量/单价 | input 事件 → updateLineAmount → updateTotal |
| 输入批号 | input 事件 → 300ms防抖 → 查询批次 → 自动填充日期/注册证号 |
| 点击暂存 | doSave(0) → 逐行校验 → POST /pc/sbill/add 或 edit |
| 点击入账 | doSave(1) → 逐行校验 → $.myConfirm → POST |

### 4. 关键计算逻辑

| 计算 | 公式 | 触发时机 | 精度 |
|------|------|----------|------|
| 行金额 | inNum × cost（未税单价） | 数量或单价变化 | Decimal, toFixed(9) |
| 合计 | Σ(行金额) | 任何行金额变化 | Decimal, toFixed(2) |
| 含税→未税 | cost = price / (1 + tax/100) | 选商品 / 改含税单价 | Decimal, toFixed(9) |
| 未税→含税 | price = cost × (1 + tax/100) | 改未税单价 / 改税率 | Decimal, toFixed(9) |

### 5. 写死的常量和默认值

| 常量 | 值 | 位置 | 说明 |
|------|-----|------|------|
| type | 11（采购入库单） | doSave 时传入 | 每个模块不同，必须确认 PC 端值 |
| types | "11"（字符串） | selectPage 查询时 | 同上但传字符串 |
| isRed | 0 | doSave 时传入 | 红字标志 |
| currencyId | 1 | doSave 时传入 | 人民币 |
| status 默认筛选 | "B,C" | 列表初始化 | 未结 = 编辑中 + 已入账 |
```

---

## 使用方式

1. **FE Agent**：读完 PC 代码后，先写数据流分析 → 贴给用户确认 → 再写 API 契约 → 再写视图字段清单 → 再写代码
2. **用户**：从数据流分析可以一眼看出 AI 是否真正理解了 PC 端的业务逻辑，而不只是机械地抄了字段名
3. **CV Agent**：用数据流分析来验证移动端的交互逻辑是否完整
