# 自建 auth 测试指南：编译 / 打包 / 启动

> 本文件只存在于本 fork 的 `feat/self-hosted-auth` 分支，上游没有再这份文件。
> 自建 auth 服务本身**不在本仓**（仓库边界：服务端在独立仓库），需要单独把 `cindy-auth/` 目录带过去。

## 先选路径

| 你想验证什么 | 用哪条 | 能否连自建 auth |
|---|---|---|
| 登录流程、界面、自建 auth 全链路 | **dev 模式** | ✅ 可以（localhost 允许 http） |
| 应用能否正常启动、打包产物是否完整 | **打包（installer / 未压缩目录）** | ⚠️ **必须有 https 地址** |
| 发给别人安装 | 打包 + 签名 | 同上 |

关键约束：**打包版只接受 https 端点**（`allowHttp` 只在 dev 的本地清单文件模式放开）。所以没有域名时，**只有 dev 模式能真正验证登录**；打包版可以验证"能启动、能开窗"，但登录会连不上。

---

## 一、环境准备

| 依赖 | 要求 | 检查 |
|---|---|---|
| Node.js | 22.x（仓库 `engines.node >= 22.13`） | `node -v` |
| pnpm | **11.28.9**（`packageManager` 指定，`engineStrict` 会拦旧版） | `corepack prepare pnpm@11.28.9 --activate` 然后 `corepack pnpm -v` |
| Git LFS | 必需（仓库有 LFS 资源） | `git lfs version` |

Windows 上 PowerShell 的执行策略会拦 corepack 生成的 `pnpm.ps1`，**统一用 `pnpm.cmd` 调用**即可绕过：

```powershell
pnpm.cmd -v
```

如果 `pnpm -v` 报"无法加载文件 pnpm.ps1，未对文件进行数字签名"，就是这个问题。

---

## 二、安装依赖

```powershell
git clone -b feat/self-hosted-auth https://github.com/loszhang/cindy.git
cd cindy
git lfs pull
pnpm.cmd install --frozen-lockfile
```

Electron 下载失败时加镜像重跑：

```powershell
$env:ELECTRON_MIRROR = "https://npmmirror.com/mirrors/electron/"
pnpm.cmd install --frozen-lockfile
```

预期：`node_modules` 约 5–6 GB。末尾出现 `3/4 agent 二进制未就位（best-effort，不阻断）` 属正常——那是 claude/codex/ripgrep/pi 的运行时二进制，下载失败不影响登录验证。

---

## 三、编译（未压缩目录，最快）

只把应用编译成一个可直接运行的目录，不生成安装包：

```powershell
$env:NODE_OPTIONS = "--max-old-space-size=16384"
pnpm.cmd --filter desktop package
```

- 等的就是 electron-forge 的 `package` 目标
- 产物在 `apps/desktop/out/`，里面有 `*.exe`，双击即可运行
- 不生成安装程序，也不做签名，适合自己快速验证

> `NODE_OPTIONS` 必须给。默认堆在 6 GB 左右，这个仓库的 main + preload + renderer 会 OOM（实测 `FATAL ERROR: Reached heap limit`）。

---

## 四、打包（安装包）

### 方式 A：electron-forge make（生成本平台安装包）

```powershell
$env:NODE_OPTIONS = "--max-old-space-size=16384"
pnpm.cmd build
```

Windows 用 NSIS 生成 `Setup.exe`，macOS 生成 ZIP/DMG，Linux 生成对应包。产物在 `apps/desktop/out/make/`。

### 方式 B：仓库的正式打包入口（带区域、版本、签名流程）

```powershell
$env:NODE_OPTIONS = "--max-old-space-size=16384"
pnpm.cmd release:package -- --region dev --no-sign --skip-smoke
```

参数（来自 `apps/desktop/scripts/package-desktop.mjs`）：

| 参数 | 作用 |
|---|---|
| `--region cn\|global\|dev` | 构建区域；不带 `--version` 时默认 `global` |
| `--platform win32\|darwin\|linux` | 默认当前平台，**不支持交叉打包** |
| `--arch x64\|arm64` | 只打单架构 |
| `--no-sign` | 主动跳过签名（隐含 `--allow-unsigned`） |
| `--skip-smoke` | 跳过打包后的 smoke test（调试用） |

自己用就加 `--no-sign --skip-smoke`，否则会要求签名配置。

> **区域要和 auth 服务对齐**：客户端把构建区域归一成 `AUTH_REGION`（只有 `global` 会得到 `global`，其余都是 `cn`），而 `providers.region` 必须逐字相等。所以 `--region dev` 或 `--region cn` 配 `AUTH_REGION=cn` 的服务；`--region global` 必须配 `AUTH_REGION=global` 的服务，否则登录页直接 `REGION_MISMATCH`。

---

## 五、启动

### 5.1 dev 模式（推荐，唯一能验登录的方式）

**第一步：生成本地端点清单。** `config/endpoint.dev.json` 被 `.gitignore` 忽略（`.gitignore:169`），必须自己建。不要直接复制 `.example`——它的 `cdnBaseUrl` 是 `http://`，构建期校验要求 HTTPS。

在 `cindy/` 下创建 `config/endpoint.dev.json`：

```json
{
  "schemaVersion": 1,
  "authApiBaseUrl": "http://localhost:3344",
  "deviceLinkApiBaseUrl": "http://localhost:3335",
  "oauthBrokerApiBaseUrl": "http://localhost:3337",
  "ossApiBaseUrl": "http://localhost:3340",
  "heartbeatUrl": "http://localhost:3338",
  "telegramHookWsUrl": "",
  "feishuHookWsUrl": "",
  "discordHookWsUrl": "",
  "slackHookWsUrl": "ws://localhost:3334",
  "websiteUrl": "http://localhost:3000",
  "modelAccessApiBaseUrl": "http://localhost:3339",
  "voiceApiBaseUrl": "http://localhost:3342",
  "githubApiBaseUrl": "http://localhost:3336",
  "skillhubApiBaseUrl": "http://localhost:3341",
  "cindySkillHubApiBaseUrl": "http://localhost:3345",
  "pluginApiBaseUrl": "http://localhost:3343",
  "cdnBaseUrl": "https://localhost:3345/cindy",
  "mobileUpdateBaseUrl": "http://localhost:3346",
  "review": "0.1.0",
  "shareBaseUrl": "http://localhost:3350"
}
```

校验（同时确认没有 BOM）：

```powershell
node -e "JSON.parse(require('fs').readFileSync('config/endpoint.dev.json','utf8'));console.log('ok')"
```

**第二步：起 auth 服务**（另一个终端，`cindy-auth/` 目录需要你单独拷到测试机）：

```powershell
cd ..\cindy-auth
$env:AUTH_REGION = "cn"
$env:AUTH_FIXED_CODE = "123456"
node server.mjs
```

**第三步：起客户端**：

```powershell
cd cindy
$env:NODE_OPTIONS = "--max-old-space-size=16384"
pnpm.cmd --filter desktop dev -- --region=dev
```

首次编译 1–3 分钟，然后弹出登录窗口。邮箱随便填，验证码 `123456`。

### 5.2 已打包的应用

```powershell
# 未压缩目录
.\apps\desktop\out\<应用目录>\<应用>.exe
```

或者装 NSIS 的 `Setup.exe` 后从开始菜单启动。

注意：**打包版启动时会去 `<cdnBaseUrl>/endpoint.json` 拉线上清单**。如果你改了 `config/endpoint.json` 但 `cdnBaseUrl` 不可达，它会回落到**随包清单**（就是打包时 `config/endpoint.json` 的内容），所以只要把 `authApiBaseUrl` 写进那份文件并重新打包，仍然生效。但地址必须是 **https**。

---

## 六、验证成功的标志

auth 服务的 stdout 依次出现：

```
[code] me@example.com -> 123456
[login] me@example.com device=...
```

说明请求真的打到了自建服务并换到了 token。进一步确认 `data-dev.json` 里出现该用户。

客户端侧：进入主界面，不是登录页；重启应用后仍是登录态（走了 `/api/auth/refresh`）。

---

## 七、常见问题

| 现象 | 原因 | 处理 |
|---|---|---|
| `无法加载文件 ...pnpm.ps1，未对文件…进行数字签名` | PowerShell 执行策略 | 用 `pnpm.cmd`；或 `Set-ExecutionPolicy -Scope Process Bypass` |
| `This project is configured to use 11.28.9 of pnpm. Your current pnpm is vX` | 用了系统里的旧 pnpm | `corepack prepare pnpm@11.28.9 --activate`，并确保 PATH 上 corepack 的 shim 优先 |
| `客户端端点清单字段 cdnBaseUrl 必须是无凭据 HTTPS URL` | 复制了 `.example` 的 `http://` | 改成 `https://` |
| `客户端端点清单不是合法 JSON` | 文件带 UTF-8 BOM | 用 Node 重写，别用 `Set-Content -Encoding UTF8` |
| `FATAL ERROR: Reached heap limit` | 构建期堆不足 | `NODE_OPTIONS=--max-old-space-size=16384` |
| Electron 启动后立刻退出、连 userData 都不建 | 本机已有原版 Cindy 在跑（单实例锁），或隔离未生效 | 完全退出原版 Cindy；或加 `--isolated=dev` 参数 |
| 登录页报 `REGION_MISMATCH` | `providers.region` 与构建区域不一致 | 统一 `--region dev/cn` + `AUTH_REGION=cn`，或都用 `global` |
| 登录报 `AUTH_SERVICE_UNAVAILABLE` | auth 服务没起 / 端口不对 | `curl.exe http://127.0.0.1:3344/healthz` 应返回 `{"ok":true,...}` |
