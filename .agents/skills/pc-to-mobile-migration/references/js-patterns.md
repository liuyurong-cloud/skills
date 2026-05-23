# JS 编码模式参考

基于 instock 模块的实际实现。所有模式使用 `$.me` 命名空间。

---

## JS 编码规范（强制）

### 规则1：事件回调必须抽离为 $.me 命名方法

`initEvent()`、`initSelectors()` 中**禁止**使用内联匿名函数作为回调。每个事件处理函数必须是独立的 `$.me.xxx` 方法。

**❌ 错误**：
```js
initEvent: function () {
  $("#btnSave").click(function () {
    if (!$.me.validateStep1()) return;
    $.me.doSave(1);
  });
  $("#btnDelete").click(function () {
    $.myConfirm({
      title: "删除确认",
      message: "确认删除？",
      callback: function () {
        $.me.deleteItem();
      },
    });
  });
},
```

**✅ 正确**：
```js
initEvent: function () {
  $("#btnSave").click($.me.onSaveClick);
  $("#btnDelete").click($.me.onDeleteClick);
},

onSaveClick: function () {
  if (!$.me.validateStep1()) return;
  $.me.doSave(1);
},

onDeleteClick: function () {
  $.myConfirm({
    title: "删除确认",
    message: "确认删除？",
    callback: $.me.doDelete,
  });
},

doDelete: function () {
  $.me.deleteItem();
},
```

### 规则2：initSelectors() 必须是干净的绑定列表

每个 kimi-select 的 change 事件回调抽成独立方法，不要写内联匿名函数。

**❌ 错误**：
```js
initSelectors: function () {
  $(document).on("change", "#ccSelect", function (e) {
    var detail = e.originalEvent ? e.originalEvent.detail : e.detail;
    if (!detail) return;
    $.me.ccId = detail.value;
  });
  $(document).on("change", "#supplierSelect", function (e) {
    var detail = e.originalEvent ? e.originalEvent.detail : e.detail;
    if (!detail) return;
    $.me.supplierId = detail.value;
  });
},
```

**✅ 正确**：
```js
initSelectors: function () {
  $(document).on("change", "#ccSelect", $.me.onCcSelectChange);
  $(document).on("change", "#supplierSelect", $.me.onSupplierSelectChange);
},

onCcSelectChange: function (e) {
  var detail = e.originalEvent ? e.originalEvent.detail : e.detail;
  if (!detail) return;
  $.me.ccId = detail.value;
},

onSupplierSelectChange: function (e) {
  var detail = e.originalEvent ? e.originalEvent.detail : e.detail;
  if (!detail) return;
  $.me.supplierId = detail.value;
},
```

### 规则3：$.selectDate / $.selectDateSimple 回调必须抽离

**❌ 错误**：
```js
$.selectDate("#dateOrder", { start: 2020, end: 2030, title: "选择日期" },
  function (data) {
    var d = data.year + "-" + String(data.month).padStart(2, "0") + "-" + String(data.day).padStart(2, "0");
    $("#dateOrder").val(d);
    $.me.dateOrder = d;
  }
);
```

**✅ 正确**：
```js
// 在 $.me 上定义回调方法
onDateOrderSelect: function (data) {
  var d = data.year + "-" + String(data.month).padStart(2, "0") + "-" + String(data.day).padStart(2, "0");
  $("#dateOrder").val(d);
  $.me.dateOrder = d;
},

// 调用时传入引用
$.selectDate("#dateOrder", { start: 2020, end: 2030, title: "选择日期" }, $.me.onDateOrderSelect);
```

### 规则4：AJAX 回调超过 5 行应提取为命名方法

**❌ 错误**：
```js
list: function () {
  // ...
  $.request(window.apiUrl + "/pc/sbill/selectPage", "post", params, function (res) {
    if ($.me.pageNum === 1) $("#list-content").html("");
    if (res.result && res.result.list.length > 0) {
      $.me.render(res.result.list);
      if ($("#list-content").find("li").length >= res.result.total) $.me.pageMore = false;
    } else {
      $.me.pageMore = false;
      if ($("#list-content").find("li").length === 0) {
        $("#list-content").html('<div class="list-empty">...</div>');
      }
    }
  });
},
```

**✅ 正确**：
```js
list: function () {
  if (!$.me.pageMore) return;
  // 组装参数...
  $.request(window.apiUrl + "/pc/sbill/selectPage", "post", params, $.me.onListLoaded);
},

onListLoaded: function (res) {
  if ($.me.pageNum === 1) $("#list-content").html("");
  if (res.result && res.result.list.length > 0) {
    $.me.render(res.result.list);
    if ($("#list-content").find("li").length >= res.result.total) $.me.pageMore = false;
  } else {
    $.me.pageMore = false;
    if ($("#list-content").find("li").length === 0) {
      $("#list-content").html('<div class="list-empty">...</div>');
    }
  }
},
```

### 规则5：$.myConfirm / $.myAlert 的 callback 必须抽离

**❌ 错误**：
```js
$.myConfirm({
  title: "入账确认",
  message: "确认入账该单据？",
  callback: function () {
    $.me.sendCheck(id);
  },
});
```

**✅ 正确**：
```js
onCheckClick: function (e) {
  e.stopPropagation();
  $.me._pendingCheckId = $(this).closest("li").data("id");
  $.myConfirm({
    title: "入账确认",
    message: "确认入账该单据？",
    callback: $.me.doCheck,
  });
},

doCheck: function () {
  $.me.sendCheck($.me._pendingCheckId);
},
```

### 规则6：initEvent() 只做事件绑定，不放业务逻辑

`initEvent()` 应该是一目了然的绑定列表，读者不需要读任何实现细节就能知道这个页面有哪些交互。

```js
initEvent: function () {
  // 搜索
  document.onkeyup = $.me.onSearchEnter;
  // 筛选
  $("#screenBtn").click($.me.onScreenOpen);
  $(".screen-close").click($.me.onScreenClose);
  $(".screen-submit").click($.me.onScreenSubmit);
  // 排序
  $("#sort-order").click($.me.onSortClick);
  // 状态筛选
  $("#statusList").on("click", "li", $.me.onStatusClick);
  // 列表操作
  $("#list-content").on("click", ".item-main", $.me.onItemClick);
  $("#list-content").on("click", ".btn-check", $.me.onCheckClick);
  $("#list-content").on("click", ".btn-edit", $.me.onEditClick);
  $("#list-content").on("click", ".btn-delete", $.me.onDeleteClick);
  // 新增
  $("#addBtn").click($.me.onAddClick);
},
```

### 规则7：每个方法只做一件事

- **不要**把表单校验、数据组装、AJAX 调用全塞在一个方法里
- **不要**在 `renderLineHtml()` 里同时做数据查询和 DOM 操作
- `renderLineHtml()` 超过 ~100 行时，把子区域（如税率选项、产品信息行）拆成独立方法

```js
renderLineHtml: function (i) {
  var item = $.me.detailList[i];
  return (
    '<div class="line-card" data-index="' + i + '">' +
    $.me.renderLineHeader(i, item) +
    $.me.renderLineProductInfo(i, item) +
    $.me.renderLinePriceInfo(i, item) +
    $.me.renderLineBatchInfo(i, item) +
    '</div>'
  );
},

renderLineHeader: function (i, item) { /* ... */ },
renderLineProductInfo: function (i, item) { /* ... */ },
renderLinePriceInfo: function (i, item) { /* ... */ },
renderLineBatchInfo: function (i, item) { /* ... */ },
```

### 规则8：优先使用 ES6+ 语法

箭头函数、`const`/`let`、模板字符串、方法简写等现代语法均可使用，兼容性已足够。

---

## list.js 结构

```
$.me = {
  // == 状态变量 ==
  pageNum, pageSize, sortOrder, pageMore
  searchCode, xxId, deptId, status, checkDate1, checkDate2  // 筛选条件
  xxArr, deptArr                                            // 缓存的下拉数据

  // == 方法 ==
  init()              // 入口：有 id 参数则跳过（详情页模式）
  initSelect()        // MobileSelect 实例化（筛选面板用）
  initDatePicker()    // $.selectDate()
  initEvent()         // 搜索、筛选、排序、列表项点击、scrollstop
  loadSelectData()    // 获取下拉选项 + 缓存
  resetScreen()       // 重置筛选条件
  resetList()         // pageNum=1 + 清空DOM + 重新请求
  list()              // AJAX 分页请求
  render(arr)         // 拼接 HTML → append 到 #list-content
  sendCheck(id)       // 入账操作
  getDetail(id)       // 详情请求
  renderDetail(data)  // 填充详情页
}
```

### 筛选器初始化（MobileSelect trigger 方式）

```js
initSelect: function () {
  var selectXx = new MobileSelect({
    trigger: "#triggerXx",
    wheels: [{ data: [{ id: "", value: "全部" }] }],
    callback: $.me.onXxSelect,  // ← 回调抽离为命名方法
  });
  $.me.selectXx = selectXx; // 保存引用！后续 updateWheel 需要
},

onXxSelect: function (indexArr, data) {
  $.me.xxId = data[0].id;
  $.me.resetList();
},
```

### 更新 MobileSelect 选项（异步加载后）

```js
loadSelectData: function () {
  $.request(window.apiUrl + "/pc/ccompany/selectAll", "GET", null, function (res) {
    $.me.xxArr = res.data || [];
    $.me.selectXx.updateWheel([
      { data: [{ id: "", value: "全部" }].concat(
        $.me.xxArr.map(function (item) { return { id: item.id, value: item.name }; })
      )}
    ]);
  });
},
```

### 排序三态切换

```js
// 三态循环：descend → ascend → default → descend
$("#sort-order").click(function () {
  var map = { descend: "ascend", ascend: "default", default: "descend" };
  $.me.sortOrder = map[$.me.sortOrder];
  $.me.resetList();
});
```

排序值在请求时映射为 API 参数：
```js
sortOrder: $.me.sortOrder === "descend" ? "desc"
          : ($.me.sortOrder === "ascend" ? "asc" : "")
```

### 状态筛选多选

```js
// 状态按钮 toggle active，收集后逗号拼接
$page.on("click", ".screen-box__status button", function () {
  $(this).toggleClass("active");
  var statuses = [];
  $page.find(".screen-box__status button.active").each(function () {
    statuses.push($(this).data("status"));
  });
  $.me.status = statuses.join(",");
  $.me.resetList();
});
```

### scrollstop 无限滚动

```js
$(document).on("scrollstop", "#{module}-list", function () {
  var scrollTop = $(window).scrollTop();
  var docHeight = $(document).height();
  var winHeight = $(window).height();
  if (docHeight - scrollTop - winHeight < 500) {
    $.me.pageNum++;
    $.me.list();
  }
});
```

### 列表项渲染（含操作按钮 + 阻止冒泡）

```js
render: function (arr) {
  var html = "";
  arr.forEach(function (item) {
    html += '<li class="list-item" data-id="' + item.id + '">';
    html += '<div class="list-item__header">';
    html += '<span class="list-item__code">' + (item.code || "") + '</span>';
    html += '<span class="list-item__status status-' + (item.status || "") + '">'
          + $.me.statusMap[item.status] + '</span>';
    html += '</div>';
    html += '<div class="list-item__info">';
    html += '<div><label>主体</label><span>' + (item.ccName || "") + '</span></div>';
    html += '<div><label>部门</label><span>' + (item.deptName || "") + '</span></div>';
    // ...更多字段
    html += '</div>';
    html += '<div class="list-item__actions">';
    html += '<button class="btn-check">入账</button>';
    html += '<button class="btn-edit">编辑</button>';
    html += '</div>';
    html += '</li>';
  });
  $("#list-content").append(html);
  $("#loadMore").toggle($.me.pageMore);
  $(".g-data-empty").toggle(
    !$.me.pageMore && $("#list-content").children().length === 0
  );
},

// 事件委托 + 阻止冒泡
initEvent: function () {
  var $page = $("#{module}-list");

  // 列表项点击 → 详情
  $page.on("click", ".list-item", function () {
    var id = $(this).data("id");
    $.mobile.changePage("detail.html?id=" + id);
  });

  // 按钮点击 → 阻止冒泡！（否则会触发上面列表项点击）
  $page.on("click", ".btn-check", function (e) {
    e.stopPropagation();
    var id = $(this).closest(".list-item").data("id");
    $.me.sendCheck(id);
  });

  $page.on("click", ".btn-edit", function (e) {
    e.stopPropagation();
    var id = $(this).closest(".list-item").data("id");
    location.href = $.addVerToUrl("edit.html?id=" + id);
  });
},
```

### 详情页 pageinit（写在 list.js 文件底部）

```js
$(document).on("pageinit", "#{module}-detail", function () {
  var id = $.getUrlParam(window.location.href, "id");
  if (id) $.me.getDetail(id);
});
```

---

## edit.js 结构

```
$.me = {
  // == 状态 ==
  id, type           // "add" | "edit" | "copy"
  dateOrder, ccId, deptId, ...  // 表头字段
  lineItems: []      // 商品行数组（核心数据源）
  _editingLineIdx    // 临时：当前编辑的行索引

  // == 方法 ==
  init()             // 入口：解析 URL → 初始化选择器 → 加载数据
  setDefaults(data)  // 编辑回填：同步数据层 + UI 层
  goToStep1/2()      // 步骤切换
  initSelectors()    // 绑定 kimi-select change 事件
  loadBaseData()     // 加载 6+ 个下拉选项
  loadProductList()  // 加载商品列表（依赖 ccId），必须在 init 时加载，不能在 goToStep2 中才加载
  loadData()         // 编辑/复制时回填
  initEvent()        // 所有事件绑定

  // 商品行管理
  makeLine(item, lineNo)   // 创建规范化行对象
  addLine()                // 添加空行
  calcAmount(inNum, cost)  // decimal.js 乘法
  updateLineAmount(idx)    // 更新单行金额
  updateTotal()            // 更新底部合计
  renderLineHtml(i)        // 渲染单行
  renderLines()            // 渲染全部行

  // 业务查询
  queryBatNo(idx, batNo, productId)  // 批次号查询（500ms 防抖）

  // 保存
  validateStep1()   // 第一步校验
  doSave(toSend)    // 0=暂存 1=入账
}
```

### 类型判断

```js
init: function () {
  $.me.id = $.getUrlParam($(this)[0].baseURI, "id");
  var urlType = $.getUrlParam($(this)[0].baseURI, "type");
  if ($.me.id && urlType === "copy") {
    $.me.type = "copy";  // 复制：有 id + type=copy
  } else if ($.me.id) {
    $.me.type = "edit";  // 编辑：有 id
  }
  // 否则 type = "add"（默认）
},
```

### KimiSelect change 事件（关键格式）

```js
initSelectors: function () {
  $(document).on("change", "#ccSelect", $.me.onCcSelectChange);
},

onCcSelectChange: function (e) {
  $.me.ccId = e.originalEvent.detail.value;   // ← 取值方式
  $.me.loadProductList();                      // ← 联动刷新（主体变更时重新加载）
  $("#productSelect")[0].clear();              // ← 清空关联选择器
},

// 重要：loadProductList() 必须在 init 流程中调用一次（loadBaseData 之后），
// 不能延迟到 goToStep2() 才调用。如果 ccId 有默认值，init 时就要加载商品列表，
// 确保用户点击步骤2时 productSelect 的 options 已经就绪。
```

### loadProductList 与 kimi-select 字段映射

商品列表加载时，kimi-select 的 `value-field` 和 `label-field` 属性必须与 API 返回的字段名一致：

```js
loadProductList: function () {
  $.request(window.apiUrl + "/pc/product/selectAll", "GET", { ccId: $.me.ccId }, function (res) {
    var list = res.data || [];
    // HTML: <kimi-select id="productSelect" value-field="productId" label-field="productName" ...>
    $("#productSelect")[0].options = list.map(function (item) {
      return {
        productId: item.productId,     // ← 与 value-field="productId" 对应
        productName: item.productName, // ← 与 label-field="productName" 对应
        productCode: item.productCode,  // 额外字段，从 option.code 取
        spec: item.spec,                // 额外字段，从 option.spec 取
        unit: item.unit,                // 额外字段，从 option.unit 取
      };
    });
  });
},
```

**陷阱**：不要默认用 `value`/`label` 作为字段名。必须查看 API 响应中实际返回的字段名（grep PC 端代码确认），然后在 HTML 的 `value-field`/`label-field` 属性和 options 数组中保持一致。

### 共享 kimi-select + _editingLineIdx 模式

所有商品行共用一个 `#productSelect`（带 `notrigger` 属性，隐藏输入框）。用户点击某行的"选择商品"按钮时：

```js
// 点击"选择商品"按钮（在 renderLineHtml 中生成）
$("#lineSection").on("click", ".btn-select-product", function () {
  var idx = $(this).closest(".line-card").index();
  $.me._editingLineIdx = idx;     // ← 记录当前编辑的行
  $("#productSelect")[0].open();  // ← 打开共享选择器
});

// 选择器 change 事件 → 将数据写入正确的行
$(document).on("change", "#productSelect", function (e) {
  var option = e.originalEvent.detail.option;
  var idx = $.me._editingLineIdx;
  if (idx == null) return;
  var line = $.me.lineItems[idx];
  line.productId = option.value;
  line.productName = option.label;
  line.productCode = option.code;
  line.spec = option.spec;
  line.unit = option.unit;
  $.me.renderLineHtml(idx);  // 重新渲染这一行
});
```

### 商品行管理

```js
makeLine: function (item, lineNo) {
  // 统一的数据模型，所有字段在这里定义
  return {
    lineNo: lineNo || 1,
    productId: item.productId || "",
    productName: item.productName || "",
    productCode: item.productCode || "",
    spec: item.spec || "",
    unit: item.unit || "",
    unitName: item.unitName || "",
    batNo: item.batNo || "",
    recordNo: item.recordNo || "",
    dateBegin: item.dateBegin || "",
    dateEnd: item.dateEnd || "",
    inNum: item.inNum || item.num || "",
    cost: item.cost || "",
    amount: item.amount || "",
    // ...照着 PC 端字段全部列出来，不要漏
  };
},

addLine: function () {
  $.me.lineItems.push($.me.makeLine({}, $.me.lineItems.length + 1));
  $.me.renderLines();
  $.me.updateTotal();
},

onDeleteLineClick: function () {
  var idx = $(this).closest(".line-card").index();
  $.me.lineItems.splice(idx, 1);
  $.me.renderLines();
  $.me.updateTotal();
},
```

绑定放在 `initEvent()` 中：
```js
$("#lineSection").on("click", ".btn-del-line", $.me.onDeleteLineClick);
```

### decimal.js 精确计算

```js
calcAmount: function (inNum, cost) {
  if (!inNum || !cost) return "0.00";
  return new Decimal(inNum).mul(new Decimal(cost)).toFixed(2);
},

updateTotal: function () {
  var total = $.me.lineItems.reduce(function (sum, line) {
    return sum.add(new Decimal(line.amount || 0));
  }, new Decimal(0));
  $("#lineTotal").text(total.toFixed(2));
  $("#lineCount").text($.me.lineItems.length);
},
```

### 批次号查询（500ms 防抖）

```js
// input 事件 + setTimeout 防抖
var batTimer = null;
// 绑定放在 initEvent() 中
$("#lineSection").on("input", ".inp-batNo", $.me.onBatNoInput);

onBatNoInput: function () {
  var idx = $(this).closest(".line-card").index();
  var batNo = this.value;
  var productId = $.me.lineItems[idx].productId;
  clearTimeout(batTimer);
  batTimer = setTimeout(function () {
    $.me.queryBatNo(idx, batNo, productId);
  }, 500);
},

queryBatNo: function (idx, batNo, productId) {
  if ($.isEmpty(batNo) || $.isEmpty(productId)) return;
  $.request(window.apiUrl + "/pc/sbill/selectBalanceByBatNo", "POST", {
    batNo: batNo,
    productId: productId
  }, function (res) {
    if (!res.data) return;
    var line = $.me.lineItems[idx];
    line.recordNo = res.data.recordNo || "";
    line.dateBegin = res.data.dateBegin || "";
    line.dateEnd = res.data.dateEnd || "";
    $.me.renderLineHtml(idx);
    // 禁用自动填充的字段
    $(".line-card").eq(idx)
      .find(".inp-recordNo, .inp-dateBegin, .inp-dateEnd")
      .prop("disabled", true);
  });
},
```

### doSave 模式

```js
saveStaging: function () { $.me.doSave(0); },
saveSubmit: function () { $.me.doSave(1); },

doSave: function (toSend) {
  // 逐行校验
  for (var i = 0; i < $.me.lineItems.length; i++) {
    var line = $.me.lineItems[i];
    if ($.isEmpty(line.productId)) {
      $.myToast("请选择第" + (i + 1) + "行商品");
      return;
    }
    if ($.isEmpty(line.batNo)) {
      $.myToast("请填写第" + (i + 1) + "行批次号");
      return;
    }
  }

  var data = {
    id: $.me.type !== "add" ? $.me.id : undefined,
    // ...所有表头字段
    detailList: $.me.lineItems,
    type: 31,       // ← 业务类型，从 PC 端获取
    isRed: 0,       // ← 是否为红字单据
    currencyId: 1,  // ← 币种，默认人民币
  };

  // 暂存数据以便回调使用
  $.me._saveData = data;

  if (toSend) {
    $.myConfirm({
      title: "提示",
      message: "确认提交入账？",
      callback: $.me.doRequestSave,  // ← 回调必须抽离为命名方法
    });
  } else {
    $.me.doRequestSave();
  }
},

doRequestSave: function () {
  var data = $.me._saveData;
  var url = $.me.type === "add"
    ? window.apiUrl + "/pc/sbill/add"
    : window.apiUrl + "/pc/sbill/edit";
  $.request(url, "POST", data, $.me.onSaveSuccess);
},

onSaveSuccess: function () {
  $.myToast($.me.type === "add" ? "新增成功" : "修改成功");
  setTimeout(function () {
    location.href = $.addVerToUrl("list.html");
  }, 1000);
},
```

### setDefaults：编辑回填（数据层 + UI 层同步）

```js
setDefaults: function (data) {
  // 1. 同步数据层
  $.me.code = data.code;
  $.me.dateOrder = data.dateOrder;
  $.me.ccId = data.ccId;
  // ...每个字段

  // 2. 同步 UI 层
  $("#code").text(data.code);
  $("#dateOrder").val(data.dateOrder ? data.dateOrder.split(" ")[0] : "");
  $("#ccSelect")[0].value = data.ccId;
  // ...每个 UI 元素

  // 3. 渲染商品行
  $.me.lineItems = (data.detailList || []).map(function (item, i) {
    return $.me.makeLine(item, i + 1);
  });
  $.me.renderLines();
  $.me.updateTotal();
},
```

---

## 常见模式速查

| 模式 | 代码 |
|------|------|
| URL 参数 | `$.getUrlParam($(this)[0].baseURI, "id")` |
| 页面跳转 | `location.href = $.addVerToUrl("list.html")` |
| JQM 跳转 | `$.mobile.changePage("detail.html?id=" + id)` |
| DOM 元素 | `$("#xxx")` |
| Web Component | `$("#xxxSelect")[0].value` / `.options` / `.open()` / `.clear()` |
| 事件委托 | `$("#parent").on("click", ".child", fn)` |
| 阻止冒泡 | `e.stopPropagation()` |
| 判空 | `$.isEmpty(val)` |
| Token | `$.request` 自动处理（`Access-Token-PC` header） |
| 防重复 | `$.request` 自动设置 `$.me.locked` |
