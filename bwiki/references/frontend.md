# 前端能力与实现规范

## 1. 运行时

| 项 | 值 |
|---|---|
| MediaWiki | 1.37.0 |
| 语法 | **仅 ES5**（`MediaWiki:Common.js` 与小工具须通过 ES5 检查） |
| CSS 框架 | Bootstrap **3.3.7** |
| JS 库 | jQuery **3.6.0** |
| 皮肤 | 基于 Vector 魔改 |
| 样式机制 | `#css` 解析函数、`<bstyle>` 标签；**无 TemplateStyles** |
| 用户个人 JS/CSS | **不存在** |
| 可视化编辑器 | 不支持 |

## 2. ES5 禁用清单

| 特性 | 禁用写法 | 可用写法 |
|---|---|---|
| `let` / `const` | `let a = 1;` | `var a = 1;` |
| 箭头函数 | `() => {}` | `function () {}` |
| class | `class Foo {}` | `function Foo() {}` + 原型 |
| 模板字符串 | `` `Hi ${name}` `` | `'Hi ' + name` |
| 解构 | `var {x, y} = obj;` | `var x = obj.x, y = obj.y;` |
| Map / Set | `new Map(); new Set();` | 普通对象、数组、`Object.create(null)` |
| Promise | `new Promise()` | 回调、jQuery Deferred |
| `for...of` | `for (var x of arr) {}` | `for (var i = 0; i < arr.length; i++)` |
| 展开运算符 | `[...arr]`、`{...obj}` | `concat`、`Object.assign` |
| 剩余参数 | `function (...args) {}` | `arguments` |
| 默认参数 | `function (x = 1) {}` | `x = x \|\| 1;` |
| 对象简写 | `{ x, y }` | `{ x: x, y: y }` |
| 方法简写 | `{ foo() {} }` | `{ foo: function () {} }` |
| import / export | 模块语法 | 全局命名空间、ResourceLoader |
| Symbol / Proxy / Reflect | — | 不可用 |
| `Array.prototype.includes` | `arr.includes(x)` | `arr.indexOf(x) !== -1` |
| `String.prototype.includes/startsWith/endsWith` | — | `indexOf` 判断 |

## 3. CSS

### 3.1 `#css` 与 `<bstyle>`

| | `#css` | `<bstyle>` |
|---|---|---|
| 输出位置 | 页面 `<head>` | 标签所在位置 |
| 参数 | 内联 CSS、CSS 页面名、以 `/` 开头的服务器路径 | 直接写 CSS |
| 安全清洗 | 有（`Sanitizer::checkCss`） | 无 |
| 可用 `url()` / `var()` | **否** | 是 |

`#css` 命中以下任一关键字/函数时，**整段 CSS 被替换为 `/* insecure input */`**：

`url`、`var`、`attr(…url…)`、`filter`、`accelerator`、`expression`、`image`、`image-set`、`-o-link`、`-o-link-source`、`-o-replace`。

需要这些能力时改用 `<bstyle>`。

### 3.2 去重

`#css` 与 `<bstyle>` **不自动去重**，同一页面多次注入相同样式会造成冗余甚至渲染异常。用页面级变量保证只注入一次：

```
{{#if:{{#varexists:名称}} | | {{#vardefine:名称|1}} {{#css: ... }} }}
```

`<bstyle>` 的等价写法，内容需用注释包裹的 `<pre>` 保护，避免缩进与换行被 wikitext 解析：

```
{{#if:{{#varexists:名称}} | | {{#vardefine:名称|1}}<bstyle>/*<pre>*/ ... /*</pre>*/</bstyle>}}
```

被大量页面复用的样式放 `MediaWiki:Common.css`，不要靠模板重复注入。首页尤其禁止重复调用含 `#css`/`<bstyle>` 的模板。

### 3.3 Bootstrap

Bootstrap 3.3.7 的栅格与组件类可直接使用（`row`、`col-md-*`、`panel`、`btn`、`table` 等）。不要使用 BS4/BS5 类名（`d-flex`、`col-lg-*` 的 flex 语义、`gap-*` 等）。

## 4. Widget

- 权限：创建/修改需**管理员**或**小部件编辑者**；普通用户可直接调用。
- 调用：`{{#widget:名称}}`，名称区分大小写（首字母除外）；子页面写全路径 `{{#widget:Chart/Bar}}`。
- 页面结构：`<includeonly>` 放代码，`<noinclude>` 放说明。
- **输出绕过 wikitext 解析**：组件内的 `[[链接]]`、`{{模板}}`、`{{{参数}}}`、`{{#函数}}` 会原样输出。需要 wikitext 解析的内容先在模板中解析，再交给 Widget。
- 缓存：保存 Widget 后调用页**不自动刷新**（包括自己），需对调用页 purge。
- 作用域：JS 用 IIFE 包裹；多实例场景用 class 不用 id；避免对 `div`/`table`/`p` 设全局样式破坏皮肤。

### 4.1 参数传递

首选 **DOM 传参**：模板输出带 `data-*` 的容器，Widget 脚本读取 DOM，不拼接 HTML。

```
<includeonly><div class="wiki-fold" data-open="{{{默认展开|false}}}">
  <div class="wiki-fold-h">{{{标题|详情}}}</div>
  <div class="wiki-fold-b">{{{内容|}}}</div>
</div>{{#widget:FoldPanel}}</includeonly>
```

Smarty 参数只用于受控标识符（纯数字 ID、白名单枚举），且必须转义：

| 修饰符 | 用途 |
|---|---|
| `default:'…'` | 缺省值 |
| `validate:'int'` / `'float'` / `'boolean'` / `'url'` | 类型校验，不符输出空串 |
| `escape:'html'` | HTML 实体转义，用于标签间纯文本或双引号包裹的非 URL 属性 |
| `escape:'url'` | `rawurlencode`，用于 URL 路径或查询参数值 |
| `escape:'javascript'` | 转义引号、反斜杠、换行，用于 JS 字符串字面量内部 |

定界符为 `<!--{` 与 `}-->`，例：`<!--{$title|default:'默认'|escape:'html'}-->`。禁止直接输出未修饰的 `<!--{$参数}-->`。

### 4.2 技术选型

| 需求 | 机制 |
|---|---|
| 复用排版、信息框、导航、提示框 | 模板 |
| 复杂条件、字符串/表处理、批量生成 | Lua 模块 |
| 原始 HTML、iframe、canvas、第三方图表、页面级交互 | Widget |
| 跨页面站点级脚本与样式 | Gadget |

静态排版与结构化内容优先用模板/模块，不要把纯排版逻辑放进 Widget。

## 5. Gadget

- 定义页：`MediaWiki:Gadgets-definition`（内容模型为 wikitext，仅**界面管理员**可编辑）。
- 脚本文件：`MediaWiki:Gadget-<名称>.js`；纯样式用 `MediaWiki:Gadget-<名称>.css`。
- 定义格式：`* 名称[选项]|脚本1|脚本2`，脚本名建议与名称一致。
- 常用选项（MW 1.37）：

| 选项 | 说明 |
|---|---|
| `ResourceLoader` | 启用资源加载器；纯 CSS 小工具可不开 |
| `dependencies=mediawiki.api,mediawiki.user` | 执行前加载依赖 |
| `rights=edit,delete` | 仅对拥有这些权限的用户可见 |
| `hidden` | 不在参数设置显示；需同时 `default`，否则无法启用 |
| `default` | 默认启用 |
| `type=general\|styles` | 纯 CSS 用 `styles` |

- 查看：`特殊:小工具`；使用统计：`特殊:小工具使用统计`。
- 缓存与 ES5 限制使小工具在 BWiki 的表现受限；不要引导用户用「设置」页开关（页面缓存会导致设置数据错乱）。

## 6. 脚本加载

### 6.1 RLQ 等待 jQuery

Widget 或早期脚本可能在 jQuery 之前执行：

```js
(window.RLQ = window.RLQ || []).push([["jquery"], function () {
  $(document).ready(function () {
    // 使用 $ / jQuery
  });
}]);
```

### 6.2 ResourceLoader

- 通过 `模板:ResourceLoader`（及派生的 `模板:JS`、`模板:CSS`）加载，实际逻辑在 `MediaWiki:Common.js` 或 `Widget:ResourceLoader`。
- 仅允许加载**本站**资源，且资源必须位于 `MediaWiki:` 命名空间，其它命名空间被拦截。
- 新增/修改加载逻辑需界面管理员；调用已有资源无需特殊权限。
- 用 `{{#varexists}}` 去重，避免同一页面重复加载。

### 6.3 调试

```js
mw.loader.load("/<code>/MediaWiki:Demo.js?action=raw&ctype=text/javascript&rand=" + Math.random(), "text/javascript");
```

- `&rand=` 仅用于开发期临时验证，不得写入 `Common.js` 或小工具常态加载（会让所有用户绕过缓存）。
- 需要按用户限制时用 `mw.config.values.wgUserGroups` 或 B 站 UID 判断。

## 7. 生效与验证

1. 保存后 CSS/JS 约 30 分钟生效，`MediaWiki:` 页面同样受解析器缓存影响。
2. 语法错误会导致整段脚本不执行，先看控制台。
3. 取源码核对：`?action=raw&ctype=text/javascript`。
4. 页面渲染异常时先 purge，再判断是样式冲突还是缓存。
