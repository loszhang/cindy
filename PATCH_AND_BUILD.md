# Cindy 安卓 Google 登录补丁与 APK 打包说明

整理日期：2026-11-15
基线：分叉点 main @ b975dc39bc（Merge PR #10570）；本次同步点（2026-11-15 18:55）main @ ee9fb7349a（PR #10672）
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

- 本机 Mac mini 克隆：origin = loszhang/cindy（可推）；upstream = makecindy/cindy（只拉，push 已禁用防误推）。其他克隆（如 Windows 工作区）可能只有 origin，需自行补 upstream
- main 只跟 upstream 做 ff-only 同步，永不在 main 上改代码；改动都在 feat/google-browser-login
- 同步步骤见仓库父目录 SYNC.md（/Users/zhangjiji/projects/cindy-dev/SYNC.md，844B）；该文件在仓库外、不随 git 分发，其他克隆直接按下面命令操作；上游 PR 已明确不开，补丁只留在 fork

    git checkout main
    git fetch upstream
    git merge --ff-only upstream/main
    git push origin main
    git checkout feat/google-browser-login
    git merge main
    git push origin feat/google-browser-login

若只需同步 fork 的 main（不跟踪上游后续提交），git pull origin main 即可，不必配 upstream。

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

## 四之二、本地独立 release 包（离线可用）

不连 Metro、Mac 关机也能用的包。本机交付件：/Users/zhangjiji/projects/cindy-dev/cindy-release-arm64-v8a.apk（133.3 MiB）。

1. 构建（在 apps/mobile/android 下，env 与第四节相同）：

    cd /Users/zhangjiji/projects/cindy-dev/cindy/apps/mobile/android
    EXPO_PUBLIC_CINDY_AUTH_REGION=global CINDY_AGENT_NET_ANDROID_ABIS=arm64-v8a ANDROID_HOME=$HOME/Library/Android/sdk ANDROID_SDK_ROOT=$HOME/Library/Android/sdk ANDROID_NDK_HOME=$HOME/Library/Android/sdk/ndk/27.1.12297006 NODE_OPTIONS=--max-old-space-size=6144 ./gradlew :app:assembleRelease -Pandroid.enableMinifyInReleaseBuilds=false -Pandroid.enableShrinkResourcesInReleaseBuilds=false -PreactNativeArchitectures=arm64-v8a

2. 三个开关的原因：默认在 minifyReleaseWithR8 阶段失败过一次（日志 `Gradle build daemon has been stopped: stop command received`），关掉 minify/shrink 后 BUILD SUCCESSFUL in 2m 20s；限定 arm64 后 APK 从全 ABI 的 316M 降到 133M。产物在 apps/mobile/android/app/build/outputs/apk/release/app-release.apk，内嵌 assets/index.android.bundle（41.7 MB）。
3. 签名仍是 debug.keystore（build.gradle 的 release 也指向它），与原 debug 包同签名，可直接 adb install -r 覆盖，不丢登录态和本地数据；这不是商店正式包（无自有 keystore、无 R8 混淆、无 OTA/Play 配置）。
4. 校验与安装：

    $HOME/Library/Android/sdk/build-tools/37.0.0/apksigner verify --print-certs <apk>
    adb -s <serial> install -r <apk>

5. 不要照抄第四节第 1 步的 adb uninstall：会清掉登录态和本地数据，并把手机退回必须连 Metro 的 debug 包。

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
7. debug 包依赖 Metro：JS 从 Mac 的 Metro（localhost:8081）加载，关掉 expo 进程后 App 打不开属正常；测试期间保持 Mac 上进程存活。release 包内嵌 assets/index.android.bundle，不受此限制（见四之二）。
8. 连接稳定性：USB 序列号（6381CP260062098）最稳；无线地址（IP:port）会变、mDNS 条目会 offline，构建命令里的 --device 要填当次 adb devices 里状态为 device 的那条。
9. 本地无 self-host-regions.json 时回落空白模板：不烘焙官方 Google 配置，原生 Google 路径在本地包里不可用——正好纯粹验证浏览器回退；要验原生路需另配真实 client ID。
10. 同步上游的冲突面：补丁只动移动端 auth 的 6 个文件，上游改到同文件才可能冲突；自动化/手动同步都遵循冲突即停、回滚、不硬解。

## 七、状态记录（截至 2026-11-15 18:55）

- 补丁（ef2d086ca7）已提交并推送到 fork；feat/google-browser-login 完整包含 main
- 本次同步点（2026-11-15 18:55）：main 先 ff-only 到 ee9fb7349a（PR #10672），再以 d981976fc6 合入 feat/google-browser-login。本批 44 个提交 / 165 文件 / +8901/−263，与补丁 6 个文件零重叠、依赖与原生文件无变化，合并零冲突。合并后验证：定向 47/47、移动端回归 107/107、tsc --noEmit exit 0、git diff --check 通过。上游之后仍持续推提交，同步状态一律以 git rev-list --left-right --count main...upstream/main 为准，不要再写「当前已同步」
- 手机状态：2026-11-15 18:04 已装本地 release 包（133.3 MiB，内嵌 41.7 MB assets/index.android.bundle，apksigner 验签通过，adb install -r Success），离线启动为已登录主界面、无崩溃，8081 无 Metro，不再依赖电脑；交付件 /Users/zhangjiji/projects/cindy-dev/cindy-release-arm64-v8a.apk。两次合并（b975dc39bc..553c21b22f 的 443 文件、553c21b22f..ee9fb7349a 的 165 文件）都没有 android/、.so、CMake、gradle、Rust、app.config.js、lockfile，原生层无需重编；首次合并后重跑 expo run:android 为 41s BUILD SUCCESSFUL、864 任务中 838 个 UP-TO-DATE、APK 时间戳仍是 16:08，只是 Gradle 判定内容一致。注意：该离线包内嵌的是 18:01 的 JS bundle，本次 18:55 合并后代码又前进 44 个提交，需按四之二重新打包安装才与代码一致（记录时 USB 未连接）
- 待办：上游自动监控方案（本机 LaunchAgent / GitHub Actions）尚未选定；手机离线包需按四之二重打才能跟到最新代码
