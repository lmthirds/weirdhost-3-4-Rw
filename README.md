# Weirdhost Auto Renew

基于 **GitHub Actions + SeleniumBase CDP Mode + sing-box** 的 Weirdhost 多账号全自动续期与离线保活工具。

脚本全程运行在纯 CDP Mode 下（彻底脱离 ChromeDriver），原生支持 **Cookie 优先 + 邮箱密码自动重登兜底 + 最新 Cookie 自动回写 Secrets**、**单账号独立代理与出口时区对齐**、**离线 / 休眠（`수면중`）自动开机**，以及 **Telegram 图文状态推送**。

---

## 核心功能

- **双模式免维护登录**：
  - 优先注入 `remember_web_*` Cookie 秒登，避免频繁触发登录风控。
  - Cookie 失效或未填写时，自动切换为 `WEIRDHOST_ACCOUNT_N` 邮箱密码登录。
  - 每次登录成功后，自动将浏览器最新生成的 `remember_web_*` 回写到 GitHub Secrets（即使当天服务器处于续期冷却期也会同步更新）。
- **单账号独立网络与指纹隔离**：
  - 通过 `sing-box` 为每个账号单独启动本地 `mixed` 代理端口，运行完毕立即销毁。
  - 自动探测代理出口 IP 与地理时区，并同步设置浏览器 `TZ` 环境变量，同时禁用非代理 UDP 防止 WebRTC 泄露。
- **Cloudflare 全屏过盾与弹窗续期**：
  - 自动识别韩/英/中 Cloudflare 全屏拦截页，结合 CDP 内置过盾与 X11 `xdotool` 坐标点击自动放行。
  - 自动展开续期弹窗内被截断的 Turnstile 控件，精确匹配复选框尺寸并提交续期，随后通过 API 校验到期时间变化。
- **离线 / 休眠自动开机**：
  - 每次巡检自动通过 `/api/client/servers/{id}/resources` 与页面 DOM 综合判断电源状态。
  - 检测到 `offline` 或 `수면중`（休眠中）时，等待控制台 `Start` 按钮就绪自动点击，若页面未就绪则自动走 Client API 兜底发送开机指令并轮询确认状态。
- **通知与可视化排障**：
  - 单账号独立推送 Telegram 文本/截图报告（含出口 IP、登录方式、电源状态、续期结果与剩余时长）。
  - 每次运行自动归档所有截图至 GitHub Actions Artifacts（`debug-screenshots`，保留 3 天），并自动清理旧运行记录（保留最近 3 次）。

---

## 目录结构

```text
.
├── .github/
│   └── workflows/
│       └── Weirdhost_renew.yml   # GitHub Actions 定时/手动工作流配置
├── scripts/
│   └── weirdhost_renew.py        # 自动登录、开机、续期与通知主脚本
└── README.md
```

---

## 快速开始

1. **创建私有仓库**：将本项目 Fork 或上传到你的 GitHub 仓库（**强烈建议设为 Private 私有仓库**，避免日志或配置泄露）。
2. **配置 Secrets**：打开仓库 **Settings → Secrets and variables → Actions → New repository secret**，参照下方配置表添加账号、代理及通知变量。
3. **测试运行**：前往 **Actions → Weirdhost 多账号自动续期 → Run workflow** 手动触发一次，观察日志与 Telegram 通知。

---

## 配置变量详解

### 一、GitHub Secrets（在仓库 Settings 中配置）

工作流默认映射 **1 ~ 5 号账号**，且按序号一一对应（例如账号 1 对应 `WEIRDHOST_ACCOUNT_1`、`WEIRDHOST_COOKIE_1`、`PROXY_NODE_1`）。

#### 1. 账号登录配置（`ACCOUNT` 与 `COOKIE` 二选一或同时配置，推荐同时配置）

| Secret 名称 | 是否必填 | 格式 | 说明 |
| :--- | :---: | :--- | :--- |
| `WEIRDHOST_ACCOUNT_1` ~ `5` | **强烈推荐** | `邮箱-----密码`<br>或<br>`备注-----邮箱-----密码` | 邮箱密码登录凭据；当对应 `WEIRDHOST_COOKIE_N` 失效或为空时自动启用重登兜底。分隔符为 **5 个短横线 `-----`**。 |
| `WEIRDHOST_COOKIE_1` ~ `5` | 可选 | `备注-----remember_web_xxx=yyy`<br>或<br>`remember_web_xxx=yyy` | 优先使用的登录 Cookie。若已配置 `WEIRDHOST_ACCOUNT_N` 和 `REPO_TOKEN`，此项可留空，首次账密登录成功后会自动创建/回写。 |

> **示例**：
> - `WEIRDHOST_ACCOUNT_1`：`主号-----user1@gmail.com-----MyPass123!` 或 `user1@gmail.com-----MyPass123!`
> - `WEIRDHOST_COOKIE_1`：`主号-----remember_web_59ba36addc2b2f9401580f014c7f58ea4e30989d=eyJ...In0%3D`

#### 2. 代理节点配置（可选，均不填则直连）

| Secret 名称 | 优先级 | 格式 | 说明 |
| :--- | :---: | :--- | :--- |
| `PROXY_NODE_1` ~ `5` | **高（专属绑定）** | 单条节点链接 | 与账号序号 1~5 一对一绑定，确保每个账号固定使用各自独立出口 IP。 |
| `PROXY_NODES` | 低（公共节点池） | 每行一条节点链接（`#` 开头为注释） | 未配置对应 `PROXY_NODE_N` 时，按账号序号从节点池中轮询分配。 |

**`sing-box` 支持的代理协议清单：**

| 协议 | 链接前缀 | 支持的传输与安全参数 |
| :--- | :--- | :--- |
| **VLESS** | `vless://` | `security=reality`（含 `pbk`/`sid`/`fp`/`sni`/`flow`）、`security=tls`、`tcp`/`ws`/`grpc`/`httpupgrade`/`h2` |
| **Hysteria 2** | `hy2://` / `hysteria2://` | `sni`/`peer`、`insecure`/`allowInsecure`、`alpn`、`obfs`/`obfs-password` |
| **VMess** | `vmess://` | Base64 JSON 格式，支持 `tls`、`ws`/`grpc`/`h2`/`httpupgrade`、`sni`/`fp`/`alpn` |
| **Trojan** | `trojan://` | `tls`、`sni`/`fp`/`alpn`/`insecure`、`tcp`/`ws`/`grpc`/`httpupgrade`/`h2` |
| **Shadowsocks** | `ss://` | 支持明文与 Base64 编码的 `method:password@host:port` |
| **SOCKS5 / HTTP** | `socks5://`、`socks://`、`http://`、`https://` | 支持可选的 `username:password@` 认证 |

#### 3. 自动回写与 Telegram 通知配置（可选）

| Secret 名称 | 是否必填 | 格式 / 示例 | 说明 |
| :--- | :---: | :--- | :--- |
| `REPO_TOKEN` | 推荐 | `github_pat_xxx` 或 `ghp_xxx` | 用于在登录成功或 Cookie 轮换时，通过 GitHub API 自动加密回写 `WEIRDHOST_COOKIE_N`。不填则跳过回写，不影响续期。 |
| `TG_BOT_TOKEN` | 可选 | `123456789:ABCdef...` | Telegram Bot Token（通过 `@BotFather` 创建获取）。 |
| `TG_CHAT_ID` | 可选 | `123456789` 或 `-100xxxxxxxxxx` | 接收续期通知的个人 ID 或群组 ID（不填则跳过通知）。 |

---

### 二、工作流环境变量（在 `.github/workflows/Weirdhost_renew.yml` 中修改）

| 环境变量名 | 默认值 | 可选值 | 说明 |
| :--- | :---: | :---: | :--- |
| `PROXY_FALLBACK_DIRECT` | `"true"` | `"true"` / `"false"` | 当 `sing-box` 代理启动失败或节点不可用时，是否自动回退到 GitHub Actions 直连。如需强制多账号严格 IP 隔离，改为 `"false"`。 |
| `AUTO_START_OFFLINE` | `"true"` | `"true"` / `"false"` | 检测到服务器为 `offline` 或 `수면중`（休眠中）时，是否自动点击 `Start` 并尝试 API 兜底开机。不需要自动开机可改为 `"false"`。 |
| `GITHUB_REPOSITORY` | `${{ github.repository }}` | 自动注入 | 当前仓库名（`owner/repo`），供 `REPO_TOKEN` 回写 Secrets 使用，无需手动修改。 |

---

## 进阶配置指引

### 1. Google / Kakao 一键注册的账号如何设置密码？
Weirdhost 面板原生支持邮箱 + 密码登录：
1. 访问 `[https://hub.weirdhost.xyz/auth/login](https://hub.weirdhost.xyz/auth/login)`，点击最底部的 **`비밀번호를 까먹었나요?`**（忘记密码）。
2. 输入该账号绑定的邮箱，查收重置邮件并设置固定密码（或登录后在左侧 **`계정`（账户）** 页面设置密码）。
3. 使用浏览器无痕窗口验证可用 **邮箱 + 密码** 正常登录后，将 `邮箱-----密码` 填入 `WEIRDHOST_ACCOUNT_N`。

### 2. 如何配置 `REPO_TOKEN`（开启 Cookie 自动回写）？
前往 GitHub 右上角头像 → **Settings → Developer settings → Personal access tokens**：
- **方式 A（Fine-grained tokens，推荐）**：
  - `Repository access` 选择 **Only select repositories** → 选中本仓库。
  - `Repository permissions` 中将 **Secrets** 设为 **Read and write**。
- **方式 B（Tokens classic）**：勾选 **`repo`** 权限。
- 复制生成的 Token，填入仓库 Secrets 的 `REPO_TOKEN` 中。

### 3. 如何手动获取 `remember_web_*` Cookie（可选）？
若某账号不想配置密码，只想单独使用 Cookie：
1. 新开一个**独立无痕窗口**登录 `[https://hub.weirdhost.xyz/auth/login](https://hub.weirdhost.xyz/auth/login)`。
2. 进入面板后按 `F12` → **Application → Cookies → `[https://hub.weirdhost.xyz](https://hub.weirdhost.xyz)`**（或在 **Network** 任意请求的 `Cookie` 请求头中），复制完整的 `remember_web_xxx=yyy`（不含末尾分号和其他字段）。
3. **关键**：复制完毕后**直接关闭无痕窗口，切勿点击面板左下角的退出登录按钮**，否则服务端会立刻作废刚复制的 `remember_token`。

### 4. 如何扩展到 6 ~ 10 个账号？
脚本内置上限为 `MAX_COOKIE_COUNT = 10`（10 个账号）。如需添加第 6 个及以上账号：
1. 在仓库 Secrets 中添加 `WEIRDHOST_ACCOUNT_6`、`WEIRDHOST_COOKIE_6`、`PROXY_NODE_6`。
2. 在 `.github/workflows/Weirdhost_renew.yml` 的 `env:` 块中补上对应映射行即可。

### 5. 如何修改定时执行时间？
编辑 `.github/workflows/Weirdhost_renew.yml` 顶部的 `cron`（使用 **UTC 时间**）：
```yaml
on:
  schedule:
    - cron: '20 4 * * *'   # 默认每天 UTC 04:20（北京时间 12:20）
```
- 例如每天北京时间 08:30 与 20:30 各巡检一次，可改为 `'30 0,12 * * *'`。

---

## 调试与常见问题

### 如何查看运行截图排障？
每次工作流运行结束后（无论成功或失败），页面底部都会生成 **`debug-screenshots`** 压缩包（保留 3 天），常见截图文件含义如下：
- `accN_cf_fail.png`：第 N 个账号卡在 Cloudflare 全屏验证页。
- `accN_login_fail.png`：第 N 个账号 Cookie 失效或邮箱密码登录被拒 / 触发登录页 reCAPTCHA。
- `accN_srvM_popup.png` / `*_turnstile_*.png`：续期弹窗与 Turnstile 验证点击过程截图。
- `accN_srvM_btn_disabled.png` / `*_no_btn.png`：续期按钮处于冷却禁用状态或未找到按钮。

### 常见报错排查

| 现象 / 报错 | 原因 | 解决方法 |
| :--- | :--- | :--- |
| `代理配置不含 '://'` | `PROXY_NODE_N` 误填了 Cookie 或不完整链接 | 检查对应 Secret，确保以 `vless://`、`hy2://` 等协议头开头 |
| `节点连通性测试失败` | 代理节点已失效或端口不通 | 更换可用节点；或确认 `PROXY_FALLBACK_DIRECT: "true"` 允许回退直连 |
| `该账号开启了二步验证 (2FA)` | 账号开启了 2FA，无法自动账密登录 | 关闭该账号 2FA，或改用 Cookie 登录 |
| `账密登录超时未跳转` | 密码错误或代理出口 IP 质量较差触发 reCAPTCHA 图片验证 | 核对密码并下载 `accN_login_fail.png` 查看提示，必要时更换干净节点 |
| `WEIRDHOST_COOKIE_N 回写失败` | `REPO_TOKEN` 权限不足或已过期 | 重新生成具备本仓库 `Secrets: Read and write` 权限的 Token 并更新 `REPO_TOKEN` |

---

## 免责声明

本项目仅供个人服务器维护与自动化技术学习研究使用，请合理设置定时频率，切勿滥用平台资源并遵守服务商相关使用条款。
