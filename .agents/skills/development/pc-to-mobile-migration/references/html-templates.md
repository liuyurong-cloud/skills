# HTML 模板参考

## 依赖加载顺序（不能改变）

### list.html

```
CSS: jquery.mobile-1.4.5.css → reset.css → alert.css → globule.css → mobileSelect.css → dropload.css → ./css/list.css → ./css/detail.css
JS:  jquery.min.js → base.js → mobileSelect.min.js → selectDate.js → jquery.mobile-1.4.5.js → dropload.min.js → tool.js → ./js/list.js → start.js
```

**为什么 detail.css 在 list.html 中加载**：JQM 通过 AJAX 加载 detail.html 时只取 `<div data-role="page">` 的内容，不解析其 `<head>`。因此详情页的样式必须由"宿主页"（list.html）提供。

### edit.html

```
CSS: jquery.mobile-1.4.5.css → reset.css → mobileSelect.css → alert.css → globule.css → ./css/edit.css
JS:  jquery.min.js → base.js → decimal.js(CDN) → mobileSelect.min.js → KimiSelect/index.js → UdiScanner/index.js(type=module) → selectDate.js → jquery.mobile-1.4.5.js → tool.js → ./js/edit.js → start.js
```

**与 list.html 的差异**：
- 不需要 dropload.css（列表页才有无限滚动）
- 需要 decimal.js CDN（精确计算）
- 需要 KimiSelect + UdiScanner Web Components
- UdiScanner 是 ES module，必须 `type="module"`

### detail.html 的加载策略

detail.html **自身不加载任何 CSS 或 JS**。它的工作方式是：

```
用户点击 list.html 中的列表项
  → $.mobile.changePage("detail.html?id=xxx")
    → JQM AJAX 加载 detail.html，提取 <div data-role="page"> 插入 DOM
      → JQM 触发 pageinit 事件
        → list.js 中的 $(document).on("pageinit", "#{module}-detail", ...) 响应
          → 从 URL 取 id → 调 API → 渲染数据
```

---

## list.html 完整模板

```html
<!doctype html>
<html lang="zh-CN">
  <head>
    <meta charset="UTF-8" />
    <title>模块名</title>
    <meta name="viewport" content="width=device-width, initial-scale=1.0, minimum-scale=1.0, maximum-scale=1.0, user-scalable=no" />
    <meta name="format-detection" content="telephone=no" />
    <link rel="stylesheet" href="/css/jquery.mobile-1.4.5.css" />
    <link rel="stylesheet" href="/css/reset.css" />
    <link rel="stylesheet" href="/css/alert.css" />
    <link rel="stylesheet" href="/css/globule.css" />
    <link rel="stylesheet" href="/css/mobileSelect.css" />
    <link rel="stylesheet" href="/css/dropload.css" />
    <link rel="stylesheet" href="./css/list.css" />
    <link rel="stylesheet" href="./css/detail.css" />
  </head>
  <body>
    <div data-role="page" id="{module}-list">
      <!-- 固定导航栏 -->
      <div class="{module}-list__navbar">
        <input id="search" type="search" placeholder="搜索单号" />
        <img id="screenBtn" src="/image/icon_screen.png" />
        <img id="sort-order" src="/image/icon_sort.png" />
      </div>

      <!-- 列表 -->
      <ul id="list-content"></ul>

      <!-- 空状态 -->
      <div class="g-data-empty"><img src="/image/img-empty.png" /><p>暂无数据</p></div>

      <!-- 加载更多 -->
      <div id="loadMore"><img src="/image/icon_loading.gif" /><span>正在加载</span></div>

      <!-- 新增按钮（固定右下角） -->
      <a class="add-btn" href="edit.html"><img src="/image/icon_add.png" /></a>

      <!-- 筛选面板 -->
      <div class="screen-box">
        <div class="screen-box__item">
          <label>主体</label>
          <input id="triggerCc" type="text" readonly placeholder="请选择" />
        </div>
        <div class="screen-box__item">
          <label>部门</label>
          <input id="triggerDept" type="text" readonly placeholder="请选择" />
        </div>
        <div class="screen-box__status">
          <button data-status="B,C">全部</button>
          <button data-status="B">编辑中</button>
          <button data-status="C">已入账</button>
          <button data-status="O,P">已关闭</button>
          <button data-status="Q">已冲销</button>
        </div>
        <div class="screen-box__date">
          <input id="checkDate1" type="text" readonly placeholder="开始日期" />
          <span>—</span>
          <input id="checkDate2" type="text" readonly placeholder="结束日期" />
        </div>
        <div class="screen-box__btns">
          <button id="resetScreen">重置</button>
          <button id="submitScreen">确定</button>
        </div>
      </div>
    </div>
  </body>
  <script src="/js/jquery.min.js"></script>
  <script src="/js/base.js"></script>
  <script src="/js/mobileSelect.min.js"></script>
  <script src="/js/selectDate.js"></script>
  <script src="/js/jquery.mobile-1.4.5.js"></script>
  <script src="/js/dropload.min.js"></script>
  <script src="/js/tool.js"></script>
  <script src="./js/list.js"></script>
  <script src="/js/start.js"></script>
</html>
```

---

## edit.html 完整模板

```html
<!doctype html>
<html lang="zh-CN">
  <head>
    <meta charset="UTF-8" />
    <title>模块名</title>
    <meta name="viewport" content="width=device-width, initial-scale=1.0, minimum-scale=1.0, maximum-scale=1.0, user-scalable=no" />
    <meta name="format-detection" content="telephone=no" />
    <link rel="stylesheet" href="/css/jquery.mobile-1.4.5.css" />
    <link rel="stylesheet" href="/css/reset.css" />
    <link rel="stylesheet" href="/css/mobileSelect.css" />
    <link rel="stylesheet" href="/css/alert.css" />
    <link rel="stylesheet" href="/css/globule.css" />
    <link rel="stylesheet" href="./css/edit.css" />
  </head>
  <body>
    <div data-role="page" id="{module}-edit">
      <!-- 步骤条 -->
      <div class="{module}-edit__step-bar">
        <div class="step-item active" id="step1">1. 创建单据</div>
        <div class="step-arrow"><img src="/image/icon_right_arrow.png" /></div>
        <div class="step-item" id="step2">2. 商品明细</div>
      </div>

      <!-- 第一步：表头信息 -->
      <div id="step1Content">
        <div class="form-section">
          <div class="form-item">
            <span class="form-item__label">单号</span>
            <span class="form-item__value" id="code">自动生成</span>
          </div>
          <div class="form-item required">
            <span class="form-item__label">单据日期</span>
            <input id="dateOrder" type="text" readonly placeholder="请选择日期" />
          </div>
          <!-- 更多表单行...每个下拉用 kimi-select -->
          <!-- 注意：所有下拉选择器都必须用 kimi-select，禁止使用原生 <select> -->
          <div class="form-item required">
            <span class="form-item__label">主体</span>
            <kimi-select id="ccSelect" placeholder="请选择" searchable></kimi-select>
          </div>
          <div class="form-item">
            <span class="form-item__label">备注</span>
            <textarea id="memo" placeholder="备注"></textarea>
          </div>
        </div>
        <button id="btnNext" class="btn-primary">下一步</button>
      </div>

      <!-- 第二步：商品明细 -->
      <div id="step2Content" style="display:none">
        <div id="lineSection"><!-- JS 动态渲染 --></div>
        <div class="add-line-wrapper">
          <button id="btnAddLine">+ 添加商品</button>
        </div>
        <!-- 固定底部汇总栏 -->
        <div id="lineSummary">
          <span>共 <em id="lineCount">0</em> 行</span>
          <span>合计：<em id="lineTotal">0.00</em></span>
          <button id="btnSaveDraft" class="btn-draft">暂存</button>
          <button id="btnSaveSubmit" class="btn-submit">入账</button>
        </div>
      </div>

      <!-- 共享组件（放在 step divs 之外） -->
      <udi-scanner id="udiScanner" label="扫描"></udi-scanner>
      <kimi-select id="productSelect" placeholder="请选择商品" searchable notrigger></kimi-select>
    </div>
  </body>
  <script src="/js/jquery.min.js"></script>
  <script src="/js/base.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/decimal.js@10.4.3/decimal.min.js"></script>
  <script src="/js/mobileSelect.min.js"></script>
  <script src="/components/KimiSelect/index.js"></script>
  <script src="/components/UdiScanner/index.js" type="module"></script>
  <script src="/js/selectDate.js"></script>
  <script src="/js/jquery.mobile-1.4.5.js"></script>
  <script src="/js/tool.js"></script>
  <script src="./js/edit.js"></script>
  <script src="/js/start.js"></script>
</html>
```

---

## detail.html 完整模板

```html
<!doctype html>
<html lang="zh-CN">
  <head>
    <meta charset="UTF-8" />
    <title>模块详情</title>
    <script>location.replace("list.html" + location.search);</script>
  </head>
  <body>
    <div id="{module}-detail" data-role="page">
      <!-- 信息卡片 -->
      <div class="detail-card">
        <div class="detail-card__header">基本信息</div>
        <div class="detail-card__body">
          <div class="detail-card__item">
            <span class="detail-card__label">单号</span>
            <span class="detail-card__value" data-field="code"></span>
          </div>
          <div class="detail-card__item">
            <span class="detail-card__label">状态</span>
            <span class="detail-card__value" data-field="status"></span>
          </div>
          <!-- 更多字段... -->
        </div>
      </div>

      <!-- 商品明细表 -->
      <div class="detail-section">
        <div class="detail-section__title">商品明细</div>
        <div class="detail-table-wrapper">
          <table class="detail-table">
            <thead>
              <tr>
                <th>序号</th>
                <th>商品名称</th>
                <th>规格型号</th>
                <th>数量</th>
                <th>单位</th>
                <th>注册证号</th>
                <th>生产日期</th>
                <th>有效期至</th>
                <th>批号</th>
                <th>存储条件</th>
              </tr>
            </thead>
            <tbody id="detailLineList"></tbody>
          </table>
        </div>
      </div>
    </div>
  </body>
</html>
```

**关键点**：
- `<head>` 中只有重定向 script，没有 CSS/JS 引用
- 直接访问 detail.html 会跳转到 list.html（保留 query string）
- 数据字段用 `data-field` 标记，方便 JS 填充
- 表格列多时用 `overflow-x: auto` 支持水平滑动
