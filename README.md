# dsh-web-preview-panel

> **Fork note** — vendored from npm `dsh-web-preview-panel@0.2.4` (author
> zoumutou, MIT; upstream publishes no repository). Local patches on top of
> the pristine import (see git history):
>
> 1. **Streamed-message linkify fix** — after DOM mutations settle for 1.5s,
>    one idempotent rescan restores plain-text file links that React's
>    streaming commits destroyed mid-turn.
> 2. **Configurable link capture policy** — external http(s) links release to
>    the system browser by default; local paths stay in the side panel. See
>    「链接打开策略」 below for `externalLinks` / `panelHosts` / `browserHosts`.

DeepSeek Harness（DSH）侧边网页预览面板 —— 一个标准 Cordis 插件包（Host + Client 双半），
把工作区目录托管成 iframe 预览，并支持项目运行、元素标记批注与链接点击接管。

## 功能

- **侧边预览**：对话右上角 ▶ 按钮打开右侧玻璃卡片式预览面板（每个会话独立状态），
  地址栏实时同步、可拖动调宽（挤压对话 → 400px 封顶 → 覆盖模式）
- **本地文件预览**：工作区静态文件经 Harness 自身的 HTTP 服务托管
  （`/__dsh-preview/<sessionId>/...`）—— Markdown 渲染、代码带行号、图片直显、
  HTML 原样
- **项目运行**：自动检测项目类型（Cargo / package.json / go.mod / Python），
  ▶ 运行 / ⏹ 停止，日志实时滚动、自动识别 `localhost:PORT` 并加载预览；
  「日志→对话」一键把日志发给 AI 辅助诊断
- **元素标记**：🖉 标记模式（悬停手型高亮、点击选取元素、批注气泡、一键发送到对话），
  选择器/HTML 快照自动剥离内部标记类
- **对话框直接拖入文件**：把文件拖进对话框即保存到工作区 `.dsh-drops/` 并以
  路径引用加入草稿（对话框保持干净，AI 按需读取文件内容，上限 64 MB）；
  图片单独拖入仍走 DSH 原生图片附件轨，混合拖入时图片同样保存为文件引用
- **链接点击接管**：对话里点击 http(s) 链接、相对/绝对文件路径、纯文本路径
  （自动转链接）；外网链接默认放行到系统浏览器，本地路径在侧边打开
  （见下方「链接打开策略」）；Cmd/Ctrl+点击保留浏览器默认行为
- **会话隔离**：每个会话独立的预览状态、根目录（默认当前会话所属工作区）、运行进程

## 链接打开策略

`externalLinks` 决定外网 http(s) 链接（非 localhost / 非本站 origin）的默认去向；
`panelHosts` / `browserHosts` 按域名强制覆盖默认，匹配规则为精确域名或 dot 子域
后缀（`github.com` 匹配 `api.github.com`）。本地文件路径与 localhost 始终在侧边
预览打开，不受配置影响。缺省值声明在安装清单 `cordis.patch.yml`；profile 层用
id 定向整行覆盖（config 替换语义，需重述全部字段）：

```yaml
- id: dsh-web-preview-panel
  config:
    externalLinks: browser   # browser：放行到系统浏览器（默认）| panel：侧边预览打开
    panelHosts: []           # 例：[github.com] 强制 GitHub 在侧边预览
    browserHosts: []         # 例：[example.com] 强制放行到浏览器（externalLinks: panel 时有用）
```

配置经宿主半 `get-config` RPC 下发到浏览器半；修改后需重启 `dsh web` 并刷新页面。

## 安装

要求 DSH（DeepSeek Harness）Web 版。在 profile 目录（如 `~/.dsh/profiles/web`）：

```bash
npm install dsh-web-preview-panel
```

然后在 `cordis.patch.yml` 中挂载：

```yaml
- insert:
    - name: dsh-web-preview-panel
```

重启 `dsh web` 即生效。

## 开发

```bash
npm run build   # 产出 lib/index.js（host）与 lib/client.js（ModuleLoader bundle）
```

- `src/host.js` —— Host 半：`webServer` 路由（静态托管 + `POST /api` JSON RPC）、
  `subprocess` 项目运行、`fs` 文件访问
- `src/client.js` —— Client 半：React 面板 UI、标记/批注、点击接管、会话隔离；
  通过 `fetch('/__dsh-preview/api')` 与 Host 通信

## 协议

- `GET/HEAD /__dsh-preview/<sessionId>/<path>` —— 静态文件 / Markdown / 代码 / 图片
- `POST /__dsh-preview/upload` —— 拖入文件落盘（原始字节，头字段 `X-Dsh-Sid` /
  `X-File-Name` / `X-File-Size` / `X-File-Type`，保存到 `<root>/.dsh-drops/`，上限 64 MB）
- `POST /__dsh-preview/api` —— `{ method, args }` JSON RPC：
  `set-root` `pages` `resolve-path` `detect-project` `run-project` `project-status`
  `stop-project` `state`

## License

MIT
