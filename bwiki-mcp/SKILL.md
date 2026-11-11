---
name: bwiki-mcp
description: bwiki-mcp-server 的调用手册：31 个 BWiki 读写工具、参数契约、身份校验、节流与错误分诊。当需要用 MCP 工具读写 wiki.biligame.com 的页面、文件、分类、SMW 数据，或排查 MCP 返回的错误时加载。平台规范另见 bwiki 技能。
---

# bwiki-mcp-server 调用手册

服务器：`bwiki-mcp-server`（v1.1.0，31 个工具）。以已保存的 B 站账号身份读写整个 BWiki 农场。

## 1. 前置约定

| 项 | 约定 |
|---|---|
| 站点参数 | 所有 wiki 工具都有可选 `wiki`，传站点代号；省略用配置中的默认站（`wikis` 数组第一项） |
| 默认站 | 不确定当前默认站时，先 `list-wikis` 看 `default` 标记 |
| 响应 | 同时返回叙述文本与 `structuredContent`（JSON），每条响应注明本次作用的站点 |
| 会话 | 只有 `bwiki-login` 无需会话；其余工具在无会话时拒绝服务，**不降级为匿名** |
| 写操作 | 每次写入前校验身份；身份失效则拒绝并断开，不会以 IP 身份写入 |
| 节流 | 服务端串行，相邻请求间隔 ≥1.1 秒；收到 567/514 自动冷却 180 秒 |
| 体积 | 单次响应正文上限 75000 字节（`MCP_CONTENT_MAX_BYTES`），超出截断并提示可缩小范围 |
| 上传 | 仅允许 `uploadDirs` 白名单内的绝对路径；未配置则上传工具直接拒绝 |

会话文件由服务器写入用户配置目录下的 `bwiki-mcp-server/session.json`。按凭证对待，不复制、不提交。

## 2. 标准作业流程

```
0  确定站点：list-wikis（或已知代号）
1  确认身份：whoami（含 groups；写前必做）
2  读取现状：get-page（metadata=true 取 latestRevisionId 与章节大纲）
3  预演渲染：parse-wikitext（可选，校验模板/表格/引用）
4  写入：update-page / create-page（带 comment；改已有页带 latestId）
5  验证：get-page 复核；若改的是模板或 CSS/JS，按 bwiki §5 处理缓存
```

写操作三原则：目标站明确、身份明确、基线版本明确。

## 3. 工具目录

### 3.1 读取（17）

| 工具 | 用途 | 关键参数 |
|---|---|---|
| `get-page` | 取单页 | `title`、`content=source\|html\|none`、`metadata`、`section` |
| `get-pages` | 批量取页 | `titles`（1..50）、`content`、`metadata`、`followRedirects`（默认 true） |
| `get-revision` | 取指定修订 | `revisionId`、`content`、`metadata` |
| `get-page-history` | 页面修订列表 | `title`、`olderThan`、`newerThan`、`filter`（变更标签） |
| `list-pages` | 按命名空间枚举全部页面 | `namespace`（数字 ID）、`from`/`to`（闭区间）、`direction`、`filter=all\|redirects\|nonredirects`、`limit≤500`、`continueFrom` |
| `get-links-here` | 链入/嵌入/文件使用 | `title`、`type=wikilinks\|transclusions\|fileusage`、`namespaces`、`filter`、`expandRedirects`、`limit≤500`、`continueFrom` |
| `get-category-members` | 分类成员 | `category`（可省 `Category:`）、`types`、`namespaces`、`limit≤500`、`continueFrom` |
| `search-page` | 全文搜索 | `query`、`limit≤100`、`namespaces` |
| `search-page-by-prefix` | 标题前缀搜索 | `prefix`、`limit`、`namespace` |
| `get-recent-changes` | 全站最近更改 | `since`/`until`（ISO 8601）、`namespace`、`types`、`user`/`excludeUser`、`tag`、`hideBots`/`hideMinor`/`hideAnon`/`hideRedirects`、`continue` |
| `get-site-info` | 站点配置 | `includeStatistics`；返回版本、语言、命名空间、扩展、许可、`maxarticlesize` |
| `get-file` | 文件页 | `title`（可省 `File:`） |
| `get-file-data` | 取图片字节并内联返回 | `title`、`width`（默认 1024，text 模式 512，上限 1568）、`format=image\|text`（默认 `image`） |
| `compare-pages` | 版本对比 | `fromTitle`/`fromRevision`/`fromText` 与 `toTitle`/`toRevision`/`toText` 二选一，`includeDiff` |
| `parse-wikitext` | 预渲染 wikitext | `wikitext`、`title`（上下文标题）、`applyPreSaveTransform`（默认 true） |
| `list-wikis` | 农场名册 | `limit≤500`、`query`（名称/代号模糊）、`configuredOnly` |
| `whoami` | 当前身份 | `includeRights` |

### 3.2 写入（10）

| 工具 | 用途 | 关键参数 |
|---|---|---|
| `create-page` | 新建页面 | `title`、`source`、`comment`、`contentModel`（如 `javascript`、`css`）、`bot` |
| `update-page` | 改写已有页 | `title`、`operation=replace\|append\|prepend\|find-replace`、`source`、`find`/`replaceWith`、`section`、`latestId`、`removeSubsections`、`removeUnreadContent`、`comment`、`bot` |
| `move-page` | 移动/重命名 | `fromTitle`、`toTitle`、`moveTalk`、`moveSubpages`、`leaveRedirect`、`ignoreWarnings`、`comment` |
| `delete-page` | 删除 | `title`、`comment` |
| `undelete-page` | 恢复 | `title`、`comment` |
| `protect-page` | 保护 | `title`、`protections`（如 `{"edit":"autoconfirmed"}`，`all` 解除）、`expiry`（默认 `infinite`）、`cascade`、`comment` |
| `upload-file` | 上传本地文件 | `filepath`、`title`、`text`、`comment` |
| `update-file` | 覆盖为新版本（本地） | `filepath`、`title`、`comment` |
| `upload-file-from-url` | 从 URL 上传 | `url`、`title`、`text`、`comment` |
| `update-file-from-url` | 从 URL 覆盖 | `url`、`title`、`comment` |

### 3.3 会话（2）

`bwiki-login` 打开官方 B 站登录页，等待完成后校验并保存会话；`bwiki-logout` 删除会话文件。

### 3.4 SMW（2，仅装有 SMW 的站点注册）

`smw-query`（`query` 为 `#ask` 查询串，`limit`、`continueFrom`）、`smw-list-properties`（`search`、`limit`、`continueFrom`）。

## 4. 写入细则

### 4.1 局部改写

- `operation=find-replace`：`find` 必须**逐字唯一匹配**（含空白与模板标记）；匹配到 0 处或多处都拒绝写入，需扩展到上下文唯一定位。
- `operation=replace` 配合 `section=N`：只替换该节，并**连带删除其下所有子节**；`append` 写到该节末尾，`prepend` 插到该节标题之前。
- `section` 必须与 `latestId` 同时使用（wiki 按该修订解析节号）；`latestId` 取自 `get-page(metadata=true)` 的 `latestRevisionId`。
- 不带 `section` 时可省略 `latestId`，但即失去冲突检测；并发编辑场景应显式传入。
- 目标页字节数超过单次可读上限、而 `source` 更短时，`replace` 需显式 `removeUnreadContent=true` 确认丢弃未读部分。
- 新增章节用 `append`，`source` 以空行开头，如 `"\n\n== 历史 ==\n\n内容"`；不以换行开头会被并到最后一行。

### 4.2 新建

`create-page` 一次写入整页，上限为 wiki 自身 `maxarticlesize`（BWiki 2 MB）。只有超过该上限才用 `create-page` + 连续 `update-page(append)` 分块。标题已存在时失败，改用 `update-page`。

### 4.3 移动与保护

- `move-page` 默认行为由 `leaveRedirect`、`moveSubpages`、`moveTalk` 控制；跨命名空间移动前确认目标命名空间允许该内容模型。
- `protect-page` 级别使用站点自身级别，常见 `autoconfirmed`（半保护）、`sysop`（全保护）；`{"edit":"all"}` 解除。需要对应管理权限。

### 4.4 上传

- 文件路径必须是绝对路径、位于 `uploadDirs` 白名单（或 `MCP_UPLOAD_DIRS`）之内，且经 `realpath` 解析后仍在白名单内；白名单内指向外部的符号链接会被拒绝。
- URL 上传不受目录限制，但受 SSRF 防护：解析到私有或回环地址的目标拒绝。
- 图片改名不影响已有引用；覆盖同名文件用 `update-file`。

## 5. 错误分诊

| 返回 | 含义 | 动作 |
|---|---|---|
| 提示需要登录 / 会话无效 | 无凭证或会话失效 | `bwiki-login` 重新登录；不要反复重试写操作 |
| `567` / `514` | EdgeOne 拦截 | 立即停手，等 3 分钟以上单发；确认登录态 |
| `missingtitle` | 页面不存在 | 改用 `create-page` |
| 编辑冲突 / `editconflict` | `latestId` 过期 | 重读页面，合并后再写 |
| 内容被截断（含截断标记） | 超过 75000 字节 | `get-page(section=N)` 或 `content=none` 取大纲后逐节读 |
| `uploads are disabled` | 未配置 `uploadDirs` | 配置允许目录或改用 URL 上传 |
| `badtoken` | CSRF 令牌失效 | 重新调用（服务端自动取新 token） |
| 权限不足 | 账号在该站无相应用户组 | 查该站用户组；不要重复重试 |

## 6. 边界

- 写入即代表账号行为，操作日志记录 `bilibili:<UID>`；删除、移动、保护仅在明确要求时执行。
- 编辑内容以 **wikitext** 为主，HTML 仅在 wikitext 无法表达时使用（见 `bwiki` §6）。
- 不修改 `MediaWiki:Sidebar` 的模板语法、不在首页重复调用含 `#css`/`<bstyle>` 的模板（见 `bwiki` §2）。
- 读取优先 `get-page`；需要全量清单用 `list-pages`，不要用 `search-page` 替代（全文索引不覆盖全部页面）。
- 同一批任务内复用已取到的数据（分类成员、页面清单），不要重复拉取。
