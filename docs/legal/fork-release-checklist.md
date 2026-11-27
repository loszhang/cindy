# Fork 开源发布前 checklist

> **适用范围**：基于本仓 fork，使用自己的商标和名字，以开源、免费方式分发桌面客户端。
> **依据**：根 `LICENSE`（Apache-2.0）、根 `NOTICE`、`README.md` §License、`docs/legal/README.md`。
> **说明**：Apache-2.0 允许修改、再分发、商用和不收费发布；开源免费不减少下列义务。
> 核心边界是：**源码许可可以继承，商标与官方身份不能继承**。本文是按仓库现有规则整理的工程清单，不是法律意见；对外发布前请自行做法务复核。

## 1. 许可、署名与第三方声明

- [ ] fork 仓库根保留 `LICENSE`（Apache-2.0）和 `NOTICE`（含 `Cindy — Copyright 2026 XD Inc.`），不删改。
- [ ] 在 README 写明来源：基于 Cindy 修改，给出上游仓库链接，并说明主要改动方向。
- [ ] 按 Apache-2.0 §4(b) 对修改过的文件保留/标注变更声明，不移除上游版权与许可信息。
- [ ] 保留 `docs/legal/notices/`；依赖变化时用 `pnpm licenses:generate` 重新生成，不手工编辑生成物。
- [ ] 发布二进制或安装包时，随包带上 `THIRD-PARTY-NOTICES.txt`、`THIRD-PARTY-RESTRICTED.txt` 和对应平台 SBOM；桌面生成副本在 `apps/desktop/resources/`。
- [ ] vendored 源码、原生二进制与字体的上游 `LICENSE` / `NOTICE` / `COPYING` 保持相邻保留，具体路径见 `docs/legal/README.md` 的固定路径表。
- [ ] 项目整体许可沿用 Apache-2.0，或选择与之兼容的许可证；不要把原 Apache-2.0 代码整体改成不兼容许可（例如 GPLv2）。

## 2. 商标与品牌

- [ ] 使用自有或已获授权的新商标，**不要**使用 Cindy 名称、logo、appicon 或品牌文案。
- [ ] 不暗示与 XD Inc. / Cindy 官方存在背书、关联、兼容承诺或官方支持。
- [ ] 已知边界：`README.md` §License 写明 trademarks 等单独标识材料不受仓库级 Apache-2.0 授权。

## 3. 构建期身份（改名必须改这里）

身份是构建期常量，不是运行时开关；改完需要重新打包。`apps/desktop/forge.config.ts` 从这些常量派生 exe、安装目录、快捷方式、AUMID、UTI 等。

- [ ] `packages/maker-shared/src/branding.ts`：`BRAND_NAME`。
- [ ] `packages/maker-shared/src/brandIdentity.ts`：
  - `executableNameByRegion`（exe / 安装目录 / 快捷方式名）
  - `appIdByRegion`（Windows AUMID、NSIS appId、macOS bundle id）
  - `userDataDirNameByRegion`（用户数据目录）
  - `primaryScheme`（深链 scheme）
  - `updaterName`、`dbFilePrefix`、`cdnPrefix`
- [ ] `legacySchemes`、`legacyUserDataDirNames*`、`legacyDbFilePrefixes` **只增不减**，保留兼容锚点。
- [ ] 同步镜像常量：`scripts/restart-desktop-remote.mjs` 的 `BRAND_USER_DATA_DIR_NAME` 由 `brand-identity-sync.test.mjs` 约束，改名后一并更新。
- [ ] 以 `--region dev` 打包为起点可获得独立身份（`CindyDev` / `com.xd.cindydev` / `CindyDev`），再按上面字段改成你自己的名字。

## 4. 服务端与基础设施

- [ ] 自建或关闭认证、CDN、更新服务、统计上报；不要沿用官方端点。
- [ ] 不要把官方客户端的 refresh token、设备身份或用户数据复制进 fork。
- [ ] 打包版只接受 https 端点；自建 auth 若用 http，只能在 dev 模式联调，不能进发行包。
- [ ] 用户凭证、令牌、授权文件与 `data.json` 等持久化数据不得进入仓库或任何 Git 跟踪路径。

## 5. 必须停下核对的红线

- [ ] 修改 `updaterName` 或 Electron 更新服务前：先读 `docs/dev-rules/cindy-updater.md`，并取得仓库维护者确认。
- [ ] 修改 `primaryScheme` 等跨端协议前：先读 `docs/dev-rules/protocol-compatibility.md`，并完成同步适配与互通验证。
- [ ] 修改进入模型 system 段的提示词前：先读 `docs/dev-rules/maker-core-and-agent-behavior.md`。
- [ ] 不改动 `tools/<kind>/latest.json` 里由维护者统一判断的 Desktop runtime 二进制版本 pin。
- [ ] 不删除或弱化任何上游兼容锚点、日志脱敏规则、插件迁移路径。

## 6. 验证与交付

- [ ] 跑 `pnpm rules:changed` 读完全部命中规则，再按风险跑定向测试与类型检查（默认 `pnpm test:unit:related`）。
- [ ] 纯文档改动跑 `pnpm check:dev-docs`。
- [ ] 打包后验证：新名字、安装目录、快捷方式、AUMID 正确；与原版 Cindy / CindyGlobal 能同机并存且数据互不干扰。
- [ ] 验证深链、更新器与自建 auth 都指向你自己的基础设施，没有请求官方 `auth.cindy.*`。
- [ ] 每个提交带与 author / committer 一致的 DCO `Signed-off-by`（`git commit -s`）。
