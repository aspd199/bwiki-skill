---
name: bwiki
description: BWiki（wiki.biligame.com 旗下任意 WIKI）平台规范与操作向导：硬约束红线、权限与审核、缓存与可见性、编辑语法、前端能力边界、内容准则、站点探测、API 直连。涉及 BWiki / bw / bilibili wiki / MediaWiki 词条的读写、样式或脚本开发时先加载；具体工具调用见 bwiki-mcp 技能。
---

# BWiki 平台规范

适用范围：`wiki.biligame.com` 农场内全部站点。站点代号由申请方自定、不可预测，不要靠猜 slug 找站。

## 0. 路径选择

| 任务 | 去向 |
|---|---|
| 读页面、写页面、移动、删除、上传、查历史 | 加载 `bwiki-mcp`：工具手册，会话与节流由服务端处理 |
| 写正文、表格、模板、分类 | §6 编辑写法 |
| 判断能不能做、会不会把站弄坏 | 本文 §2 硬约束、§7 前端能力 |
| 写完别人看不到 / 页面不更新 | §5 缓存与可见性 |
| 改 CSS/JS/Widget/Gadget | §7，细节见 `@references/frontend.md` |
| 接手陌生站点 | §9 站点探测 |
| 绕过 MCP 直接发 API | §10，代码见 `@references/api-fallback.md` |
| 报错定位 | §11 症状索引 |

## 1. 平台模型

| 项 | 事实 |
|---|---|
| 引擎 | MediaWiki **1.37.0**（不支持高版本特性，JS 仅 ES5） |
| 站点地址 | `https://wiki.biligame.com/<code>/` |
| API | `https://wiki.biligame.com/<code>/api.php` |
| 文件域名 | `patchwiki.biligame.com/images/<code>/...`（不要手工拼接，见 `@references/wikitext.md`） |
| 内容语言 | `zh-cn`；**不支持字词转换**，页面名区分大小写与繁简 |
| 账号 | B 站统一账号，首次访问自动建号（用户名为 B 站 UID） |
| 权限 | **按站点独立**：A 站管理员在 B 站无任何权限 |
| 站点规模 | 名册会增长：用 `list-wikis` 取实时值。不要假设某个站点代号存在 |
| 已装扩展 | SMW、Widgets、Gadgets、Scribunto、Moderation、Arrays、Variables、Loops、DPL、LST、ParserPower |
| 未提供 | 可视化编辑器、字词转换、TemplateStyles、用户个人 JS/CSS |

## 2. 硬约束

平台强制项与明确限制。违反会导致页面不可用、整站不可访问或平台处理。改动前逐条确认。

| # | 约束 | 检查动作 |
|---|---|---|
| R1 | 禁止打开页面自动播放音乐 | 检查是否引入 `<audio autoplay>` / 播放器组件 |
| R2 | 飘浮物遮罩面积不超过 10%，任何设备（平台措辞为「不建议超过」） | 检查浮层、全屏遮罩的宽高 |
| R3 | 禁止任何设备下自动弹框 | 检查 `alert`、自动展开的 modal、`Widget` 内定时弹出 |
| R4 | 禁止高强度或快速明暗闪烁 | 检查高频 `animation` / 闪烁贴图 |
| R5 | 禁止整页内容嵌套 | 嵌套页不得含额外分享、引导打开/下载 APP、非 B 站登录、额外评论系统 |
| R6 | **`MediaWiki:Sidebar` 禁止使用模板语法** | 只写纯链接列表；模板语法会导致**不可逆的整站无法访问** |
| R7 | 任何页面（含壁纸）禁止二维码 | 需要时联系管理员 |
| R8 | **首页禁止重复调用含 `#css` 或 `<bstyle>` 的模板** | 同一模板在首页只调用一次；改用 `Widget` 或站点 CSS |
| R9 | 页面 URL 禁止半角 `%` | 页面名含百分号时用全角 `％` |

社区规则中的行为约束（违反按社区/平台流程处理）：

- 禁止发布违法、侵权、广告垃圾内容；**禁止站外引流**（非引用、授权、转载用途的外链）。
- 禁止绕过平台限制，包括用脚本屏蔽页面区域。
- 尊重游戏原著：禁止外挂、脚本、改动游戏设定或基础数据；同人内容必须显著标注同人属性。
- 站内内容可被他人编辑、修改、删除、转载；站外转载需授权、署名、附原始链接且不得演绎。

## 3. 权限与审核

### 3.1 用户组

B 站等级 ≥2 且注册 >7 天，首次访问自动获得 `bilibili`；编辑达到一定次数后自动获得 `自动确认用户`（获得后不能手动撤销，只能由滥用过滤器撤销）。

| 用户组 | 能力 | 边界 |
|---|---|---|
| `bilibili` / `自动确认用户` | 编辑页面与评论；后者可编辑半保护页 | 无管理权 |
| `自动版主化用户` | **编辑跳过审核** | 由行政员/管理员授予 |
| `版主` | 在 `特殊:Moderation` 审稿 | 不参与界面管理 |
| `管理员` | 常规管理全权；**可编辑 Widget** | **不能**编辑全站及他人 CSS/JS |
| `界面管理员` | 全站及他人 CSS/JS | **不能**编辑 Widget |
| `小部件编辑者` | 仅 Widget | — |
| `行政员` | 分配用户组 | **无编辑特权** |
| `监督员` | 删除/恢复特定版本与日志 | — |
| `机器人` | API 上限同管理员；编辑标【机】，默认不进最近更改 | — |

完整用户组（含 SMW 组）与特殊页清单见 `@references/permissions.md`。

**新人门槛**：不属于上述任何用户组时，须先完成 **3 次编辑**才能查看特殊页面。

**匿名态判别**：`action=query&meta=userinfo&uiprop=groups|rights` 返回的 `name` 是出口 IP 而非匿名标记，`anonymous` 也不置位；`rights` 中没有 `edit`。登录后 `name` 为 B 站 UID。每次写操作前先确认身份，不要等提交被拒才发现。

### 3.2 审核（Moderation）

触发条件：编辑内容**疑似**包含「特定词汇」。触发后该修改**只对编辑者自己可见**，其他人（含匿名）看到的仍是旧版本；编辑者可继续编辑该页，不受影响。

处理流程：

1. 判别：登录态能看到、其他登录账号与匿名看不到，且 `特殊:Moderation` 有待审项 → 在审核队列，不是缓存问题。
2. 用 `特殊:Moderation` 查看待审项（需版主或管理员权限）。
3. 等待审核；版主不 24×7 巡查，超时可联系站点编辑组。
4. 队列出现时间通常即时，少数情况约需一天。

绕过：`自动版主化用户`、`管理员`、`机器人` 的编辑不触发审核。

写入前确认自身用户组（`whoami` 含 `groups`），避免把审核延迟误判为写入失败。

## 4. 内容准则

- 定位：游戏百科，不是数据库、攻略站或论坛；资源有限时优先保证主题聚焦。
- 来源优先度：**游戏本体 > 官方公告 > 玩家验证 > 合理推测**。
- **发现错误立即修正，无需等待共识。**
- 关键内容标注来源：具体游戏版本或更新时间。
- 中立观点不完全适用；游戏内容以官方设定为准，攻略与综合分析允许并鼓励主观判断与原创研究。
- 实用主义：内容优先于格式与样式；允许先发「施工中」内容再迭代；优先用模板、批量工具、机器人降低人工。
- 尊重原著与版权：不写外挂、脚本、私服、改动游戏数据；同人内容标注属性；转载注明作者与原始链接。

## 5. 缓存与可见性

缓存分层与刷新方式：

| 层 | 影响对象 | 有效期 | 可刷新 |
|---|---|---|---|
| 浏览器 | 自己 | 本地策略 | 强制刷新 |
| CDN | **匿名访客** | 较长 | **否**（编辑者无权限） |
| 解析器 | 所有人 | 数天；BWiki 可达数月 | 是：保存、`?action=purge`、空编辑 |
| 资源（CSS/JS） | 所有人 | 约 30 分钟 | 否，只能等 |
| URL（`?curid=`、`load.php`） | 通过短链访问者 | 约 30 分钟 | 加随机参数（仅调试用） |

操作规则：

1. **改模板/Widget 不会刷新引用页**。需要立即生效时，对每个引用页执行 purge：
   - 页面右上「Wiki功能 → 刷新」
   - URL 追加 `?action=purge`
   - API：`action=purge&titles=A|B|C`（单次 ≤50 个标题，`forcelinkupdate=1` 同时更新链入表）
2. **改 CSS/JS 后等约 30 分钟**，期间不要重复提交；调试可在控制台用 `&rand=` 加载，但不得写入 `Common.js` 常态加载。
3. **匿名访客的 CDN 缓存无法主动刷新**。验证改动是否生效必须用登录态，或直接看 API 返回。
4. 批量场景用现成工具：`Widget:刷新`（逗号分隔多页）、`MediaWiki:Bikit.js` 的 `Bikit.fixs.purgeCategory`（分类页批量刷新）。
5. 页面渲染结果与源文本不一致时，先 purge 再看；需要不带缓存的渲染结果，用 `action=parse&text=<wikitext>`。
6. 排查顺序：审核队列 → CDN（登录/匿名对比）→ 解析器缓存（purge）→ 资源缓存（等 30 分钟）→ 语法错误（控制台）。

细节与判断树见 `@references/moderation-cache.md`。

## 6. 编辑写法

### 6.1 优先级

1. **Wikitext**：正文、标题、列表、表格、链接、分类、引用、模板调用，全部用 wikitext 表达。
2. **模板 / 模块**：重复结构先查站点现成模板；没有则新建模板或 Lua 模块，不要复制 HTML 片段。
3. **HTML**：仅在 wikitext 无法表达时使用，例如第三方嵌入、必须使用特定标签的交互容器。允许范围受 Sanitizer 限制；`<script>`、`<style>`、`<iframe>` 在普通词条被禁止，这类需求封装为 Widget。
4. **CSS**：样式集中在 `MediaWiki:Common.css`、`#css` 或 `<bstyle>`；不要用行内 `style` 批量替代类。

结构规则：

- 不用 HTML 表格替代 wikitext 表格；不用 `<br>` 堆版式；不用 `<div>` 手工分栏。
- 内容与表现分离：结构写 wikitext，样式写 class。
- 内容模型不是 wikitext 的页面（JS/CSS/JSON/Lua）用对应内容模型，不混写。

### 6.2 语法规则

| 规则 | 写法 |
|---|---|
| 标题 | 正文从 `==` 开始，不跳级；`=` 一级等同页面标题，正文不用 |
| 段落 | 空行分段；段落内不换行 |
| 行首空格 | 整行变成 `<pre>`；正文不以空格开头 |
| 列表 | 行首 `*` `#` `;` `:`；嵌套用连续符号（`**`、`##`）；项间不留空行 |
| 强调 | `'''粗体'''`、`''斜体''`；不用 `<b>`/`<i>` |
| 表格 | 起始、行分隔、结束标记必须在行首；`!` 表头，`\|` 单元格，同行用 `\|\|` 分隔；行/列合并用 `rowspan`/`colspan` |
| 表格内容 | 单元格含字面竖线时用 `<nowiki>` 或 `{{!}}` |
| 内链 | `[[页面]]`、`[[页面\|显示文字]]`、`[[页面#章节]]` |
| 文件 | `[[文件:名称.png\|thumb\|200px\|说明]]`；尺寸、对齐、`alt` 均为选项 |
| 外链 | `[https://example.com 文字]`；非 B 站域名自动加 `linkfilter` 前缀 |
| 引用 | `<ref name=X>内容</ref>` + 文末 `<references/>`；每个 name 必须有定义 |
| 模板 | `{{模板名\|参数=值}}`；模板内参数 `{{{名\|默认}}}` |
| 解析函数 | 单页高开销函数 ≤100 次；预览页底部「解析器分析数据」查用量 |
| 分类 | `[[分类:名称]]`，排序键 `[[分类:名称\|键]]`，全局排序 `{{DEFAULTSORT:键}}` |
| 重定向 | `#REDIRECT [[目标]]` 必须在首行 |
| 转义 | 字面标记用 `<nowiki>`；代码用 `<syntaxhighlight lang="…">` |
| 注释 | `<!-- 注释 -->`，不输出到页面 |
| 源文本 | 读写时模板、Widget、解析器函数保持原样，不手工展开 |
| 体积 | 单页上限 2 MB（`maxarticlesize`） |

完整语法、模板与解析函数清单见 `@references/wikitext.md`。

## 7. 前端能力边界

先确认「能不能做」，再动手。

| 项 | 结论 |
|---|---|
| JS 语法 | **仅 ES5**。`let`/`const`、箭头函数、模板字符串、`class`、`Promise`、`Map`/`Set`、`includes`、展开运算符、`for...of`、默认参数、解构**均不可用** |
| 框架 | **Bootstrap 3.3.7** + **jQuery 3.6**（无 BS4/5、无现代打包链） |
| 样式 | 支持 `#css` 解析函数与 `<bstyle>` 标签；**无 TemplateStyles** |
| CSS 限制 | `#css` 会整段拒绝含 `url()`、`var()` 等关键字的样式（输出 `/* insecure input */`）；需要这些能力用 `<bstyle>` |
| 个人 JS/CSS | **不存在**用户级 JS/CSS 页 |
| Gadget | 脚本文件为 `MediaWiki:Gadget-<name>.js`，定义页 `MediaWiki:Gadgets-definition`；修改需**界面管理员** |
| Widget | 输出绕过 wikitext 解析（组件内 wiki 语法原样输出）；修改需**管理员**或**小部件编辑者** |
| Widget 参数 | Smarty 参数必须转义：`escape:'html'`、`escape:'url'`、`escape:'javascript'`；新组件优先 DOM 传参 |
| 脚本时序 | Widget 脚本可能早于 jQuery 执行，须用 `RLQ` 等待 |
| 外部资源 | 通过 ResourceLoader 加载的 JS/CSS 必须位于 `MediaWiki:` 命名空间 |
| 调试 | 修改后约 30 分钟生效；用 `?action=raw&ctype=text/javascript` 直接取源码核对 |
| 可视化编辑 | 不支持 |

禁用清单、替换写法、Gadget/Widget/RLQ 模板与 XSS 防护见 `@references/frontend.md`。

## 8. 站点探测

接手陌生站点先取一次结构事实：

```bash
curl -s 'https://wiki.biligame.com/<code>/api.php?action=query&format=json&formatversion=2&meta=siteinfo&siprop=general|namespaces' \
  -H 'Referer: https://wiki.biligame.com/<code>/'
```

读法：

- `query.namespaces`：**即使 `formatversion=2` 也返回以 ns id 为键的对象**，用 `Object.values()` / `dict.values()` 遍历。
- `query.pages`：`formatversion=2` 下为数组，可直接 `.find()`。
- 共 **37** 个命名空间；ns6 = `文件`（`媒体文件` 是 ns **-2**）。
- **ns4 本地名 = 站点名前缀，每个站点不同**，必须探测后使用；不要硬编码为 `Project` 或其它站点的名字。
- 固定项：`模板(10)` `文件(6)` `分类(14)` `帮助(12)` `属性(102)` `表单(106)` `概念(108)` `Widget(274)` `模块(828)` `Gadget(2300)` `Topic(2600)`。

全表与探测脚本见 `@references/namespaces.md`。

## 9. API 直连

仅在 MCP 不可用时使用。

1. 请求头同时带 **`Cookie: SESSDATA=<值>`** 与 **`Referer: https://wiki.biligame.com/<code>/`**；缺 Referer 会被当作匿名。
2. 写操作先取 CSRF token：`action=query&meta=tokens&type=csrf` → `query.tokens.csrftoken`。
3. 一律 `format=json`（XML 已弃用）；写请求一律 **POST**（GET 携带长 `text=` 会被代理返回 502）。
4. User-Agent 带联系方式（编辑群昵称），便于平台联系。
5. 优先用库：Python `mwclient`（官方推荐）、`pymediawiki`；Node `mwn`、`mwbot`、`wiki-saikou`。手写 HTTP 仅作最后手段。
6. 串行请求，间隔 **>1 秒**；先 1–2 页验证再放量；用 `list=` 批量取，不循环单发。
7. 写操作带 `baserevid`，冲突时重读合并，不盲目覆盖。
8. 单次 `titles` ≤ **50** 个。
9. 收到 **567 / 514** 立即停手，等 3 分钟以上单发重试。平台文档《异常访问提示》说明该检测针对未登录访问，登录用户不触发。API 与整页导航的恢复时间不同步。

请求样例、token、编辑、purge、上传代码见 `@references/api-fallback.md`。

## 10. 症状索引

| 症状 | 原因 | 动作 |
|---|---|---|
| 我改了，别人看不到 | 审核队列 | 查 `特殊:Moderation`；其他登录账号同样看不到即为审核 |
| 页面内容或样式没更新 | 解析器缓存 | `?action=purge`；模板改动需逐页刷 |
| 匿名访客始终看到旧版 | CDN 缓存 | 无法主动刷新，等待；用登录态验证 |
| CSS/JS 改了没生效 | 资源缓存 30 分钟 | 等待；检查 ES5 语法错误 |
| 编辑报需要 `bilibili` 等用户组 | 匿名态或会话过期 | 重新登录；不要改权限配置 |
| 567 / 514 | EdgeOne 风控 | 停手，登录后等 3 分钟单发 |
| 分类链接是红的 | 分类页实体不存在 | 正常现象；用 `prop=categories` 查归属 |
| 新号打不开特殊页面 | 未满 3 次编辑 | 先完成 3 次编辑 |
| 上传被拒 | 上传目录未配置或不在白名单 | 检查 `uploadDirs` 与绝对路径 |
| 表格内容错位 | 单元格含裸竖线或行间有空行 | 用 `<nowiki>` 包裹；去掉空行 |
| 首页样式错乱 | 重复调用含 `#css`/`<bstyle>` 的模板 | 合并为单次调用（R8） |
| 整站打不开 | `MediaWiki:Sidebar` 含模板语法 | 立即回退为纯链接列表（R6） |

## 11. 实时数据

下列值随平台变化，不写死在正文，按需获取：

| 数据 | 获取方式 |
|---|---|
| 站点名册 | `list-wikis` |
| 命名空间 | §8 的 `siteinfo` 请求 |
| 引擎版本、扩展清单 | `siteinfo` 的 `general.generator` 与 `extensions` |
| 工具清单与参数 | `bwiki-mcp`，以服务端返回的工具定义为准 |
| 规则页出处 | `@references/sources.md` |

## 12. 外部文档

| 文档 | 地址 |
|---|---|
| MediaWiki 手册 | <https://www.mediawiki.org/wiki/Help:Contents> |
| Action API | <https://www.mediawiki.org/wiki/API:Main_page> |
| Extension:CSS | <https://www.mediawiki.org/wiki/Extension:CSS> |
| Extension:Widgets | <https://www.mediawiki.org/wiki/Extension:Widgets> |
| Extension:Gadgets | <https://www.mediawiki.org/wiki/Extension:Gadgets> |

链接本身规范有效；网络不可达不代表链接失效。
