# Upstream References

> codex整理发布
>
> 检索时间：2026-04-10

下面是和这次 OpenClaw / WSL / 代理链路问题最接近的 GitHub 公开案例索引。

## 最接近

### 1. `web_fetch` 绕过 env proxy
- Issue: [openclaw/openclaw#63565](https://github.com/openclaw/openclaw/issues/63565)
- 标题：`web_fetch tool bypasses env proxy in proxy-only environments (SSRF guard DNS pinning)`
- 关联度：最高
- 关联点：
  - `web_fetch`
  - `useEnvProxy`
  - SSRF guard
  - strict direct pinned dispatcher
  - 代理必需环境

### 2. 企业代理下 `web_fetch` 失败
- Issue: [openclaw/openclaw#47598](https://github.com/openclaw/openclaw/issues/47598)
- 标题：`web_fetch does not work behind corporate HTTP proxy (DNS NXDOMAIN); needs opt-in env-proxy mode + optional allowlist for internal URLs`
- 关联点：
  - 企业代理
  - direct DNS 不可用
  - `web_fetch` 需要 trusted env proxy 模式

## 同类代理问题

### 3. `memory_search` 不尊重代理
- Issue: [openclaw/openclaw#53007](https://github.com/openclaw/openclaw/issues/53007)
- 标题：`Memory Search fails with fetch failed when behind proxy (HTTP_PROXY not respected)`

### 4. Telegram 媒体下载路径掉代理
- Issue: [openclaw/openclaw#44980](https://github.com/openclaw/openclaw/issues/44980)
- 标题：`Telegram media download fails when proxy is required - fetchRemoteMedia bypasses configured proxy`

### 5. OAuth token exchange 代理路径不一致
- Issue: [openclaw/openclaw#42176](https://github.com/openclaw/openclaw/issues/42176)
- 标题：`openai-codex OAuth login does not honor proxy env for oauth/token exchange and refresh`

### 6. 全局代理支持诉求
- Issue: [openclaw/openclaw#43821](https://github.com/openclaw/openclaw/issues/43821)
- 标题：`Feature: Add global HTTP proxy support via environment variables`

## pairing / scopes 相关

### 7. CLI 控制路径 `pairing required`
- Issue: [openclaw/openclaw#57688](https://github.com/openclaw/openclaw/issues/57688)
- 标题：`cron add fails with pairing required`
- 关联点：
  - 本地 CLI 需要更高 scopes
  - 读操作可用，写/控制操作失败

### 8. 设备 repair / approve 死锁
- Issue: [openclaw/openclaw#59889](https://github.com/openclaw/openclaw/issues/59889)
- 标题：`devices approve deadlock after gateway restart — node-host repair request cannot be approved via CLI`

### 9. 本地 operator/device scopes 错配
- Issue: [openclaw/openclaw#51516](https://github.com/openclaw/openclaw/issues/51516)
- 标题：`openclaw devices/* and gateway RPC fail locally with pairing/operator scope mismatch`

### 10. 历史 token/scopes 旋转后触发 pairing
- Issue: [openclaw/openclaw#22067](https://github.com/openclaw/openclaw/issues/22067)
- 标题：`Token rotation causes pairing required errors - scopes not preserved after rotation`

## 检索结论

这次问题并不是完全没有先例。

最接近的现成公开案例是：

- [openclaw/openclaw#63565](https://github.com/openclaw/openclaw/issues/63565)

而本次本地修复的价值在于：

- 不是只停在“理论推断”
- 而是完成了完整闭环：定位 -> 热修 -> gateway pairing 修复 -> 删除 TLS 临时绕过 -> 清理后回归

因此本仓库更适合作为“实战 runbook / 修复记录”，而不是重复提交一条完全同义的 upstream bug。
