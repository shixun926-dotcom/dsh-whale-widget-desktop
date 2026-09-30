# dsh-whale-widget 桌面端（DSH Desktop / Electron）适配补丁

**补丁文件**：`lib/index.js`（`apply()` 末尾，`tapIndex` 注册之后）
**原始版本**：dsh-whale-widget v0.2.5（原始 zip 仍在 `D:\Downloads\dsh-whale-widgetv0.2.5.zip`，可直接解压回滚）
**症状**：网页端小鲸鱼挂件正常，**桌面端右下角完全不出现**（无报错、无日志）。

## 根因（已实测确认）

DSH 的 index 注入分两层（`@deepseek-ai/dsh-host-webserver/README.zh.md:51`）：

1. **结构化注入行**：`collectIndexInjections()` 每次发一次 `webserver/index-inject` 事件，
   订阅方推入自己的行（`kind` 为 `global` / `script` / `script-src` / `script-preload` / `style` / `html`）；
2. **原始 `tapIndex` 转换**：`renderIndex(html)` 在渲染完行之后再按注册顺序应用。

关键差别在"静态部署"：

- **网页端**：index.html 由宿主 web server 现场渲染 → 两层都会生效；
- **桌面端**：index.html 是 `dsh-web-frontend/dist` 里的静态文件，由 Electron 主进程直接返回
  （`lib/main.js:11546`），宿主只把 **`collectIndexInjections()` 的行**塞进 `dsh-desktop:boot`
  的 payload（`dsh-desktop-host/lib/index.js:341`）。渲染器的应用函数只认行类型：

  ```js
  // dsh-web-frontend/dist/assets/index-*.js
  case "script-src": await n(o.src); break
  case "script-preload": break
  case "html": (...).insertAdjacentHTML("beforeend", o.html); break
  default: fM(o)   // 未知行类型直接抛错
  ```

  它**从不执行 `tapIndex` 的转换**。

而 v0.2.5 只用 `tapIndex` 注入 `<script defer src="/dsh-whale/widget.js">`，所以在桌面端
这个 script 标签永远不会进入文档，挂件自然不出现。

## 修复

在保留原 `tapIndex`（网页端行为不变）的同时，再注册一行结构化注入：

```js
disposers.push(ctx.on('webserver/index-inject', (table) => {
  if (!Array.isArray(table)) return
  if (table.some((row) => row && typeof row === 'object' &&
    String(row.html || row.text || row.src || '').indexOf('/dsh-whale/widget.js') !== -1)) return
  table.push({
    kind: 'script',
    placement: 'body',
    text: "(function(){if(document.querySelector('script[data-dsh-whale-widget]'))return;" +
      "var s=document.createElement('script');s.src='/dsh-whale/widget.js';s.defer=true;" +
      "s.setAttribute('data-dsh-whale-widget','');document.body.appendChild(s)})()",
  })
}))
```

设计要点：

- **用 `script` 行而不是 `script-src`**：网页端的 `script-src` 行会在 `<body>` 开头插入阻塞脚本，
  早于应用 DOM 解析；内联启动器动态插入外部脚本，等价于原来的 `defer` 语义（解析完成后执行），
  两端行为一致。
- **不能用 `html` 行**：`insertAdjacentHTML` 插入的 `<script>` 按规范不执行。
- **网页端不会重复注入**：宿主先渲染行、再跑 `tapIndex`，原 tapIndex 里的幂等判断
  `html.indexOf('/dsh-whale/widget.js') !== -1` 会命中内联文本而跳过。
- **桌面端渲染器支持 `script` 行**：`document.createElement('script')` + `textContent` + append，
  内联脚本会执行。
- 注册返回的 disposer 一并进 `disposers`，沿用 HMR 清理约定。

## 验证

`node D:\ProgramData\dshUI\tools\verify-widget-injection.mjs`（桩验证，无需启动应用）：

```
registered routes   : 8
registered taps     : 1
index-inject subs   : 1
--- 注入行 ---
[ { "kind": "script", "placement": "body", "text": "(function(){...})()" } ]
kind 被桌面端支持   : true
内联脚本语法        : ok
同表二次触发后行数  : 1 (应为 1)
网页端 widget.js 出现次数 : 1 (应为 1 = 未重复注入)
新表行数            : 1 (应为 1)
```

客户端请求全部为相对路径（`/dsh-whale/size.json`、`/dsh-whale/widget.js` 等），桌面端经
`dsh-app://app/<path>` 转发到宿主 web server（携鉴权 cookie），因此路由与素材加载无需改动。

## 生效方式

宿主侧插件代码改动需要**重启桌面应用**：托盘右键 → 退出应用 → 重新打开。
（`/dsh-whale/widget.js` 路由带 `Cache-Control: no-store`，客户端脚本改动刷新即可。）

## 更新挂件时如何重新打补丁

1. 解压新版本 zip 到 `D:\ProgramData\dshUI\dsh-whale-widget`（覆盖前先备份 `lib/index.js`）；
2. 在 `apply()` 里 `ctx.webServer.tapIndex(...)` 之后，按上面的代码块补上
   `ctx.on('webserver/index-inject', ...)`；
3. 跑一次 `node D:\ProgramData\dshUI\tools\verify-widget-injection.mjs` 确认行类型与幂等；
4. 重启桌面应用。

> 若上游已改用结构化注入行，则不需要本补丁。
