# Case Comparison

> codex整理发布
>
> 本文把 upstream 公开案例与本地实修结果逐条对照，帮助区分“完全同根”“同类不同支路”“仅提供背景线索”三种关系。

## 总览

| 类型 | Upstream issue | 与本地关系 | 结论 |
|---|---|---|---|
| `web_fetch` / env proxy | [#63565](https://github.com/openclaw/openclaw/issues/63565) | 几乎完全同根 | 最接近，和本地热修命中同一参数漏传链路 |
| `web_fetch` / corporate proxy | [#47598](https://github.com/openclaw/openclaw/issues/47598) | 高度相似 | 本地实修验证了它的总体方向，但比它更具体 |
| memory / proxy | [#53007](https://github.com/openclaw/openclaw/issues/53007) | 同类问题族 | 提示“全局代理不等于所有子路径都走代理” |
| CLI `pairing required` | [#57688](https://github.com/openclaw/openclaw/issues/57688) | 高度相似 | 本地现象更轻，修复路径更直接 |
| devices approve deadlock | [#59889](https://github.com/openclaw/openclaw/issues/59889) | 同家族但更严重 | 本地没有落入 deadlock，`devices approve` 成功 |
| local operator/device mismatch | [#51516](https://github.com/openclaw/openclaw/issues/51516) | 高度相似 | 本地可视作它的较新变体 |

## 1. `web_fetch` / env proxy：#63565

链接：
- [openclaw/openclaw#63565](https://github.com/openclaw/openclaw/issues/63565)

### 它说了什么
这条 issue 直接指出：

- `web_fetch` 没有把 `useEnvProxy: true` 传给 SSRF guard
- `web_search` 能工作，是因为它走 `withTrustedWebToolsEndpoint`
- `web_fetch` 却走了不带 `useEnvProxy` 的路径
- 结果就是 SSRF guard 在代理之前先做 DNS pinning / strict direct dispatch

### 和本地的重合点
这和本地修复几乎一模一样：

- 本地实际验证到 `web_fetch` 真实路径里 `useEnvProxy` 被丢了
- 丢失后落回 strict direct pinned-dispatcher
- 底层错误不是证书，而是直连目标公网 IP 超时：`UND_ERR_CONNECT_TIMEOUT`
- 本地热修也是围绕这两层参数传递做的

### 本地比 issue 多出来的部分
本地不只是“看源码推断”，而是完成了闭环：

- 真实工具实例化执行前后对比
- `undici.fetch` / `EnvHttpProxyAgent` 对照实验
- 网关路径回归
- 删除 `NODE_TLS_REJECT_UNAUTHORIZED=0` 之后再次回归

### 结论
如果只保留一条最相关 upstream 参考，这条就是最接近本地根因的案例。

## 2. 企业代理场景：#47598

链接：
- [openclaw/openclaw#47598](https://github.com/openclaw/openclaw/issues/47598)

### 它说了什么
这条旧 issue 讨论的是：

- corporate proxy 环境下 direct DNS 失败
- `web_fetch` 使用 strict SSRF-safe fetch
- 即使系统代理可用，`web_fetch` 也还是先做 direct DNS pinning，最终 `ENOTFOUND`

### 和本地的重合点
本地也验证了同一大方向：

- global `fetch` 能通，不代表 `web_fetch` 真正能通
- 一旦 guarded fetch 路径创建了自己的 dispatcher，全局代理设置就可能被绕开
- `curl` 成功不代表 OpenClaw 工具路径成功

### 和本地的差异
这条 issue 更偏“设计层问题描述”，而本地修复把问题精确到了：

- 不是抽象的“strict SSRF-safe fetch 会绕过代理”
- 而是当前版本已有 `TRUSTED_ENV_PROXY` 模式，但 `web_fetch` 调用方没把 `useEnvProxy` 传进去

换句话说：

- #47598 更像早期问题画像
- 本地这次更像已经把具体漏点钉在调用链上

### 结论
它不是和本地一模一样，但本地修复很好地证实了它的总体判断。

## 3. `memory_search` 代理失败：#53007

链接：
- [openclaw/openclaw#53007](https://github.com/openclaw/openclaw/issues/53007)

### 它说了什么
这条 issue 认为：

- `memory_search` / embedding 远程请求没有正确尊重 `HTTP_PROXY`
- 建议在 gateway startup 就装 `EnvHttpProxyAgent`

### 和本地的关系
这条不是同一个子系统，但属于同一个问题族：

- “环境变量已经设置” 不等于 “所有网络子路径都真的走代理”
- 某些调用路径如果显式创建自己的 fetch / dispatcher，就会跳过全局行为

### 本地能支持它什么，不能支持它什么
本地能支持的结论：

- 这类问题确实存在，而且会出现在 OpenClaw 的不同网络子系统里
- 全局层面的 `NODE_USE_ENV_PROXY=1` 或 `setGlobalDispatcher(...)` 并不能自动修正所有 guarded/pinned fetch 路径

本地不能直接替它背书的部分：

- 我这次没有实测 `memory_search` 路径
- 所以不能直接说 #53007 和本地已经被同一补丁修复

### 结论
它更像“同家族旁证”，说明这不是单点巧合，而是 OpenClaw 网络层存在多处相似的代理路径分裂问题。

## 4. `pairing required`：#57688

链接：
- [openclaw/openclaw#57688](https://github.com/openclaw/openclaw/issues/57688)

### 它说了什么
这条 issue 的现象是：

- `cron add` 一类需要更高权限的 CLI / agent 操作报 `pairing required`
- 读操作能用，写/控制类操作失败
- 用户误以为没有 CLI pairing 机制

### 和本地的重合点
本地几乎同类：

- `openclaw gateway status`、`devices list` 这类读路径能走
- `openclaw agent` 这种需要 scope upgrade 的路径报 `pairing required`
- 根因也是当前 CLI device 只有 `operator.read`

### 和本地的差异
本地比它更顺利的地方在于：

- `openclaw devices list --json` 能看到 pending repair request
- `openclaw devices approve --latest --json` 直接成功
- 批准后本机 CLI device 获得完整 operator scopes
- `openclaw agent` 恢复正常

也就是说：

- #57688 更像“用户没有找到可用批准路径，或者版本存在更深的问题”
- 本地属于“repair request 存在，但走 local fallback 仍能批准”的较轻场景

### 结论
高度相似，但本地没有落入“无可恢复路径”的状态。

## 5. devices approve deadlock：#59889

链接：
- [openclaw/openclaw#59889](https://github.com/openclaw/openclaw/issues/59889)

### 它说了什么
这条 issue 更严重：

- `devices list` 能看到 pending request
- 但 `devices approve` 返回 unknown requestId
- 没有 local fallback approve 能力
- 最终形成死锁

### 和本地的关系
本地属于同家族，但没有到这一步：

- 本地确实有 pending repair request
- 但 approve 命令是成功的
- 批准后 scopes 立即恢复正常

### 本地对它的启发
本地说明：

- 不是所有 `pairing required` 都会升级成 #59889 这种 deadlock
- 先做 `devices list --json` 看 pending repair request 很重要
- 如果 approve 路径仍可用，修复会非常直接

### 结论
本地是“同问题家族的轻度分支”，不是这条 issue 的完全复刻。

## 6. local operator/device mismatch：#51516

链接：
- [openclaw/openclaw#51516](https://github.com/openclaw/openclaw/issues/51516)

### 它说了什么
这条 issue 描述的是：

- loopback gateway 健康
- 但 CLI / gateway RPC 出现 `missing scope: operator.read`、`pairing required`
- `devices list/approve` 等本地 operator/device 路径异常

### 和本地的重合点
这和本地 pairing 问题高度相似：

- gateway 本身是健康的
- 问题在 device scopes / repair pairing，而不是服务没起来
- 本地也出现了 scope upgrade request

### 和本地的差异
本地在 device 管理这层没有完全失效：

- `devices list` 正常
- `devices approve --latest` 正常
- 不是“路径全坏”，而是“设备 scope 漂移后等待 repair 批准”

### 结论
如果把 #59889 看作更严重死锁分支，那 #51516 更像是本地场景的历史同类案例。

## 综合判断

### 哪条最像本地 `web_fetch` 问题？
- [#63565](https://github.com/openclaw/openclaw/issues/63565)

### 哪条最像本地 `pairing required` 问题？
- [#57688](https://github.com/openclaw/openclaw/issues/57688)
- [#51516](https://github.com/openclaw/openclaw/issues/51516)

### 哪条虽然相关，但本地没有完全落进去？
- [#59889](https://github.com/openclaw/openclaw/issues/59889)

### 哪条提供了“同类问题族”的背景？
- [#53007](https://github.com/openclaw/openclaw/issues/53007)
- [#47598](https://github.com/openclaw/openclaw/issues/47598)

## 最终一句话

本地这次不是“全新未知问题”，而是两个 upstream 已经有明显影子的已知问题族叠加：

- `web_fetch` 侧，和 [#63565](https://github.com/openclaw/openclaw/issues/63565) 基本同根
- `pairing` 侧，和 [#57688](https://github.com/openclaw/openclaw/issues/57688) / [#51516](https://github.com/openclaw/openclaw/issues/51516) 很接近，但本地幸运地没有掉进 [#59889](https://github.com/openclaw/openclaw/issues/59889) 那种 approval deadlock
