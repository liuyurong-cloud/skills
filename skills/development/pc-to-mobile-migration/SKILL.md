---
name: pc-to-mobile-migration
description: |
  Use this skill whenever the user mentions migrating a PC-side module, porting Vue code to mobile, creating a mobile version of an existing feature, or implementing new module pages (list/edit/detail) based on PC-side logic. Also trigger when the user talks about "移动端化", "迁移模块", "PC端转移动端", provides PC Vue file paths alongside a module name, or asks to create mobile pages that mirror existing PC functionality. This skill orchestrates a three-agent pipeline (FE→CV→TEST) that ensures 1:1 feature parity between PC Vue code and the jQuery Mobile target, with automated testing to validate correctness.
---

# PC端→移动端模块迁移

将 PC 端 Vue 功能模块 1:1 迁移到 jQuery + jQuery Mobile 移动端，按三阶段流水线执行。

## 核心原则

1. **1:1 功能对等**：除用户明确排除的功能外，PC 端每一个功能点都必须在移动端实现
2. **逻辑一致**：校验规则、计算方式、状态流转、API 参数组装必须与 PC 端完全一致
3. **字段名原样沿用**：API 请求/响应的字段名不允许改动，不确定就 grep PC 代码确认
4. **字段顺序原样沿用**：列表展示、详情分组、表单布局的字段顺序必须以 PC 端为准，不得自行调整
5. **禁止胡编乱造**：每个字段名、每个 API 参数、每个标签文字，都必须能在 PC 代码中找到原始出处。不确定就 grep，grep 不到就不要加
6. **先理解数据流再动手**：读完 PC 代码后，必须先输出三份分析文档（数据流 → API 契约 → 视图字段），**逐份获得用户确认后**才能创建文件
7. **以 instock/ 为唯一模板**：参考 `instock/` 的目录结构、依赖加载顺序、JS 模式

## 参考文件

当需要查看完整代码示例时，按需读取：

| 文件 | 内容 | 何时读 |
|------|------|--------|
| `references/pc-code-reading.md` | 如何高效阅读 PC 端 Vue 代码 | 开始分析 PC 代码前 |
| `references/data-flow-analysis.md` | **必做1**：数据流分析模板（数据从哪来、到哪去、交互如何连接） | 读完 PC 代码后、写 API 清单前 |
| `references/field-checklist-template.md` | **必做3**：视图字段清单模板 | 数据流分析之后 |
| `references/html-templates.md` | 三种页面的完整 HTML 和依赖顺序 | 创建 HTML 文件时 |
| `references/js-patterns.md` | list.js 和 edit.js 的完整编码模式 | 写 JS 逻辑时 |
| `references/css-patterns.md` | CSS 命名、单位、状态色、常用布局、审查清单 | 写 CSS 前和完成后 |
| `references/quick-reference.md` | 工具函数 API、KimiSelect API、Vue→jQuery 映射表 | 随时查阅 |

---

## 目标模块结构

```
{module}/
├── css/
│   ├── list.css       # scoped to #{module}-list
│   ├── edit.css       # scoped to #{module}-edit
│   └── detail.css     # scoped to #{module}-detail
├── js/
│   ├── list.js        # 列表 + 详情（$.me），详情逻辑通过 pageinit 触发
│   └── edit.js        # 编辑/新增（$.me），含两步向导 + 商品行管理
├── list.html          # 列表页，包含筛选面板，detail.css 在此加载
├── edit.html          # 编辑页，含 <kimi-select> + <udi-scanner> + decimal.js CDN
└── detail.html        # 仅 body 内容 + 重定向 script，不加载自己的 CSS/JS
```

**三条关键设计决策（instock 验证过的）**：

1. **detail.css 在 list.html 中加载，detail.js 逻辑在 list.js 中** — JQM 通过 AJAX 加载 detail.html 时只取 `<div data-role="page">`，不解析其 `<head>`。详情页的样式和脚本必须由"宿主页"提供。
2. **筛选面板用 MobileSelect trigger，表单用 `<kimi-select>`** — 筛选条件简单、不需要搜索功能，用 MobileSelect 即可。表单需要搜索、联动、禁用等能力，用 KimiSelect。
3. **分页用 JQM 的 `scrollstop` 事件** — 不用 dropload 库（虽然加载了），scrollstop 更轻量且与 JQM 配合更好。

---

## 阶段1：FE Agent — PC 代码分析 + 移动端实现

### ⛔ 强制门禁（违反此规则的后果：字段错位、标签不对、功能遗漏）

**读完 PC 代码后，必须按顺序产出以下三份分析文档。每份文档贴到对话中，获得用户确认后，才能进入下一步。禁止跳过任何一步直接写代码。**

```
读 PC 代码
  ↓
【门禁1】数据流分析 → 用户确认 ✓
  ↓
【门禁2】API 契约清单 → 用户确认 ✓  
  ↓
【门禁3】视图字段清单 → 用户确认 ✓
  ↓
读 instock 参考代码
  ↓
创建移动端文件
  ↓
自查
```

---

### 步骤1：分析 PC 端代码（按优先级阅读）

参考 `references/pc-code-reading.md` 了解详细的阅读策略。核心顺序：

1. **API 调用**（最高优先级）→ 搜索 `axios`/`fetch`/`request(`，记录每个 API 的完整 URL、方法、参数、响应结构
2. **数据模型** → 搜索 `data()`/`reactive(`，理解表单字段、列表结构、写死的常量（`type: 31`, `isRed: 0`, `currencyId: 1`）
3. **校验规则** → 搜索 `validate`/`required`/`if (!`/`message.error`，整理必填项和业务规则清单
4. **状态流转** → 搜索 `status`/`state`，画出状态图 + 每个状态下的可见按钮/操作
5. **计算逻辑** → 搜索 `Decimal`/`computed`，确认用什么库、什么精度

**读取时必须逐文件、逐行阅读**：PC 端可能有多个文件（父页面 + 子组件 + 弹窗组件 + preview 组件）。确保所有相关文件都已读完再开始输出分析文档。

---

### 步骤2：门禁1 — 数据流分析

按 `references/data-flow-analysis.md` 模板输出。核心回答三个问题：

1. **数据从哪来？** — 每个视图用到的数据，逐一标注来源（哪个 API 的哪个字段？URL 参数？localStorage？计算得出？）
2. **数据到哪去？** — 每个 API 调用产出的数据，在哪些视图被消费？怎么消费？
3. **什么交互触发数据变化？** — 用户每个操作（点击、输入、选择）触发什么效果？

**贴到对话中，等待用户确认"数据流分析没问题"。用户确认后，进入门禁2。**

> 这个分析是"多余字段"和"遗漏字段"的第一道防线。如果数据流分析里没有提到的数据，就不能出现在移动端。如果数据流分析里有提到的数据但移动端没实现，就是遗漏。

---

### 步骤3：门禁2 — API 契约清单

数据流分析确认后，整理每个 API。**每个请求参数、每个响应字段都必须能追溯到 PC 代码中的具体行。**

```markdown
## API 清单

### 1. 列表查询
- **URL**: `/pc/sbill/selectPage`
- **方法**: POST
- **PC 代码位置**: revorder.vue:322-338 (selectPageFun 方法)
- **请求参数**:
  ```
  { page, pageSize, code?, status?, ccId?, supplierId?, dateOrderRange?, sortField?, sortOrder? }
  ```
- **响应字段**:
  ```
  res.result.list[].id, code, status, companyChildName, supplierName, dateOrder, dateCheck, isRed, createUserName, handingUserName
  res.result.total
  ```
- **调用时机**: 页面加载 / 筛选 / 翻页 / 排序
```

**关键要求**：
- 每个 API 标注它出现在 PC 代码的哪一行
- 请求参数名和 PC 端实际发送的字段名逐字一致
- 响应字段名和 API 实际返回的字段名一致（不能自己推测）
- `.then()` 回调中有 `map`/`filter`/字段重命名等二次处理的，必须记录

**贴到对话中，等待用户确认"API 契约没问题"。用户确认后，进入门禁3。**

---

### 步骤4：门禁3 — 视图字段清单

按 `references/field-checklist-template.md` 模板输出三张表。**每个字段必须标注它来自 API 契约中的哪个接口、哪个字段。** 如果某个字段无法追溯到 API 契约，说明你在胡编——删掉它。

1. **列表视图字段清单**：按顺序、含操作按钮条件、筛选条件
2. **详情视图字段清单**：按分组顺序、含商品明细表全部列
3. **编辑视图字段清单**：表头字段（含必填/默认值）、商品行字段（含来源）、校验规则

**贴到对话中，等待用户确认"视图字段没问题"。用户确认后，才能开始读 instock 和写代码。**

---

### 步骤5：阅读 instock 参考代码

**必须完整读取**以下文件（不要跳过）：

```
instock/list.html        → 学习依赖加载顺序
instock/js/list.js       → 学习 scrollstop 分页、pageinit 详情、筛选逻辑
instock/edit.html        → 学习 decimal.js CDN 位置、kimi-select 用法
instock/js/edit.js       → 学习两步向导、商品行管理、doSave 模式
instock/detail.html      → 学习重定向 script + data-field 模式
```

---

### 步骤6：实现移动端

**严格按照门禁2和门禁3的清单实现**，不要依赖记忆，不要添加清单上没有的东西。创建顺序：CSS → list.html + list.js → detail.html → edit.html + edit.js

每个文件的依赖加载顺序**完全不可改变**，详见 `references/html-templates.md`。

**list.js** — `init()` 检测 id 参数跳过列表；MobileSelect 保存引用；`resetList()` 三位一体；`e.stopPropagation()`；`pageinit` 监听详情；`scrollstop` 分页

**edit.js** — `$.me.type` 三态；`init()` 流程；`e.originalEvent.detail.value`；共享选择器 + `_editingLineIdx`；`setDefaults` 双同步；`makeLine` 全字段；`decimal.js`；批次查询防抖；`doSave(0|1)`

详见 `references/js-patterns.md`。

---

### 步骤7：自查（对照门禁2+3清单逐项验证）

1. 列表字段名、顺序与"列表视图字段清单"一致
2. 详情字段名、分组、顺序与"详情视图字段清单"一致
3. 编辑表单字段名、顺序、必填标记、默认值与"编辑视图字段清单"一致
4. 商品明细表列名、顺序与清单一致
5. 操作按钮的显示条件、触发动作与清单一致
6. 每个 API 的 URL、方法、参数名、响应字段名与"API 契约清单"一致
7. 所有校验规则与 PC 端一致
8. 状态流转逻辑正确
9. 金额/数量计算使用 decimal.js
10. kimi-select 取值使用 `e.originalEvent.detail.value`
11. detail.html 有重定向 script，detail.css 在 list.html 中加载
12. 依赖加载顺序与 instock 模板一致
13. CSS 选择器以页面 ID 开头
14. 没有添加清单以外的字段或功能

---

## 阶段2：CV Agent — 代码审查

用独立 agent 执行审查，对比 PC 端和移动端代码。

**审查时必须读取**：PC 端原始代码 + 移动端实现代码 + 步骤2产出的两份清单。

### 审查1：视图字段逐项对比（新增，最高优先级）

这是最常见的错误来源——字段不一致要到用户手动 check 才能发现。

**列表视图**：
- [ ] 逐字段对比：移动端 `render()` 中展示的字段名、顺序与清单中"列表视图字段清单"一致
- [ ] 状态映射：`statusMap` 的枚举值和显示文字与 PC 端一致
- [ ] 操作按钮：显示条件（status 判断）、触发动作与 PC 端一致
- [ ] 筛选条件：筛选字段、默认值与 PC 端一致

**详情视图**：
- [ ] 逐字段对比：移动端 `detail.html` 中 `data-field` 的值与清单中"详情视图字段清单"一致
- [ ] 分组顺序：detail-card 的顺序与 PC 端详情页分区顺序一致
- [ ] 表格列：`detail-table` 的表头、列顺序与清单一致

**编辑视图**：
- [ ] 逐字段对比：移动端 `edit.html` 中的表单项、id、label、顺序与清单中"编辑视图字段清单"一致
- [ ] 必填标记：带 `required` class 的字段与清单中"必填=是"的字段一致
- [ ] 默认值：写死的常量（`type`, `isRed`, `currencyId`）与清单一致
- [ ] 商品行字段：`makeLine()` 返回对象的字段列表覆盖清单中"商品行字段"的所有字段
- [ ] 校验规则：`validateStep1()`/`doSave()` 中的校验项与清单中"校验规则清单"一致

### 审查2：API 接口一致性

**每个 API 逐参数对比**：
- [ ] URL 路径与 PC 端完全一致（逐字符）
- [ ] HTTP 方法与 PC 端一致
- [ ] 请求参数的字段名、类型、嵌套结构与 PC 端一致
- [ ] 是否遗漏了 PC 端有但移动端没有的参数
- [ ] 是否多传了 PC 端没有的参数
- [ ] 响应数据字段名与 PC 端一致（特别注意 `selectPage` 的 `data` 数组元素字段名）

### 审查3：CSS 规范检查（新增）

- [ ] 所有 CSS 选择器以 `#{module}-list` / `#{module}-edit` / `#{module}-detail` 开头（样式隔离）
- [ ] BEM 命名一致：`{module}-list .list-item__header` 而不是裸的 `.header`
- [ ] 状态颜色与 PC 端一致（检查 `status-B/C/O/P/Q` 的颜色值）
- [ ] 布局使用 rem（`1rem ≈ 100px`），微调用 px
- [ ] 固定定位元素（底部栏、顶部导航栏）有对应的 padding 防止遮挡
- [ ] 水平滚动表格设置了 `min-width` 防止挤压
- [ ] KimiSelect Shadow DOM 穿透使用 `::part()` 选择器
- [ ] 没有使用 `!important`（除非覆盖 JQM 默认样式）
- [ ] 表单 label 宽度一致，对齐整齐

### 审查4：业务逻辑完整性

- [ ] 状态流转逻辑（哪些状态可编辑、哪些按钮可见）
- [ ] 校验规则（必填项、业务规则、行级校验）
- [ ] 计算逻辑（金额=数量×单价，合计=各行之和，精确度）
- [ ] 排序（三态切换、sortField/sortOrder 映射）
- [ ] 筛选（条件组合、重置行为、默认值如 `status: "B,C"`）
- [ ] 写死的常量（`type`, `isRed`, `currencyId`）

### 审查5：HTML 结构

- [ ] 依赖加载顺序与 instock 一致
- [ ] detail.html 有重定向 script，detail.css 在 list.html 中加载
- [ ] 所有必需的表单元素存在

### 审查6：JS 编码规范检查（新增）

- [ ] `initEvent()` 中没有内联匿名函数，所有回调都是 `$.me.xxx` 命名方法
- [ ] `initSelectors()` 中没有内联匿名函数
- [ ] `$.selectDate` / `$.myConfirm` / `$.myAlert` 回调已抽离
- [ ] 超过 5 行的 AJAX 回调已提取为命名方法
- [ ] `renderLineHtml()` 未超过 ~100 行，超长已拆分
- [ ] 优先使用 ES6+ 语法（箭头函数、方法简写、const/let、模板字符串等）

### 问题处理

- 发现不一致 → 分析根因（PC 端代码写的就是这个字段名吗？）→ 回写 FE Agent 修复
- 移动端有 PC 端没有的功能 → 标记出来与用户确认是否保留
- 全部一致 → 交付 TEST Agent

---

## 阶段3：TEST Agent — 自动化测试

### 第一层：静态分析（必须执行）

编写 Node.js 脚本检查：
1. API 调用对比 — 提取两端所有 HTTP 请求，对比 URL + method + params
2. 字段名检查 — 提取移动端代码中使用的 API 响应字段名，与 PC 端对比
3. HTML 结构验证 — 检查关键元素是否存在、依赖顺序是否正确

### 第二层：E2E 测试（条件执行）

如果项目可本地访问，用 Playwright 测试：
- 列表页：加载、筛选、排序、滚动分页、点击进入详情
- 编辑页：表单填写、kimi-select 交互、商品行增删、暂存/提交
- 详情页：数据正确展示

测试报告输出为 markdown。

---

## 常见错误与排查

| 问题 | 根因 | 解决 |
|------|------|------|
| 列表字段顺序和 PC 端不一样 | 没按清单实现，凭感觉排序 | 严格对照"列表视图字段清单"逐字段实现 |
| 详情页缺少字段 | 字段清单遗漏或实现时跳过 | 重新 grep PC 端详情组件，补全字段 |
| API 参数名错误 | 自己编的字段名，没 grep PC 代码 | 每个参数名都去 PC 代码中搜索确认 |
| 详情页 JS 不执行 | JQM AJAX 不执行 `<script>` 标签 | 在 list.js 用 `pageinit` 事件处理 |
| 详情页样式丢失 | detail.css 只在 detail.html 引用 | 在 list.html 中加载 detail.css |
| kimi-select change 不触发 | 事件绑定方式错误 | 用 `$(document).on("change", "#id", fn)` |
| 取值总是 undefined | 用 `e.target.value` 而不是 `e.originalEvent.detail.value` | 改用 `e.originalEvent.detail.value` |
| 浮点计算错误 | 用了 JS 原生 `*` 运算符 | 改用 `new Decimal(a).mul(b)` |
| 列表滚动不加载 | scrollstop 绑定在错误的选择器上 | 用 `$(document).on("scrollstop", "#{module}-list", fn)` |
| 点击按钮跳到了详情 | 按钮冒泡触发父级列表项点击 | `e.stopPropagation()` |
| MobileSelect 数据不更新 | 直接改了 data 属性 | 用 `selectInstance.updateWheel([{data: newArr}])` |
| CSS 样式污染其他页面 | 选择器未 scoped 到 page ID | 所有样式以 `#{module}-list/edit/detail` 开头 |
| 样式在不同页面间冲突 | 用了通用 class 名（如 `.header`） | 用 BEM 命名 `.{module}-list__navbar` |
| initEvent 函数越来越长 | 回调里写业务逻辑 | 回调抽离为 `$.me.xxx` 方法，initEvent 只做绑定 |
| 代码难以调试和测试 | 内联匿名函数无法被单独调用 | 所有回调都是命名方法，可在 console 直接调 |
| kimi-select change 处理不统一 | 每个选择器复制粘贴内联函数 | 抽成独立的 `onXxSelectChange` 方法 |
| 多次点击重复提交 | `$.me.locked` 被绕过 | `$.request` 自动管理 `$.me.locked`，请求完自动重置 |
| 商品选择影响所有行 | 未用 `_editingLineIdx` 区分当前行 | 在打开 productSelect 前设置 `$.me._editingLineIdx = idx` |
| 部分事件回调是内联匿名函数 | 有的开发者偷懒只写了一部分命名方法 | 所有回调 100% 抽离为命名方法，initEvent/initSelectors 只做绑定 |
| 商品选择器打开时空列表 | loadProductList 延迟到 goToStep2 才调用 | 在 init 流程中（loadBaseData 之后）就加载商品列表，不在步骤切换时才加载 |
| 表单中用了原生 `<select>` | 原生 select 移动端表现不一致 | 所有表单下拉都用 `<kimi-select>`，包括税率、币种等简单选项 |
| 商品选择后 kimi-select 不显示 | value-field/label-field 与 API 字段名不匹配 | 检查 API 响应中的实际字段名（如 `productId`/`productName`），在 HTML 属性和 options 数组中保持一致 |
| 详情表格缺少合计行 | 没检查 PC 端是否有 show-summary | grep PC 端详情表格的 `show-summary` 或 `getSummaries`，如有则渲染 `<tfoot>` |
| 详情页展示了 PC 端没有的字段 | 凭想象添加了"可能有用的字段" | 严格按 PC 端实际展示字段实现，不做加减 |

---

## 不该做的事

- **不要凭记忆实现** — 写代码时对照步骤2的两份清单，不要依赖"我觉得PC端是这样的"
- **不要改变字段顺序** — 列表、详情、表单的字段顺序以 PC 端为准
- **不要发明字段名** — API 字段名以 PC 端代码中实际使用的为准，每个不确定的都 grep
- **不要简化逻辑** — `if/else` 分支、边界条件全部移植
- **不要加额外功能** — 除非用户明确要求
- **不要改变依赖顺序** — CSS/JS 加载顺序经过验证
- **不要在 detail.html 加载 JS/CSS** — JQM 不解析它的 `<head>`
- **不要用 dropload 做分页** — 用 JQM 自带的 `scrollstop`
- **不要用 JS 原生浮点运算** — 金额计算必须用 decimal.js
- **不要用 `e.target.value` 取 kimi-select 的值** — 用 `e.originalEvent.detail.value`
- **不要在 CSS 中用非 scoped 选择器** — 所有样式必须以页面 ID 开头
- **不要在 initEvent() / initSelectors() 中写内联匿名函数** — 所有回调必须抽离为 `$.me.xxx` 命名方法
- **优先使用 ES6+ 语法** — 箭头函数、const/let、模板字符串、方法简写等均可使用
- **不要把业务逻辑塞在 initEvent() 里** — initEvent 只做事件绑定，业务逻辑放在独立的命名方法中
