# `.ci/` — 本 fork 自建构建层的说明

本目录（以及 `.github/workflows/`）里的文件**只属于本 fork**，上游 `zai-org/ZCode` 一个都没有。
因此 `sync-upstream.yml` 里的 rebase 天然无冲突；万一上游将来新增了同名路径，
rebase 会**响亮失败**而不是静默覆盖 —— 这是刻意设计。

## 本 fork 只加了什么

| 文件 | 作用 |
| --- | --- |
| `.github/workflows/sync-upstream.yml` | 每天 04:23 UTC（北京时间 12:23）把 fork rebase 到 `zai-org/ZCode` 的 `main` |
| `.github/workflows/build-desktop.yml` | 四平台矩阵构建 + 发布到 Release `ci-latest` |

## 补的是哪些平台

上游 v3.14.3 release 只有两个产物：

```
ZCode-3.14.3-mac-arm64.dmg
ZCode-3.14.3-win-x64.exe
```

本 fork 补齐 **macOS Intel (x64)** 与 **Linux**（AppImage / deb / rpm / pacman），
并把 arm64、win-x64 也重建一份，让四个目标来自同一 commit、同一份源码。

| 目标 | Runner | 产物 |
| --- | --- | --- |
| `mac-x64` | `macos-15-intel` | `.dmg` + `.zip` |
| `mac-arm64` | `macos-15` | `.dmg` + `.zip` |
| `win-x64` | `windows-2022` | `.exe`（NSIS） |
| `linux-x64` | `ubuntu-24.04` | `.AppImage` / `.deb` / `.rpm` / `.pkg.tar.zst` |

## 为什么不需要免签名补丁

上游 `packages/desktop/electron-builder.config.js` 已把签名与公证做成**显式 opt-in**：

```js
const shouldEnableMacSigning =
  process.env.ZCODE_ENABLE_MAC_SIGN === "1" && Boolean(macSigningIdentity);

mac: {
  identity: shouldEnableMacSigning ? macSigningIdentity : null,
  notarize: false,              // 硬编码关闭，不读任何 Apple 凭据
  hardenedRuntime: shouldEnableMacSigning,
}
```

Windows 签名同样受 `ZCODE_ENABLE_WINDOWS_SIGN=1` 门控。
于是「不设这两个变量」就是官方**支持**的免签名路径 —— 不需要 `git apply` 任何补丁，
构建期不改动工作树，rebase 永远不会因为构建期残留而失败。

> 与 `HunterXing/deepseek-harness` 那套的本质区别：那边的
> `electron-builder.config.mjs` 在**配置加载期**就硬校验签名凭据，
> 免签名必须靠构建期补丁硬绕过；ZCode 这边上游主动把签名做成开关，不需要绕过。

## 切换到官方签名流程

具备 Apple Developer Program 账号与 Windows 代码签名证书后：

1. 在仓库 Settings → Secrets and variables → Actions 添加：
   - `APPLE_SIGNING_IDENTITY`（形如 `Developer ID Application: XXX (TEAMID)`）
   - `CSC_LINK` + `CSC_KEY_PASSWORD`（p12 证书与密码）
2. 把 `build-desktop.yml` 里 `ZCODE_ENABLE_MAC_SIGN: '0'` 改成 `'1'`。
3. `notarize: false` 是上游硬编码的，公证仍需另加步骤 —— 按上游发布文档补。

## 必须配置的 secret

| Secret | 作用域 | 为什么需要 |
| --- | --- | --- |
| `ZCODE_SYNC_TOKEN` | 仅 `sync-upstream.yml` | 推送带 `workflow` scope 的 **classic PAT** |

**为什么不能用 `GITHUB_TOKEN`**：rebase 会把上游新增/修改的 `.github/workflows/*.yml`
一并带进来，而 GitHub 硬性禁止 `GITHUB_TOKEN` 修改 workflow 文件（防 workflow 自我提权），
push 会被拒：`remote rejected ... without 'workflows' permission`。
这个限制**不是** `permissions:` 块能解决的 —— 官方 `GITHUB_TOKEN` 权限键里根本没有
`workflows`（只有 actions / checks / contents / deployments 等十六个）。

**建 PAT 的最小勾选**：`repo` + `workflow`。fine-grained PAT 不行，有一堆限制。

**凭据传递方式**：走 `http.extraHeader`，**不塞进 remote URL**。公开仓库的 Actions 日志
任何人可读，而 `git push` 失败时 git 会把 remote URL 原样打进错误信息 —— 塞 URL 等于公开 token。

## 产物安装（未签名）

- **macOS**：右键 → 打开，绕过 Gatekeeper。未公证的包不会出现在
  「系统设置 → 隐私与安全性 → 已允许下载的应用」列表里。
- **Windows**：SmartScreen 弹窗点「更多信息 → 仍要运行」。
- **Linux**：
  - AppImage —— `chmod +x ZCode-*-linux-x64.AppImage && ./ZCode-*-linux-x64.AppImage`
  - deb —— `sudo dpkg -i ZCode-*-linux-x64.deb`
  - rpm —— `sudo dnf install ./ZCode-*-linux-x64.rpm`（面向 RHEL 8+ / glibc 2.28+）
  - pacman —— `sudo pacman -U ZCode-*-linux-x64.pkg.tar.zst`

## 排障

- **run 显示 success 但 fork 没更新** —— 检查 `git merge-base --is-ancestor` 的方向。
  方向写反（判 `HEAD` 是否是 `upstream` 的祖先）会让上游一前进就静默跳过。
  本文件的写法在 `sync-upstream.yml` 里有注释说明。
- **构建失败在 `pnpm install`** —— 上游 lockfile 偶有与 `package.json` 不同步的历史提交。
  流水线用 `--no-frozen-lockfile`，不用上游的锁死安装。
- **构建失败在 `bundle:desktop`** —— 用 `gh run view --log-failed` 拿到失败步骤；
  若返回空，改用 `gh api repos/HunterXing/ZCode/actions/runs/<ID>/logs` 下载 zip 解压。
- **产物连不上正式服务** —— 检查 `ZCODE_ENV: production` 是否在。
  不设它会退化成 Preview 身份 + 测试后端配置。
