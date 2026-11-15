# Cindy 安卓 Google 登录补丁与 APK 打包说明

整理日期：2026-11-15
基线：main @ b975dc39bc（Merge PR #10570）
补丁提交：ef2d086ca7 fix(mobile-auth): use browser PKCE for Google on Android without Play Services
分支：feat/google-browser-login（fork：loszhang/cindy；上游：makecindy/cindy）

## 一、修改内容

### 问题

Global Android 的 Google 登录只有原生 GMS 一条路，没有浏览器兜底：

- 登录模式只由构建时是否注入 GOOGLE_WEB_CLIENT_ID 决定，不查运行时有没有 GMS；
- 无 GMS 设备上要么按钮直接不显示（未注入 ID 时 mode 为 null），要么点了一定在 hasPlayServices 处失败；
- browser PKCE 回退分支原本只给了 Apple（global + android），Google 不在内。

### 改动（6 个文件，+166/−8）

1. apps/mobile/src/auth/mobileSocialLoginMode.ts（模式规则）
   - resolveMobileSocialLoginMode：Google 与 Apple 同支，nativeSupported=false 且 region=global 且 platform=android 时返回 'browser'；native 仍优先。
2. apps/mobile/src/auth/nativeSocial.ts（GMS 运行时探测）
   - isNativeSocialProviderAvailable 对 Google Android 新增静默探测：动态 import GoogleSignin，configure 后调 hasPlayServices({ showPlayServicesUpdateDialog: false })，不弹更新对话框；异常一律返回 false。
3. apps/mobile/src/auth/useMobileSocialProviderModes.ts（探测接入 UI 状态）
   - 新增 androidGoogleAvailable 状态，仿微信 iOS 现有模式：仅 global + android + 服务端下发 google 时启用；初始 false，挂载与 App 回前台时刷新，带竞态防护（latestRequest + cancelled）与订阅清理；
   - Google 在 Android 的 nativeSupported = 构建时支持 && 探测通过。
4. apps/mobile/app/(auth)/login.tsx（按钮分发）
   - Google/微信圆钮循环里，Google 且 mode='browser' 时 dispatch start-social-browser（带 label: socialLabel(provider)），其余保持 native-social 原字面不变（守护测试锚点完好）。
5. apps/mobile/src/auth/__tests__/mobileSocialLoginMode.test.ts（+3 用例）
   - global android 无原生支持 → browser；cn android → null；iOS → null。
6. apps/mobile/src/auth/__tests__/useMobileSocialProviderModes.test.tsx（+3 用例）
   - 探测 false → browser；探测 true → native；回前台后 GMS 出现 → 切 native。

### 改后行为矩阵（Google）

- Android + global + 有 GMS：native（不变）
- Android + global + 无 GMS：browser PKCE，经系统浏览器完成，cindy:// 回调回 App（本次新增，真机已验）
- Android + cn：无回退（Google 本就不允许出现在 cn 线）
- iOS / 微信 / Apple：全部不变

### 验证结果

- 定向测试 47/47 通过（mobileSocialLoginMode 10 + useMobileSocialProviderModes 11 + mobileAuthServerLogin 守护 26）
- typecheck 通过（需加大 Node 堆，见注意事项 2）
- git diff --check 通过
- 真机端到端通过：Coolpad CPDAY11（Android 16，零 GMS 包）上按钮可见 → 系统浏览器 feimi browser 打开 Google 登录页（继续前往 Cindy）→ 登录 → cindy:// 回跳 MainActivity → 已登录主界面

## 二、仓库与分支结构

- origin = loszhang/cindy（可推）；upstream = makecindy/cindy（只拉，push 已禁用防误推）
- main 只跟 upstream 做 ff-only 同步，永不在 main 上改代码；改动都在 feat/google-browser-login
- 同步步骤见同目录 SYNC.md；上游 PR 已明确不开，补丁只留在 fork
- gh 2.104.0 已装并登录（账号 loszhang），git user.name 已设 loszhang

## 三、打包前置条件（本机 Mac mini，arm64 / 8GB）

- Node v22.23.2（nvm），pnpm 11.28.9（仓库 packageManager 指定）
- JDK 23 可用（Gradle 9.4.1 自带下载，支持新 JDK；不需要 JDK 17）
- Android SDK：~/Library/Android/sdk，platform 34/36、build-tools 34–37、NDK 27.1.12297006（缺的版本 Gradle 会自动装）
- Rust：cargo/rustup + 四个 Android target 已装；必须有 cargo-ndk（cargo install cargo-ndk，本机为 4.1.2），缺失会在构建约 19 分钟后报 AGENT_NET_ANDROID_BUILD_FAILED
- 依赖：仓库根目录 pnpm install --frozen-lockfile（本机耗时 34 分钟，见注意事项 3）
- 手机：开发者选项 + USB 调试已授权，adb devices 显示 device（不是 unauthorized / offline）

## 四、打包与安装流程（debug 包）

1. 同包名冲突处理：debug 包与官方全球版同为 com.xd.cindy 但签名不同，装不上覆盖。先卸载官方版（数据会清）：
   adb -s <serial> uninstall com.xd.cindy
   官方版为 v0.1.39 / versionCode 47；debug 包为 0.1.0 / versionCode 1，靠版本号区分。
2. 构建并安装（一条命令完成 prebuild + 编译 + 安装 + 起 Metro）：
   cd /Users/zhangjiji/projects/cindy-dev/cindy/apps/mobile
   EXPO_PUBLIC_CINDY_AUTH_REGION=global CINDY_AGENT_NET_ANDROID_ABIS=arm64-v8a ANDROID_HOME=$HOME/Library/Android/sdk ANDROID_SDK_ROOT=$HOME/Library/Android/sdk ANDROID_NDK_HOME=$HOME/Library/Android/sdk/ndk/27.1.12297006 NODE_OPTIONS=--max-old-space-size=6144 pnpm exec expo run:android --device <serial>
   - region 必须 global：默认 cn 线根本不配 Google；
   - CINDY_AGENT_NET_ANDROID_ABIS=arm64-v8a：Rust 库只编手机这一个架构，省一大截时间；
   - 首次构建：prebuild 生成 android/、下载 Gradle 9.4.1、自动补 SDK/NDK、编 Rust 与全部原生模块，耗时长；产物在 apps/mobile/android/app/build/outputs/apk/debug/app-debug.apk（约 137M）。
3. 手机上点掉安全管家（com.freeme.secureguard）的 USB 安装确认框，adb install 才会继续（见注意事项 6）。
4. 增量重建：改 env 后先在 android/ 里 ./gradlew --stop 停旧 Daemon，再重跑同一条命令；缓存命中时全程约 2 分 46 秒（864 任务中 806 个 UP-TO-DATE）。构建中途进程被杀不用 clean，重跑即续。
5. 只装现成 APK：adb -s <serial> install -r -d --user 0 <app-debug.apk 路径>

## 五、真机验证方法

- adb -s <serial> shell dumpsys package com.xd.cindy 看版本号；dumpsys activity activities 看前台是否为 com.xd.cindy/.MainActivity
- 看不到截图时用 uiautomator dump 导 UI 树，grep text/content-desc 即可确认页面元素（登录页 / 浏览器里的 Google 页 / 登录后主界面）
- 密码必须机主本人在手机上输，任何环节不要把密码交给脚本或日志

## 六、注意事项（踩过的坑）

1. 全量 git clone 会超时（600 秒未拉完）：先 --depth 1 浅克隆顶住，之后 git fetch --unshallow 补全历史；浅克隆不能正常 merge 上游，补全前不要做同步。
2. tsc --noEmit 在 8GB 机器上默认堆会 OOM（exit 134，无任何 TS 诊断）：加 NODE_OPTIONS=--max-old-space-size=6144 再跑，属环境问题不是代码错误。
3. pnpm install 很慢：postinstall 还要下 Claude（308M）、Codex（158M）、Pi（98M）等二进制并补装 Electron；日志出现 1/4 agent 二进制未就位是 best-effort 警告，不阻断移动端构建与测试，不要中途杀进程。
4. cargo-ndk 缺失是晚期失败：构建跑了 18 分 43 秒才在 agent-net 的 Rust 检查处报错；开工前先 cargo ndk --version 自检。
5. Gradle Daemon 会缓存环境变量：改了 ABI / NDK / region 后不重启 Daemon，新 env 传不进去。
6. 国产机安装确认：Coolpad 的安全管家会拦截 USB 安装，adb 进程会一直挂着等确认，Mac 侧看不到报错，只能去手机上点允许/继续安装。
7. debug 包依赖 Metro：JS 从 Mac 的 Metro（localhost:8081）加载，关掉 expo 进程后 App 打不开属正常；测试期间保持 Mac 上进程存活。
8. 连接稳定性：USB 序列号（6381CP260062098）最稳；无线地址（IP:port）会变、mDNS 条目会 offline，构建命令里的 --device 要填当次 adb devices 里状态为 device 的那条。
9. 本地无 self-host-regions.json 时回落空白模板：不烘焙官方 Google 配置，原生 Google 路径在本地包里不可用——正好纯粹验证浏览器回退；要验原生路需另配真实 client ID。
10. 同步上游的冲突面：补丁只动移动端 auth 的 6 个文件，上游改到同文件才可能冲突；自动化/手动同步都遵循冲突即停、回滚、不硬解。

## 七、当前状态（2026-11-15 整理时）

- 补丁已提交并推送到 fork，工作区干净
- 手机上装的是基于 b975dc39bc + 补丁的 debug 包，已登录可用
- 待办：上游 main 已继续前进（最后一次成功 fetch 到 ae78647b81，本地 main 落后 149 个提交，已确认与补丁文件零重叠），合并一版 + 重装手机尚未执行；上游自动监控方案（本机 LaunchAgent / GitHub Actions）尚未选定
