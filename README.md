# OpenClaw WSL Proxy Fix Notes

> codex整理发布
>
> 公开整理时间：2026-04-10
> 
> 本仓库是一次本地实战修复记录的脱敏整理版，聚焦 OpenClaw 在 WSL + HTTP 代理环境下的网络链路问题排查、修复、验证与清理。

## 结论先看

这次链路里实际叠了 4 层问题，不是单一故障：

1. Clash `fake-ip` 让 WSL DNS 返回 `198.18.x.x`，先触发了 SSRF 拦截。
2. Node/undici 经过代理时，CA/证书链行为与 `curl` 不同，导致早期出现 `fetch failed` / `UNABLE_TO_GET_ISSUER_CERT_LOCALLY` 类症状。
3. OpenClaw `web_fetch` 的真实工具路径里，`useEnvProxy` 在参数传递中被丢掉，导致 SSRF guard 重新退回 strict direct pinned-dispatcher 路径，最终对目标公网 IP 直连超时，而不是走代理。
4. 本机 WSL CLI 设备只剩 `operator.read`，所以 `openclaw agent` 的 gateway 路径会报 `pairing required`，随后回退到 embedded。

最终状态：

- `web_fetch` 正常
- `web_search` 正常
- `openclaw agent` 直走 gateway 正常
- `openclaw channels status --probe` 正常
- 已移除临时绕过 `NODE_TLS_REJECT_UNAUTHORIZED=0`

## 环境

- Ubuntu 24.04 on WSL2
- OpenClaw `2026.4.9`
- Node `v24.14.1`
- 代理：`HTTP_PROXY` / `HTTPS_PROXY` / `ALL_PROXY` -> `http://127.0.0.1:7897`
- 保留环境：`NODE_USE_ENV_PROXY=1`
- 清理后保留环境：`NODE_OPTIONS=--use-openssl-ca`
- 清理后移除：`NODE_TLS_REJECT_UNAUTHORIZED=0`

## 关键定位

### 1. 先把“代理坏了”与“工具路径坏了”分开

同一台机器、同一组环境变量下，直接测试结果是：

- Node global `fetch('https://example.com')` 成功
- `undici.fetch('https://example.com')` 失败，底层是 `UND_ERR_CONNECT_TIMEOUT`
- `undici.fetch(..., { dispatcher: new EnvHttpProxyAgent() })` 成功

这说明问题不是“代理必然坏了”，而是“某些调用路径没有真正用上代理 dispatcher”。

### 2. `web_fetch` 的真实问题不是 TLS，而是参数漏传

实际调用链是：

- `createWebFetchTool(...)`
- `runWebFetch(...)`
- `fetchWithWebToolsNetworkGuard(...)`
- `fetchWithSsrFGuard(...)`

表面上 `fetchWithSsrFGuard(...)` 已经支持 env proxy 模式，但真实 `web_fetch` 路径里 `useEnvProxy` 在两层参数传递中被漏掉，最终落回 strict 模式。

strict 模式下会创建 pinned dispatcher，直接去连目标公网 IP，因此在代理必需环境里就会出现：

- `TypeError: fetch failed`
- `cause: ConnectTimeoutError`
- `attempted addresses: <public-ip>:443`

也就是“看起来像网络不通”，但本质上是“根本没走代理”。

### 3. `pairing required` 是另一条独立链路

本地 WSL CLI 设备原本只有：

- `operator.read`

而 `openclaw agent` 真实需要更高 scopes，所以 gateway 会要求 repair pairing。批准本机设备的 pending request 之后，CLI device 被提升到完整 operator scopes，gateway 路径就恢复了。

## 本次实际修复

### A. 修 `web_fetch` 的 env proxy 传递

在安装中的 OpenClaw bundle 里做了定点补丁：

- `createWebFetchTool(...)` 调 `runWebFetch(...)` 时补上 `useEnvProxy: true`
- `runWebFetch(...)` 调 `fetchWithWebToolsNetworkGuard(...)` 时继续透传 `useEnvProxy`

说明：

- 这是针对本机已安装 bundle 的热修，不是上游源码 PR
- 目标是先恢复可用链路并验证根因

### B. 修 CLI device 的 pairing / scope 问题

执行本机设备 repair 批准后，CLI device 从只读升级到完整 operator scopes，`openclaw agent` 不再回退 embedded。

### C. 清理临时 TLS 绕过

删除了 user systemd drop-in：

- `~/.config/systemd/user/openclaw-gateway.service.d/tls.conf`

然后做完整回归，确认系统只保留：

- `NODE_USE_ENV_PROXY=1`
- `NODE_OPTIONS=--use-openssl-ca`

## 清理后的回归验证

以下验证均在 `2026-04-10` 完成：

### 1. `web_fetch`

目标：`https://example.com`

结果：

- HTTP `200`
- `finalUrl = https://example.com`
- `extractor = readability`

### 2. `web_search`

provider：`tavily`

结果：

- 返回正常结果
- `count = 2`
- `tookMs ≈ 1.2s`

### 3. `channels probe`

结果：

- Gateway reachable
- Telegram default: works
- openclaw-weixin: enabled, configured, running

### 4. `openclaw agent`

验证消息要求 agent 真正调用 `web_fetch`。

结果：

- gateway 路径成功完成
- 返回：`CLEAN_GATEWAY_OK https://example.com`
- 不再出现 `Gateway agent failed; falling back to embedded`

## 最终保留状态

推荐保留：

- `HTTP_PROXY` / `HTTPS_PROXY` / `ALL_PROXY`
- `NODE_USE_ENV_PROXY=1`
- `NODE_OPTIONS=--use-openssl-ca`

不再保留：

- `NODE_TLS_REJECT_UNAUTHORIZED=0`

## 经验总结

### 1. 不要把所有 `fetch failed` 都归结成证书问题

这次至少出现过三种不同层级：

- Fake-IP -> SSRF block
- CA / TLS trust mismatch
- `useEnvProxy` 参数丢失 -> strict direct dispatcher -> 直连超时

### 2. Node 能通，不代表 OpenClaw 的真实工具路径能通

必须分开验证：

- global `fetch`
- `undici.fetch`
- `undici.fetch + EnvHttpProxyAgent`
- 真实 tool 实例化执行
- 真实 `openclaw agent` gateway 执行

### 3. `pairing required` 可能不是“网关挂了”

本地 gateway 健康、loopback 正常时，也可能只是当前 CLI device scopes 不够，需要批准 repair request。

### 4. 最后一定要做“清理后回归”

如果不在移除 `NODE_TLS_REJECT_UNAUTHORIZED=0` 之后再次完整跑：

- `web_fetch`
- `web_search`
- `channels status --probe`
- `openclaw agent`

那就不能证明系统真的恢复到了干净状态。

## 上游相关公开案例

这次并不是完全孤例。GitHub 上已经有若干强相关问题：

- [openclaw/openclaw#63565](https://github.com/openclaw/openclaw/issues/63565)
  - 这是和本次 `web_fetch` 根因最接近的一条：`web_fetch` 在代理环境里绕过 env proxy，落回 SSRF guard DNS pinning / strict path。
- [openclaw/openclaw#47598](https://github.com/openclaw/openclaw/issues/47598)
  - 讨论 `web_fetch` 在企业 HTTP 代理环境下失效，需要 opt-in env-proxy mode。
- [openclaw/openclaw#53007](https://github.com/openclaw/openclaw/issues/53007)
  - 代理环境下 `memory_search` 不尊重 `HTTP_PROXY`。
- [openclaw/openclaw#44980](https://github.com/openclaw/openclaw/issues/44980)
  - Telegram 媒体下载路径被 strict guarded fetch 吃掉代理。
- [openclaw/openclaw#42176](https://github.com/openclaw/openclaw/issues/42176)
  - OAuth token exchange 不稳定走代理。
- [openclaw/openclaw#57688](https://github.com/openclaw/openclaw/issues/57688)
  - `pairing required` 导致本地 CLI 控制路径失效。
- [openclaw/openclaw#59889](https://github.com/openclaw/openclaw/issues/59889)
  - gateway restart 后 repair request / devices approve 的死锁类问题。

## 适用范围

这份记录更适合这些环境：

- WSL2 + Windows 代理（Clash / Clash Verge / 类似 HTTP CONNECT 代理）
- 中国网络 / 企业代理 / 校园网代理环境
- direct DNS 不可靠，或者 direct egress 不可用，但代理可用的场景
- OpenClaw 2026.x 系列中依赖 Node 24 / undici 路径的安装

## 仓库说明

- 本仓库只公开脱敏后的思路与修复路径
- 不包含任何 token、密码、私有 endpoint key、账号标识或原始配置文件
- 如果你在自己的环境复现，请先确认代理、DNS、SSRF guard、pairing/device scopes 是不是同一层问题

## 致谢

这份整理基于一次真实修复过程完成，但已做脱敏和结构化重写。
