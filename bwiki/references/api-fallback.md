# API 直连参考

仅在 MCP 不可用时使用。所有示例针对 `https://wiki.biligame.com/<code>/api.php`。

## 1. 请求头

| 头 | 值 | 说明 |
|---|---|---|
| `Cookie` | `SESSDATA=<值>` | B 站会话；等同密码，不入库、不入日志 |
| `Referer` | `https://wiki.biligame.com/<code>/` | **必需**，缺失会被当作匿名 |
| `User-Agent` | 自定，带联系方式（如编辑群昵称） | 官方建议 |
| `Content-Type` | `application/x-www-form-urlencoded;charset=UTF-8` | 写请求用表单 |

BWiki 不提供 API 登录，只能复用浏览器会话。SESSDATA 在浏览器开发者工具 → Application → Cookies → `wiki.biligame.com` 中取。

## 2. 最小客户端

```python
import requests

class BWiki:
    def __init__(self, code: str, sessdata: str):
        self.api = f"https://wiki.biligame.com/{code}/api.php"
        self.s = requests.Session()
        self.s.cookies.set("SESSDATA", sessdata, domain="wiki.biligame.com", path="/")
        self.s.headers.update({
            "Referer": f"https://wiki.biligame.com/{code}/",
            "User-Agent": "MyWikiBot/1.0 (contact: <你的联系方式>)",
        })
        self._token = None

    def get(self, **params):
        params.update(format="json", formatversion=2)
        r = self.s.get(self.api, params=params, timeout=30)
        r.raise_for_status()
        return r.json()

    def post(self, **data):
        data.update(format="json", formatversion=2)
        r = self.s.post(self.api, data=data, timeout=60)
        r.raise_for_status()
        return r.json()

    @property
    def token(self) -> str:
        if not self._token:
            self._token = self.get(action="query", meta="tokens", type="csrf")["query"]["tokens"]["csrftoken"]
        return self._token
```

## 3. 读页面

```python
data = wiki.get(
    action="query",
    prop="revisions",
    rvprop="content|ids|timestamp",
    rvslots="main",
    titles="页面A|页面B",          # 单次 ≤50
)
for page in data["query"]["pages"]:   # formatversion=2 下为数组
    if page.get("missing"):
        continue
    rev = page["revisions"][0]
    print(page["title"], rev["revid"], rev["slots"]["main"]["content"])
```

## 4. 写页面

```python
res = wiki.post(
    action="edit",
    title="页面A",
    text=wikitext,
    summary="更新数据",
    token=wiki.token,
    baserevid=latest_revid,   # 冲突检测
    bot=True,                 # 需要机器人权限
)
```

返回 `edit.result == "Success"` 视为成功；同时检查 `edit.newrevid`。

冲突：传入的 `baserevid` 不是最新版时返回 `editconflict`，应重新读取、合并后重写。

新建页面：`createonly=True`；页面已存在会失败。

## 5. 预渲染校验

```python
html = wiki.post(
    action="parse",
    text=wikitext,
    title="页面A",              # 提供上下文标题，影响 {{PAGENAME}} 等
    contentmodel="wikitext",
    prop="text",
)["parse"]["text"]
```

`text=` 形式不复用目标页的解析器缓存，适合上线前验证模板、表格、引用。

## 6. 刷新缓存

```python
wiki.post(
    action="purge",
    titles="模板:示例|页面A|页面B",   # ≤50
    forcelinkupdate=1,
)
```

模板改动后必须对每个引用页执行；匿名无法刷新 CDN，验证时用登录态。

## 7. 分类、搜索、链入

```python
# 分类成员（单次 ≤500，cmcontinue 翻页）
wiki.get(action="query", list="categorymembers", cmtitle="分类:角色", cmlimit=500)

# 页面所属分类
wiki.get(action="query", prop="categories", titles="页面A", cllimit="max")

# 全文搜索
wiki.get(action="query", list="search", srsearch="关键词", srlimit=50, srnamespace=0)

# 链入
wiki.get(action="query", list="backlinks", bltitle="模板:信息框", bllimit=500)
```

## 8. 上传

```python
with open(path, "rb") as f:
    res = wiki.s.post(wiki.api, data={
        "action": "upload",
        "filename": "示例.png",
        "comment": "上传图片",
        "token": wiki.token,
        "ignorewarnings": 1,
        "format": "json",
    }, files={"file": f}, timeout=120).json()
```

同名文件用 `action=upload` 即覆盖为新版本，需文件版本权限。

## 9. 错误处理

| 错误码 / 状态 | 含义 | 处理 |
|---|---|---|
| HTTP 567 / 514 | EdgeOne 风控 | 停手 3 分钟以上，登录后单发重试 |
| `badtoken` / `notoken` | CSRF 失效 | 重新取 token 重试一次 |
| `protectedpage` | 页面被保护 | 检查账号权限或改走讨论页 |
| `permissiondenied` | 无对应权限 | 换账号或申请权限 |
| `editconflict` | 基线版本过期 | 重读合并 |
| `missingtitle` | 页面不存在（`nocreate`） | 改用 `createonly` |
| `articleexists` | 页面已存在 | 改用 `action=edit` |
| `ratelimited` | 触发限流 | 退避后重试 |
| `readonly` | 站点维护中 | 稍后重试 |
| HTTP 502（长 GET） | 代理拒绝长 URL | 改用 POST |

## 10. 频率与批量

- 串行请求，间隔 **>1 秒**；不要并发打同一站点。
- 先 1–2 页验证，再放量。
- 批量取数用 `list=`、`titles=`、`generator=`，不要循环单发。
- 分类成员、页面清单等结果本地缓存复用。
- 机器人权限可将编辑标 `bot=True`，默认不进最近更改。

## 11. 库

| 语言 | 库 | 说明 |
|---|---|---|
| Python | `mwclient` | 官方推荐，封装完整 |
| Python | `pymediawiki` | SMW 支持更好 |
| Node | `mwn` | 功能最全，支持 TypeScript |
| Node | `mwbot` / `wiki-saikou` | 轻量 |
| 浏览器 | `mw.Api()` | 复用登录态，无文件上传能力 |

使用库时仍需注入 `SESSDATA` cookie 与 `Referer` 头；各库注入方式不同，以所用版本文档为准。浏览器控制台内可直接使用 ES6+ 语法，与站点 `Common.js` 的 ES5 限制无关。

## 12. 站点差异

- `wiki.biligame.com` 是所有站点的同一 host，按路径第一段区分。
- 命名空间 id 全农场一致，本地名不一定（ns4 每站不同）。
- 权限按站点独立，同一账号在不同站点的用户组不同。
