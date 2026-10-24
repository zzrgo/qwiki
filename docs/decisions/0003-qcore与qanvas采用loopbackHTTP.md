# 0003. qcore 与 qanvas 采用 loopback HTTP/JSON

- 状态：已接受
- 日期：2026-10-25
- 相关：[ADR 0002](0002-v1本机网关与AI只读.md)、[q-0004](../questions/q-0004-qcore与qanvas跨进程通信.md)

## 背景

- qanvas 为 Electron 常驻应用，qcore 为独立 CLI，供外部 Claude Code 调用。
- QA 使用 macOS / Linux / Windows（Windows 最后实现），需要跨平台。
- 暂不需要流式输出 / 长任务进度。
- Electron IPC 只用于应用内渲染进程与主进程，外部 CLI 无法使用。

## 决策

1. qcore 与 qanvas 主进程通过 **HTTP/JSON over 127.0.0.1** 通信；不使用 Electron IPC、stdio、gRPC、WebSocket。
2. qanvas 启动时监听 `127.0.0.1:0`（随机端口），生成每次启动有效的 bearer token，写入 `runtime.json`（POSIX 权限 0600；Windows 用 ACL 限制）。
3. qcore 通过 `QANVAS_RUNTIME` 或平台默认路径发现 `runtime.json`，握手 `/v1/health` 后带 token 请求。
4. 不采用 Unix socket / named pipe 作为 v1 主通道；它们要求平台分支和客户端适配，收益不足以抵消跨平台成本。未来若只面向 macOS / Linux，可作为 transport adapter 加入。
5. 不引入流式传输；请求 / 响应 + 超时即可。长查询通过提高超时或异步任务（未来）处理。
6. qcore 随 qanvas 安装包分发，保证版本匹配；Claude Code 通过 Bash 调用，qcore 必须非交互、输出 JSON、稳定退出码。
7. 所有 AI 请求经 qanvas 服务端只读策略与审计；qcore 的客户端护栏是第二层。

## 备选方案

- **Unix domain socket / named pipe**：更强的本机隔离，但跨平台成本高。
- **stdio**：仅适合父子进程；qanvas 常驻且 AI 在外部，排除。
- **gRPC**：类型契约好，但 v1 过重，且无流式需求。
- **WebSocket / SSE**：无流式需求，排除。

## 后果

- 本机任意同用户进程都可能读取 `runtime.json` 并尝试连接；token 只读、短时效，服务端策略仍是边界。
- 随机端口 + runtime 文件需要处理陈旧文件、重启后端口 / token 变化；qcore 在 401 / 连接失败时重读 runtime。
- Windows 需要验证 loopback 监听、ACL、路径与防火墙行为；最后实现，但路径与权限抽象现在就要留好。
- 无流式意味着超长查询只能等待或超时；v1 用行数 / 时间上限控制。
- 若未来需要写操作、跨机器、流式进度，需新 ADR。
