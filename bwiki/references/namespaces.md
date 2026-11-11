# 命名空间

## 事实

- 共 **37** 个命名空间。
- ns6 本地名是 **`文件`**；`媒体文件` 是 ns **-2**（Media），用于直接嵌入媒体。
- **ns4（项目空间）本地名 = 站点名前缀，每个站点不同**，必须探测后使用。
- 其余命名空间本地名全农场一致。

## 全表

| id | 本地名 | canonical | 用途 |
|---|---|---|---|
| -2 | 媒体文件 | Media | 直接引用原始文件 |
| -1 | 特殊 | Special | 特殊页面 |
| 0 | （无前缀） | — | 条目 |
| 1 | 讨论 | Talk | 主空间讨论 |
| 2 | 用户 | User | 用户页 |
| 3 | 用户讨论 | User talk | — |
| 4 | **站点名** | Project | 项目页（每站不同） |
| 5 | 站点名讨论 | Project talk | — |
| 6 | **文件** | File | 文件描述页 |
| 7 | 文件讨论 | File talk | — |
| 8 | MediaWiki | MediaWiki | 界面消息、站点 CSS/JS |
| 9 | MediaWiki讨论 | MediaWiki talk | — |
| 10 | 模板 | Template | — |
| 11 | 模板讨论 | Template talk | — |
| 12 | 帮助 | Help | — |
| 13 | 帮助讨论 | Help talk | — |
| 14 | 分类 | Category | — |
| 15 | 分类讨论 | Category talk | — |
| 102 | 属性 | 属性 | SMW 属性 |
| 103 | 属性讨论 | 属性讨论 | — |
| 106 | 表单 | Form | Page Forms |
| 107 | 表单讨论 | Form talk | — |
| 108 | 概念 | 概念 | SMW 概念 |
| 109 | 概念讨论 | 概念讨论 | — |
| 112 | smw/schema | smw/schema | SMW schema |
| 113 | smw/schema talk | smw/schema talk | — |
| 114 | Rule | Rule | 规则 |
| 115 | Rule talk | Rule talk | — |
| 274 | Widget | Widget | Widget 扩展 |
| 275 | Widget talk | Widget talk | — |
| 828 | 模块 | Module | Scribunto |
| 829 | 模块讨论 | Module talk | — |
| 2300 | Gadget | Gadget | 小工具定义 |
| 2301 | Gadget talk | Gadget talk | — |
| 2302 | Gadget definition | Gadget definition | — |
| 2303 | Gadget definition talk | Gadget definition talk | — |
| 2600 | Topic | Topic | 结构化讨论 |

## 探测

```bash
curl -s 'https://wiki.biligame.com/<code>/api.php?action=query&format=json&formatversion=2&meta=siteinfo&siprop=general|namespaces' \
  -H 'Referer: https://wiki.biligame.com/<code>/'
```

```python
import json, urllib.request

code = "<站点代号>"
url = (f"https://wiki.biligame.com/{code}/api.php?action=query&format=json"
       "&formatversion=2&meta=siteinfo&siprop=general|namespaces")
req = urllib.request.Request(url, headers={
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) Chrome/120 Safari/537.36",
    "Referer": f"https://wiki.biligame.com/{code}/",
})
data = json.load(urllib.request.urlopen(req, timeout=30))
ns = data["query"]["namespaces"]           # dict，键为字符串 id
for n in ns.values():                       # 不要用 .find()
    print(n["id"], repr(n["name"]), n.get("canonical"))
```

## 解析注意

| 参数 | `query.namespaces` | `query.pages` |
|---|---|---|
| `formatversion=2` | 仍是**对象**（键为 ns id） | **数组** |
| `formatversion=1` | 对象 | 对象 |

- 取列表用 `Object.values(...)` / `dict.values()`；`Array.prototype.find` 只适用于 `pages`。
- `namespacealiases` 不含规范本地名（`特殊` 是规范名，不是别名），搜不到属正常。
- 命名空间 id 在同一农场内稳定，可作为跨站脚本的常量；本地名不要当常量。

## 常用 id

| 用途 | id |
|---|---|
| 条目 | 0 |
| 模板 | 10 |
| 文件 | 6 |
| 分类 | 14 |
| 帮助 | 12 |
| Widget | 274 |
| 模块 | 828 |
| Gadget | 2300 |
| MediaWiki 界面页 | 8 |
