# 审核与缓存：可见性排查

## A. 改了但别人看不到

```
登录态能看到新内容？
├─ 否 → 写入是否成功？（看编辑返回、最近更改、特殊:Moderation）
└─ 是 → 其他人（含匿名、另一登录账号）看不到？
         ├─ 是 → 审核队列（A1）
         └─ 否 → 对方本地缓存或 CDN，让其强刷
```

### A1. 审核队列

判别（按可靠性排序）：

1. `特殊:Moderation` 中存在该次编辑 → 审核队列。
2. 换一个登录账号能看到 → 不是审核，是 CDN 或对方缓存。
3. 只有作者本人能看到、其他登录账号也看不到 → 审核队列。
4. 作者退出登录后看不到 → 可能是审核，也可能是 CDN，回到第 1 条确认。

事实：

- 触发条件是内容**疑似**含「特定词汇」，与账号、页面、命名空间无关。
- 入队后修改只对编辑者可见，编辑者可继续编辑，不产生冲突。
- 队列入口 `特殊:Moderation`，需要 `版主` 或 `管理员`。
- 通常即时入队，少数情况约一天；`自动版主化用户`、`管理员`、`机器人` 不触发。
- 通过后全站可见，仍受 CDN 与解析器缓存影响（见 B 节）。

处理：

1. 确认自身不在绕过组（`whoami` 的 `groups`）。
2. 到 `特殊:Moderation` 查待审项；无权限则联系站点版主或编辑组。
3. 等待审核；不要反复提交同一修改。
4. 审核通过后若仍不见更新，按 B 节 purge。

### A2. 不是审核的两种情况

| 现象 | 判断 | 处理 |
|---|---|---|
| 自己刷新才看到，别人稍后也能看到 | CDN 未过期 | 等待；无法由编辑者 purge |
| 所有访问者都看不到新内容 | 解析器缓存，或写入未生效 | purge 目标页；仍无则回查编辑结果与审核队列 |

## B. 页面内容没更新

```
改的是哪类页面？
├─ 模板 / Widget / 模块 → 引用页不会自动刷新 → 逐页 purge
├─ 普通条目 → 目标页 purge
├─ CSS / JS → 等约 30 分钟，无法 purge
└─ 首页 → 检查是否命中 R8（重复调用含 #css / <bstyle> 的模板）
```

### B1. 缓存分层

| 层 | 影响 | 有效期 | 刷新手段 |
|---|---|---|---|
| 浏览器 | 当前设备 | 本地策略 | 强制刷新 |
| CDN | 匿名访客 | 较长，未登录缓存优先 | **编辑者无法刷新** |
| 解析器 | 所有人 | 数天；BWiki 可达数月 | 保存、`?action=purge`、空编辑 |
| 资源（CSS/JS/多媒体） | 所有人 | 代码约 30 分钟；多媒体即时 | 只能等 |
| URL（`?curid=`、`load.php`） | 短链访问者 | 约 30 分钟 | 加随机参数（调试用） |

登录用户的 CDN 缓存极短；极端情况下会出现客户端「串缓存」（`RLCONF` 里的组件配置属于其他用户），不影响使用。

### B2. purge 方式

| 方式 | 操作 | 适用 |
|---|---|---|
| 界面 | 页面右上「Wiki功能 → 刷新」 | 单页 |
| URL | `<page>?action=purge` | 单页 |
| API | `action=purge&titles=A\|B\|C`（≤50 个标题） | 批量 |
| 空编辑 | 不改内容直接保存 | 兜底 |
| `Widget:刷新` | 页面内填多个标题（英文逗号或顿号分隔） | 站点已装该 Widget |
| `MediaWiki:Bikit.js` | `Bikit.fixs.purgeCategory` 给分类页加批量刷新按钮 | 分类批量 |
| 油猴脚本 | BWiki 工具列表 / 快捷菜单栏 | 手动运维 |

API 示例：

```bash
curl -s -X POST 'https://wiki.biligame.com/<code>/api.php' \
  -H 'Cookie: SESSDATA=<值>' \
  -H 'Referer: https://wiki.biligame.com/<code>/' \
  --data-urlencode 'action=purge' \
  --data-urlencode 'format=json' \
  --data-urlencode 'titles=模板:示例|页面A|页面B' \
  --data-urlencode 'forcelinkupdate=1'
```

`forcelinkupdate=1` 同时刷新链入表（`templatelinks` 等），模板类改动建议加。

### B3. 验证

1. `action=query&prop=revisions&rvprop=content` 读源文本：确认新内容已保存。若该编辑在审核队列，源文本仍是旧版，审核通过后才会更新。
2. `action=parse&text=<wikitext>` 渲染：确认语法输出正确（不依赖页面缓存）。
3. 登录态访问页面：确认渲染结果。
4. 匿名访问：确认 CDN 是否仍为旧版（可能滞后，不作为写入失败依据）。

### B4. CSS/JS 专项

- 修改 `MediaWiki:Common.js`、`MediaWiki:Common.css`、`MediaWiki:Gadget-*.js`、站点 CSS 后，资源缓存约 30 分钟。
- 调试期可在控制台加载 `...?action=raw&ctype=text/javascript&rand=<随机数>` 立即验证；不得把随机数写进常态加载。
- 语法错误会导致整段脚本不执行，先看浏览器控制台。
- 确认脚本是否被加载：查看页面源码中的 `load.php` 引用，或直接请求 `?action=raw`。

### B5. 分类页

- 分类归属真实，但分类页实体可能不存在，底部红链是常态。
- 判断归属用 `action=query&prop=categories&titles=<页面>`。
- 分类页内容本身也可能被缓存，批量改分类后用 `Bikit.fixs.purgeCategory` 或逐页 purge。
