# qcore 与 qanvas 的跨进程通信方式

- 状态：已解答
- 日期：2026-10-25
- 相关：[ADR 0001](../decisions/0001-资源管理自研注册中心与执行路径.md)、[ADR 0002](../decisions/0002-v1本机网关与AI只读.md)

## 背景

qanvas 是常驻的 Electron 应用（按 Electron 理解）；qcore 是独立 CLI，供 AI 调用。AI 通过 qcore 访问 qanvas 的本机网关。需要确定：

- qcore 如何找到 qanvas？
- 两者用什么传输方式通信？
- 身份、错误、生命周期怎么处理？

Electron 的 `ipcMain` / `ipcRenderer` 只用于应用内渲染进程与主进程，外部 CLI 无法使用。因此 qcore 必须通过主进程暴露的本机端点通信。

## 候选

| 方式 | 跨平台 | 外部 CLI 可用 | 优点 | 缺点 |
| --- | --- | --- | --- | --- |
| HTTP/JSON over 127.0.0.1 | 是 | 是 | 简单、可 curl 调试、任意语言、随机端口 | 需 token；本机其他进程可尝试连接 |
| Unix domain socket | macOS / Linux | 是 | 文件权限、无端口冲突 | Windows 需 named pipe；客户端要适配 |
| Windows named pipe | Windows | 是 | ACL、无端口 | 平台特定 |
| Electron IPC | 是 | 否 | 应用内简单 | 只限 renderer ↔ main，排除 |
| stdio | 是 | 仅父子进程 | 简单、无需端口 | qanvas 常驻，不适合做主通道 |
| gRPC | 是 | 是 | 类型契约、流式 | 依赖与构建复杂，v1 过度 |
| WebSocket | 是 | 是 | 双向流式 | 请求/响应场景复杂化 |
| D-Bus / XPC / COM | 否 | 平台特定 | 系统集成 | 不跨平台，排除 |

## 倾向

v1 用 **HTTP/JSON + 127.0.0.1 + 随机端口 + 每次启动生成的 token**：

- qanvas 主进程在 `app.whenReady()` 后监听 `127.0.0.1:0`，把实际端口、token、pid、API 版本写入 `runtime.json`（权限 0600，位于 `app.getPath('userData')`）。
- qcore 优先读 `QANVAS_RUNTIME` 环境变量，否则读平台默认的 `runtime.json`；连接前检查 pid 与 `/v1/health`。
- 服务端从 token 解析会话身份，不信任客户端自报的 agent / task header。
- 换传输方式时，qcore 的命令语义不变；HTTP 只是 transport adapter。

如果 QA 全部在 macOS / Linux，且希望更强的本机隔离，可以改用 Unix domain socket；如果包含 Windows，loopback HTTP 更省事。

## 通用要素（跨进程调用）

1. **发现**：runtime 文件 / 环境变量 / 固定路径；处理陈旧文件与端口变化。
2. **握手**：`/v1/health` 返回 app 版本、API 版本、能力列表，qcore 做兼容性检查。
3. **身份**：token 绑定会话（agent / task / 发起人），短时效；服务端每次请求重新判权。
4. **序列化**：JSON；统一错误信封（code、message、audit_id）。
5. **生命周期**：Electron 单实例锁；窗口关闭后托盘常驻；重启后 token / 端口变化，qcore 重读 runtime 文件；服务不可用时 fail closed。
6. **可观测性**：请求 ID / audit ID、服务端结构化日志；qcore 不打印 token。
7. **超时与重试**：只读请求可有限重试；写操作（未来）不可盲目重试。

## Electron 侧注意

- 网关放在主进程，不要放渲染进程；渲染进程通过 Electron IPC 调主进程，qcore 通过本地端点调主进程，两条路径复用同一个 gateway 模块。
- 不要给渲染进程 Node 权限；开启 `contextIsolation`，preload 只暴露最小 API。
- 窗口关闭后要能常驻（托盘 / `window-all-closed` 处理），否则 qcore 会随窗口退出而失败。
- 本机端点只绑 `127.0.0.1`，不要绑 `0.0.0.0`；检查 `Host`，防止 DNS rebinding；不启用 CORS。

## 待确认

- [x] QA 的操作系统是否统一？→ macOS / Linux / Windows（Windows 最后实现），因此 v1 统一用 loopback HTTP。
- [x] AI 运行时在 qanvas 内，还是外部进程？→ 外部 Claude Code；qanvas 提供 qcore CLI。
- [x] 是否需要流式输出 / 长任务进度？→ 暂不需要，请求 / 响应 + 超时即可。
- [ ] qcore 用什么语言实现？（影响打包与 Windows 分发）
- [x] Claude Code 与 qanvas 是否同机？→ 已确认同机运行，按本机 loopback 模型设计；Docker / WSL / 远端等隔离环境的访问方式不在本次确认范围内。

## 结论（2026-10-25）

采用 loopback HTTP/JSON；不做 Unix socket / named pipe 分支，不引入流式。详见 [ADR 0003](../decisions/0003-qcore与qanvas采用loopbackHTTP.md)。

## Agent 调用说明（2026-10-25 补充）

已确认：qanvas 会提供 qcore skill 给 Agent harness（当前为 Claude Code），用于说明 qcore 的命令行调用方式。术语见 [GLOSSARY](../../GLOSSARY.md)。

- Claude Code 读取 skill 后，通过 Bash 调用 qcore；qcore 再通过本机 HTTP/JSON 请求 qanvas。
- skill 承载调用说明；只读策略与审计仍由 qanvas 服务端执行，沿用 [ADR 0002](../decisions/0002-v1本机网关与AI只读.md)。
- qcore 的具体命令集继续待定；本次仅确认 skill 的提供方、使用方与用途。

后续待定：skill 如何安装到 Claude Code、如何随 qcore 更新并保持说明与版本一致。
