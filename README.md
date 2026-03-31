# ccx

**Claude Code Exchange** — 一个 OAuth 反向代理，让多台设备共享一个 Claude Code 订阅。

[English](./docs/en/README.md)

```
Device A (claude code) ──┐
Device B (claude code) ──┼──→ ccx ──→ api.anthropic.com
Device C (headless)    ──┘   (token 管理
                              + 自动刷新)
```

## 功能

- 集中管理 OAuth token
- 过期前自动刷新（或在 401 时立即刷新）
- 反向代理 `api.anthropic.com`，自动注入 `Authorization: Bearer` 头
- 客户端只需指向 ccx，无需逐台设备登录

## 客户端配置

在每台设备上：

```bash
export ANTHROPIC_BASE_URL=http://<ccx-host>:8300
export CLAUDE_CODE_OAUTH_TOKEN=dummy  # 跳过本地登录
```

## 安全提示

- 仅在可信网络（LAN / Tailscale）上运行 ccx
- 切勿暴露到公网
- Refresh token 是长期有效的凭证，视同密码处理

## 许可证

MIT
