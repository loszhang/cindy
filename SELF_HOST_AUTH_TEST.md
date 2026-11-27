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
| Node.js | **≥ 22.19.0**（推荐按 `.nvmrc` 用 22.22.3） | `node -v` |
| pnpm | **11.28.9**（`packageManager` 指定，`engineStrict` 会拦旧版） | `corepack prepare pnpm@11.28.9 --activate` 然后 `corepack pnpm -v` |
| Git LFS | 必需（仓库有 LFS 资源） | `git lfs version` |

> **Node 不能只满足根 `package.json` 的 `engines.node >= 22.13`。**
> 依赖 `@earendil-works/pi-ai@0.85.1` 要求 `node >= 22.19.0`，用 22.13–22.18 会在
> `pnpm install --frozen-lockfile` 阶段直接报 `ERR_PNPM_UNSUPPORTED_ENGINE`。
> 本仓 `.nvmrc` 固定为 22.22.3，优先按它准备。

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
| `corepack prepare` 报 `Cannot find matching keyid` | Node 自带 corepack 版本太旧，不认识 pnpm 11.28.9 的签名密钥 | 在该 Node 安装目录下用 `npm install -g pnpm@11.28.9` 装好后直接调 `pnpm.cmd` |
| `ERR_PNPM_UNSUPPORTED_ENGINE`（如 `@earendil-works/pi-ai` 要求 `>=22.19.0`） | Node 低于依赖要求（22.13–22.18 会中招） | 升级到 22.19+，推荐 `.nvmrc` 的 22.22.3 |
| `客户端端点清单字段 cdnBaseUrl 必须是无凭据 HTTPS URL` | 复制了 `.example` 的 `http://` | 改成 `https://` |
| `客户端端点清单不是合法 JSON` | 文件带 UTF-8 BOM | 用 Node 重写，别用 `Set-Content -Encoding UTF8` |
| `FATAL ERROR: Reached heap limit` | 构建期堆不足 | `NODE_OPTIONS=--max-old-space-size=16384` |
| Electron 启动后立刻退出、连 userData 都不建 | 本机已有原版 Cindy 在跑（单实例锁），或隔离未生效 | 完全退出原版 Cindy；或加 `--isolated=dev` 参数 |
| 登录页报 `REGION_MISMATCH` | `providers.region` 与构建区域不一致 | 统一 `--region dev/cn` + `AUTH_REGION=cn`，或都用 `global` |
| 登录报 `AUTH_SERVICE_UNAVAILABLE` | auth 服务没起 / 端口不对 | `curl.exe http://127.0.0.1:3344/healthz` 应返回 `{"ok":true,...}` |
| 启动器报 `STARTUP_TIMEOUT`，但应用随后能起来 | 首次冷构建超过 180 秒 readiness 上限（非真实失败） | 冷构建阶段已放宽到 600 秒；看到该 verdict 时先看 Electron 是否仍在编译，不要直接判失败 |
| 窗口出现后空白好几分钟 | 重启清 Vite 缓存导致 renderer 重新预打包，`index.html` 无静态占位 | 已改为按依赖/导出面指纹判断：输入不变则复用缓存；只有改导出面才重新冷编译 |

---

## 八、本机实测记录（2026-11-27）

按本文在 Windows 上完整走通一次，供复现时对照：

| 步骤 | 结果 |
|---|---|
| 依赖安装 | Node 22.22.x + pnpm 11.28.9，`pnpm install --frozen-lockfile` 成功；agent 二进制 best-effort 下载完成 |
| auth 服务 | `AUTH_REGION=cn` + `AUTH_FIXED_CODE=123456`，`/healthz` 返回 `{"ok":true,"region":"cn"}` |
| dev 客户端 | `pnpm restart:desktop:remote --region=dev --isolated=dev`（首次冷构建较慢） |
| 登录 | 服务端 stdout 依次出现 `[code]` / `[login]`，`data.json` 出现该用户，客户端进入主界面 |
| 重启保持登录 | 冷启动后未回登录页；refresh token 哈希发生变化（旧 token 被轮换），证明走了 `/api/auth/refresh` |
| 缓存复用 | 第二次重启打印 `Desktop dev cache reused (dependency inputs unchanged)`，无需重新预打包 |

过程中遇到并修复的问题：

1. **Node 版本写低了**：只写 22.x，但依赖 `@earendil-works/pi-ai` 要求 `>=22.19.0` → 已修正环境要求表与常见问题。
2. **启动器误报 `STARTUP_TIMEOUT`**：冷构建被算进 180 秒 readiness 上限 → 改为两段式，冷构建/首个窗口 600 秒，`window-ready` 后 180 秒。
3. **窗口空白数分钟**：每次重启清 Vite 缓存，renderer 重新预打包 → 改为按 lockfile / workspace / patch / 内部包导出面指纹判断，输入不变则复用缓存。

> 打包版只能验证“能启动、能开窗”：它只接受 https 端点，本机 http auth 无法在打包版登录；登录链路仍需在 dev 模式验证。

---

## 九、换机器 / 新 clone 的注意事项

下面这些配置都是**按机器/按 clone** 的，不会随 Git 带过去，新测试机必须逐项准备：

| 项目 | 说明 |
|---|---|
| Node | ≥ 22.19.0，推荐 `.nvmrc` 的 22.22.3；低于此值 `pnpm install` 报 `ERR_PNPM_UNSUPPORTED_ENGINE` |
| pnpm | 11.28.9；若 `corepack prepare` 报 `Cannot find matching keyid`，在该 Node 目录下 `npm install -g pnpm@11.28.9` 后直接用 `pnpm.cmd` |
| PowerShell | 执行策略会拦 `pnpm.ps1`，统一用 `pnpm.cmd` |
| `config/endpoint.dev.json` | 被 `.gitignore` 忽略，需按 §5.1 自行创建；不要直接复制 `.example`（`cdnBaseUrl` 是 http） |
| `cindy-auth/` | 服务端不在本仓，需单独带到测试机；`data.json` 含 refresh token，不要提交进仓库 |
| `apps/desktop/.env` | 由启动器按 `--region=dev` 生成，不用手工维护 |

**提交代码前的 DCO 要求**（只要在新机器上 commit 就必须做）：

1. 每个 commit 都要带与 author/committer 一致的 `Signed-off-by`。最稳妥是始终用 `git commit -s`。
2. 在每个新 clone 上运行一次：

   ```powershell
   pnpm.cmd dco:install-hook
   ```

   hook 装在 `.git/hooks/`，**不会随仓库同步**，每台机器、每个 worktree 都要单独装。装完即使忘记 `-s`，hook 也会自动补签。

3. 提交前自查：`pnpm.cmd check:dco`。它通过才算 branch 上的提交都合规。

> Windows 上 `pnpm.cmd dco:install-hook --check` 可能提示 hook “NOT executable”——因为 `core.fileMode=false` 时 Node 按 Unix 执行位判断会误报；hook 实际会被 Git 执行。用一次空的测试提交（`git commit --allow-empty -m "test dco hook"`）看 message 里有没有 `Signed-off-by` 即可确认。

### 已经 clone 过的测试机如何继续

分支历史曾为修复 DCO 做过一次 `--signoff` rebase 并 force push。**不需要重新 clone**，但旧 clone 已与远端分叉，按情况处理：

| 本地状态 | 操作 |
|---|---|
| 不改代码、没有本地提交 | `git fetch origin` 后 `git reset --hard origin/feat/self-hosted-auth` |
| 有本地提交或未提交改动 | 先 `git stash push -u` 和 `git branch my-local-work` 保存，再 fetch + reset；之后 `git stash pop`，本地提交用 `git cherry-pick <sha>` 逐个接回 |
| 不再与远端同步 | 可以原样继续测试（rebase 只改哈希，文件内容一致），但不要从这个旧状态 `push` |

注意：

- `reset --hard` 只影响 Git 跟踪的文件；被 gitignore 的 `config/endpoint.dev.json`、`node_modules` 和仓库外的 `cindy-auth/` 不会被动到，不必重新配置或安装依赖。
- **不要用 `git pull`**：旧历史与重写后的历史分叉，`pull` 会生成把两者合并的 merge commit。
- 对齐后若这台机器还没装过 DCO hook，按上面的 `pnpm.cmd dco:install-hook` 装一次。
