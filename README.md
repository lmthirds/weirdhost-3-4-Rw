# Weirdhost Auto Renew

基于 GitHub Actions + SeleniumBase CDP Mode 的 Weirdhost 多账号自动续期与离线保活脚本。支持 **Cookie 优先 + 邮箱密码自动重登兜底**、**单账号独立 `sing-box` 代理隔离**、**离线 / 休眠（`수면중`）自动开机** 与 **Telegram 图文通知**。

## 核心特性

- **双模式免维护登录**：优先使用 `remember_web_*` Cookie 秒登；Cookie 失效时自动切换邮箱密码重新登录，登录成功后第一时间将新 Cookie 回写至 GitHub Secrets（需配置 `REPO_TOKEN`）。
- **多账号网络隔离**：通过 `sing-box` 为每个账号启动独立本地代理进程并自动对齐出口 IP 时区，支持 `vless://`、`hy2://` / `hysteria2://`、`vmess://`、`trojan://`、`ss://`、`socks5://`、`http://`。
- **Cloudflare 过盾与弹窗续期**：全程运行在纯 CDP Mode 下（脱离 ChromeDriver），自动处理全屏 Cloudflare 验证及续期弹窗内的 Turnstile 验证。
- **离线 / 休眠自动开机**：每次续期前自动检测服务器电源状态，遇到 `offline` 或 `수면중`（休眠中）自动点击控制台 `Start` 按钮并附带 API 兜底开机。
- **通知与排障**：单账号独立推送 Telegram 文本/截图通知，工作流结束后自动归档 `debug-screenshots` 调试截图（保留 3 天）。

## 目录结构

```text
.
├── .github/
│   └── workflows/
│       └── Weirdhost_renew.yml   # GitHub Actions 定时与手动工作流
├── scripts/
│   └── weirdhost_renew.py        # 自动续期与开机主脚本
└── README.md
