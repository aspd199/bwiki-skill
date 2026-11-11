# bwiki-skill

面向 AI 编码代理的 **BWiki（wiki.biligame.com）技能包**：一套平台规范，一份 MCP 工具手册。

装上它，代理就知道 BWiki 的硬约束红线、权限与审核机制、缓存分层、wikitext 写法、前端能力边界，以及怎么用 `bwiki-mcp-server` 读写页面、上传文件、查 SMW 数据。

## 包含什么

| 目录 | 技能 ID | 作用 |
|---|---|---|
| `bwiki/` | `bwiki` | 平台规范：硬约束、权限与审核、内容准则、缓存与可见性、编辑语法、前端能力、站点探测、API 直连兜底 |
| `bwiki-mcp/` | `bwiki-mcp` | `bwiki-mcp-server` 调用手册：31 个工具的参数契约、写入细则、错误分诊 |

两个技能互相引用：写页面时先由 `bwiki` 判断「能不能做、会不会把站弄坏」，再由 `bwiki-mcp` 执行具体调用。

### 目录结构

```text
bwiki-skill/
├── bwiki/
│   ├── SKILL.md                  # 入口：路径选择、平台模型、硬约束、语法、症状索引
│   └── references/
│       ├── wikitext.md           # 完整语法、模板、解析函数清单
│       ├── frontend.md           # CSS/JS/Widget/Gadget/RLQ、XSS 防护
│       ├── permissions.md        # 用户组与特殊页
│       ├── namespaces.md         # 37 个命名空间全表与探测脚本
│       ├── moderation-cache.md   # 审核队列与缓存判断树
│       ├── api-fallback.md       # 绕过 MCP 直连 Action API 的代码
│       └── sources.md            # 规则出处
└── bwiki-mcp/
    └── SKILL.md                  # 工具目录、标准作业流程、写入细则、错误分诊
```

## 安装

技能的目录名就是技能 ID，直接把两个目录整体拷进任一被识别的 skills 路径即可。

**OpenCode**（自动搜索以下位置，无需改配置）

| 范围 | 路径 |
|---|---|
| 全局 | `~/.config/opencode/skills/` |
| 项目 | `.opencode/skills/` |
| 兼容 | `~/.claude/skills/`、`~/.agents/skills/`、项目内同名目录 |

```powershell
# 全局安装
Copy-Item -Recurse .\bwiki, .\bwiki-mcp "$HOME\.config\opencode\skills\"

# 项目安装
Copy-Item -Recurse .\bwiki, .\bwiki-mcp .\.opencode\skills\
```

**Claude Code**

```bash
cp -r bwiki bwiki-mcp ~/.claude/skills/   # 全局
cp -r bwiki bwiki-mcp .claude/skills/     # 项目
```

也可以在 `opencode.jsonc` 里把本仓库目录加为技能来源：

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "skills": ["D:/Git/bwiki-skill"]
}
```

## 使用

安装后无需手动点名，代理会根据任务描述自动加载；也可以显式指定：

```text
@bwiki 帮我看下这个站点的命名空间前缀和站点规模
@bwiki-mcp 读取 MediaWiki:Common.css 并加一条 .infobox 样式
```

典型任务与去向：

| 任务 | 加载 |
|---|---|
| 读/写/移动/删除/上传页面 | `bwiki-mcp` |
| 判断某个交互效果能不能做 | `bwiki` §7 |
| 写完别人看不到、页面不更新 | `bwiki` §5 |
| 改 CSS/JS/Widget/Gadget | `bwiki` §7 + `references/frontend.md` |
| 接手陌生站点 | `bwiki` §8 站点探测 |
| MCP 不可用，需直连 API | `bwiki` §9 + `references/api-fallback.md` |
| MCP 报错定位 | `bwiki-mcp` §5 错误分诊 |

## 前提

- `bwiki-mcp` 技能依赖 **[bwiki-mcp-server](https://github.com/aspd199/bwiki-mcp-server)**（BWiki 的 MCP 服务器）。该服务需要 B 站账号会话：先运行 `bwiki-login` 打开官方登录页，完成后会话写入用户配置目录下的 `bwiki-mcp-server/session.json`。
- `bwiki` 技能只含平台知识，不依赖 MCP，可单独使用；需要执行时按其 §9 直连 Action API。

## 注意

- **写入即代表账号行为。** 操作日志记录为 `bilibili:<UID>`，删除、移动、保护只在明确要求时执行。
- **会话文件是凭证。** `session.json` 按密码对待，不复制、不提交、不分享。
- **平台规则会变。** 站点名册、命名空间、引擎版本、扩展清单等实时数据不写死在技能正文，按 `bwiki` §11 现场获取。
- 本仓库是使用手册与规范整理，内容依据见 `bwiki/references/sources.md`；与官方规则冲突时以站点规则页为准。

## 许可

[MIT](./LICENSE) © aspd199
