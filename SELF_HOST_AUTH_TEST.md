# 自建 auth 本地联调指南（fork 专用）

> 说明：本文只存在于本 fork 的 `feat/self-hosted-auth` 分支，用于验证"客户端登录到自建 auth 服务"。
> 上游仓库没有这份文件。自建 auth 服务本身**不在本仓**（仓库边界：服务端在独立仓库），
> 需要单独把 `cindy-auth/` 目录带到测试机上。

## 目标

在**一台没有安装 Cindy 的电脑**上，用 dev 模式启动客户端，让它的登录请求打到本机运行的
自建 auth 服务（`http://localhost:3344`），确认整条链路：

```
登录页 → GET /api/auth/providers → POST /api/auth/email/request-code
      → POST /api/auth/email/verify-code → 进主界面 → POST /api/auth/refresh
```

在没有原版 Cindy 运行的机器上测试，可以排除单实例锁与 profile 冲突的干扰。

## 前置

- Node.js **22.x**（`node -v` 确认；仓库 `engines.node >= 22.13`）
- pnpm **11.28.9**（仓库 `packageManager` 指定）。用 corepack 取，不要用系统里的旧版：
  ```powershell
  corepack prepare pnpm@11.28.9 --activate
  corepack pnpm -v    # 应输出 11.28.9
  ```
- Git LFS

## 1. 克隆并安装

```powershell
git clone -b feat/self-hosted-auth https://github.com/loszhang/cindy.git
cd cindy
git lfs pull
pnpm install --frozen-lockfile
```

Electron 下载失败时加镜像后重跑：

```powershell
$env:ELECTRON_MIRROR = "https://npmmirror.com/mirrors/electron/"
pnpm install --frozen-lockfile
```

安装结束若显示 "3/4 agent 二进制未就位（best-effort，不阻断）"，属正常，不影响登录验证。

## 2. 生成本地端点清单

`config/endpoint.dev.json` 被 `.gitignore` 忽略（`.gitignore:169`），需要自己创建。
**不要**直接复制 `.example`：它的 `cdnBaseUrl` 是 `http://`，而构建期校验要求 HTTPS，会报
`客户端端点清单字段 cdnBaseUrl 必须是无凭据 HTTPS URL`。

在 `cindy/` 下创建 `config/endpoint.dev.json`，内容：

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

只有 `authApiBaseUrl` 是我们真正要用的；其余留空或指向不存在的本机端口都可以，
对应能力会以非致命方式失败（dev 允许 plain http）。

写完用 Node 校验一次，确保没有 BOM（PowerShell 5.1 的 `Set-Content -Encoding UTF8` 会写 BOM，
JSON 解析会失败）：

```powershell
node -e "JSON.parse(require('fs').readFileSync('config/endpoint.dev.json','utf8'));console.log('ok')"
```

## 3. 启动自建 auth 服务

把 `cindy-auth/` 目录放到测试机上（与 `cindy/` 平级即可），然后：

```powershell
cd ..\cindy-auth
$env:AUTH_REGION = "cn"
$env:AUTH_FIXED_CODE = "123456"
node server.mjs
```

预期输出：

```
cindy-auth listening on http://0.0.0.0:3344 (region=cn)
fixed verification code enabled: 123456
```

自检：

```powershell
curl.exe http://127.0.0.1:3344/healthz
# {"ok":true,"region":"cn","users":0}
```

> `AUTH_REGION` 必须是 `cn`：dev 构建的 `AUTH_REGION` 在客户端侧被归一成 `cn`
> (`authManager.ts` 里 `VITE_CINDY_AUTH_REGION === 'global' ? 'global' : 'cn'`)，
> 而 `providers.region` 必须逐字相等，否则 `REGION_MISMATCH`。

## 4. 启动客户端（dev 模式）

另开一个终端：

```powershell
cd cindy
$env:NODE_OPTIONS = "--max-old-space-size=16384"   # 构建期堆，低于 8G 容易 OOM
pnpm --filter desktop dev -- --region=dev
```

首次会编译 main/preload/renderer，约 1–3 分钟。窗口出现后：

1. 登录页输入任意邮箱（如 `me@example.com`）
2. 验证码输 `123456`
3. 应进入主界面

`--region=dev` 会设置 `XDT_ENDPOINT_MANIFEST_FILE=config/endpoint.dev.json`，
并让 dev 的 userData 与正式 profile 隔离。

## 5. 判定成功

auth 服务的 stdout 应依次出现：

```
[code] me@example.com -> 123456
[login] me@example.com device=...
```

这两行说明客户端确实把请求打到了自建服务，并且拿到了 token。

进一步确认可以看 auth 服务的数据文件 `data-dev.json` 里出现了该用户：
`{"users":{"email:me@example.com":{...}}}`。

## 已知坑（本机实测）

| 现象 | 原因 | 处理 |
|---|---|---|
| `客户端端点清单字段 cdnBaseUrl 必须是无凭据 HTTPS URL` | `.example` 里是 `http://` | 按本文第 2 步改成 `https://` |
| `客户端端点清单不是合法 JSON` | 文件被写成 UTF-8 BOM | 用 Node 重写，不要用 `Set-Content -Encoding UTF8` |
| `Electron` 启动后立刻退出、连 userData 都不建 | 本机已有原版 Cindy 在跑（单实例锁），或隔离未生效 | 先完全退出原版 Cindy；或加 `--isolated=dev` |
| `FATAL ERROR: Reached heap limit` | 构建期堆不足 | `NODE_OPTIONS=--max-old-space-size=16384` |
| `pnpm : 无法加载文件 ...pnpm.ps1` | PowerShell 执行策略 | 用 `pnpm.cmd` 调用，或 `Set-ExecutionPolicy -Scope Process Bypass` |
