# 规则出处与取值

## 规则出处

| 规则 | 平台文档 |
|---|---|
| 平台红线（自动播放、遮罩、弹框、闪烁、嵌套、Sidebar、二维码、`#css` 重复调用、半角 `%`） | 《BWIKI平台技术规范初稿》 |
| 社区行为规范、转载与署名 | 《WIKI社区规则》 |
| 用户组、权限边界、审核机制 | 《用户组和权限管理》 |
| 平台特性（1.37、ES5、Bootstrap、jQuery、无个人 JS/CSS、无字词转换、Cookie+Referer） | 《快速上手（进阶版）》 |
| 缓存分层、purge、资源缓存 | 《缓存》 |
| 内容准则（来源优先、立即修正、中立不适用、实用主义） | 《基本原则》 |
| 风控提示（风险行为判定、登录用户不触发） | 《异常访问提示》 |
| 机器人路线、库选择、UA 与频率 | 《机器人》 |
| API 最小实现、CSRF token | 《基于Python的API示例》 |
| 短链有效期 | 《常见问题》 |
| 外链 `linkfilter`、跨 wiki 禁用 | 《链接》 |
| `#css` 清洗规则、`<bstyle>` 差异、去重写法 | 《帮助:解析函数/css》 |
| Widget 权限、输出机制、Smarty 转义、DOM 传参 | 《Widgets》 |
| Gadget 定义格式与选项 | 《小工具》 |
| ES5 禁用清单 | 《在BWIKI中使用JS》 |
| 高开销解析函数上限 | 《帮助:解析函数/ifexist》 |
| 文件 URL 推导 | 《BWiki文件URL路径计算规则》 |
| 工具清单、参数、上传白名单、会话行为、节流 | MCP 服务器的 README 与源码 |

## 实时取值

```bash
# 命名空间
curl -s 'https://wiki.biligame.com/<code>/api.php?action=query&format=json&formatversion=2&meta=siteinfo&siprop=namespaces' \
  -H 'Referer: https://wiki.biligame.com/<code>/'

# 引擎版本与扩展
curl -s 'https://wiki.biligame.com/<code>/api.php?action=query&format=json&formatversion=2&meta=siteinfo&siprop=general|extensions' \
  -H 'Referer: https://wiki.biligame.com/<code>/'

# 规则页正文
curl -s 'https://wiki.biligame.com/<code>/api.php?action=query&format=json&formatversion=2&prop=revisions&rvprop=content&rvslots=main&titles=<页面名>' \
  -H 'Referer: https://wiki.biligame.com/<code>/'
```

- 站点名册：`list-wikis`。
- 工具与参数：以 MCP 服务端注册的工具定义为准。
- 规则页正文可能被缓存，取最新版本前先 `action=purge`。
