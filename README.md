# Friday.build

Friday 的构建产物与发布页。二进制由 GitHub Actions 从源码仓库自动构建后发布到本仓库的 Releases：macOS 版用
Developer ID 证书签名并经 Apple 公证，Windows / Linux 版（Qt）每个平台一套包（Windows：安装程序 + 便携版 zip；
Linux：deb + AppImage + tar.gz）。v2.0 起 Friday 只接在线大模型、不带本地推理，不再分 CUDA / Vulkan 两套包
（v2.0 之前的源码照旧出两套）。

## 产物

| 文件 | 平台 | 说明 |
| --- | --- | --- |
| `Friday-<标签>-macOS-arm64.dmg` | macOS 26+，Apple Silicon | 打开后双击图标运行（也可拖到「应用程序」） |
| `Friday-<标签>-macOS-arm64.zip` | 同上 | 纯 app 包，自动更新用它 |
| `Friday-<标签>-windows-x64-setup.exe` / `.zip` | Windows 10 / 11 x64 | 安装程序 / 便携版 |
| `Friday-<标签>-linux-x64.deb` / `.AppImage` / `.tar.gz` | Linux x64，glibc 2.35+ | deb 给 Debian / Ubuntu 系 |
| `Friday-<标签>-windows-x64-upgrade-{cuda,vulkan}.zip`、`Friday-<标签>-linux-x64-upgrade-{cuda,vulkan}.tar.gz` | — | 只给 1.x 的 CUDA / Vulkan 版自动更新用（内容同上，只是 `package.json` 的 `asset` 是老名字），不用下载 |
| `SHA256SUMS.txt` | — | 全部文件的 SHA-256 |

Windows 安装程序双击装到 `C:\Program Files\Friday`（建快捷方式，「设置 › 应用」里可卸载）；zip 便携版解压后运行
`Friday\friday.exe`（两者都自带 VC++ 运行库，未做代码签名）。Linux 的 deb 用 `sudo apt install ./Friday-…deb` 装到 `/opt/friday`；
AppImage `chmod +x` 后直接运行，tar.gz 解压后运行 `Friday/AppRun`。包的目录布局与更新方式见源码仓库的 `qt/docs/PACKAGING.md`。

## 版本与渠道

- **正式版**：源码仓库 `main` 分支的每次提交触发，Release 标签 `vX.Y.Z`，标题「Friday 1.0.3 正式版」。
- **预发布（Beta）**：其他分支的每次提交触发，Release 标签 `vX.Y.Z-beta`，标记为预发布，标题「Friday 1.1.2 Beta（v1.1）」。
- **版本号 X.Y.Z**：分支名是 `vX.Y`（如 `v1.1`）就用它做前缀，其余分支取工程 `MARKETING_VERSION` 的前两位、已合并进来的 `vX.Y` 分支、已发布正式版的最大 X.Y 三者中最大的；第三位从 0 开始，每次打包自动 +1（正式版和预发布共用同一个序列，按本仓库已有标签计数）。
- 构建号是工作流的运行序号；版本、构建号、Release 标签、渠道一起写进 macOS app 的 Info.plist（`CFBundleShortVersionString`、`CFBundleVersion`、`FridayReleaseTag`、`FridayReleaseChannel`），以及 Qt 版的 `BuildInfo.h`（`FRIDAY_VERSION`、`FRIDAY_BUILD_NUMBER`、`FRIDAY_RELEASE_TAG`、`FRIDAY_RELEASE_CHANNEL`；v2.0 之前还有 `FRIDAY_GPU_BACKEND`）。
- 正式版和预发布各只保留最新的一个 Release，旧的连安装包一起删除；标签保留。

## 工作流

[`.github/workflows/build.yml`](.github/workflows/build.yml) 由源码仓库每次推送通过 `repository_dispatch` 触发（也可以在 Actions 页面手动运行，可选择是否同时构建 Qt 版），各次运行并行（版本号靠推送标签原子预留）：

1. **prepare**：解析源码提交，算版本号、渠道和 Release 标签，看源码还带不带本地推理（有没有 `qt/ci/build_backends.sh`）、macOS 版还用不用 MLX；
2. **macos**：Xcode 构建（还用 MLX 的老源码先装 Metal 编译器）、Developer ID 签名，dmg 也签名，公证 dmg 后把票据钉到 dmg 和 app 上、用钉过的 app 重打 zip，并用 Gatekeeper（`spctl`）确认是「Notarized Developer ID」，产物上传为 artifact；缺证书或公证密钥、公证不通过都直接失败，不发布未公证的包；
3. **qt**：`windows-2022`、`ubuntu-22.04` 并行（v2.0 之前的源码再乘 `{cuda, vulkan}`，共四个组合），互不影响（`fail-fast: false`）。调用源码 `qt/ci/` 下的脚本：构建应用并跑单元测试 → 打包（Windows：zip + 安装程序，安装程序在运行器上
静默装一遍、卸一遍核对；Linux：AppImage + tar.gz + deb，deb 用 apt 装一遍核对；源码还没有 `qt/installer/` 的老分支只出 zip / AppImage / tar.gz）。Qt 等工具链版本都钉在源码的 `qt/ci/deps.env` 里，另有 ccache；
老源码另在构建应用前取依赖、构建推理后端（按组件分别缓存，CUDA 编译很慢，依赖和工具链不变就不重编）；
4. **release**：macOS 成功才发布；某个 Qt 组合失败时照样发布其余产物，并在 Release 说明里注明缺哪些。v2.0 起先给 1.x 的 CUDA / Vulkan 版各做一份只改了 `package.json` 的便携 zip / tar.gz（`-upgrade-<后端>`）。一次性上传全部文件与重新计算的 `SHA256SUMS.txt`，然后删掉同渠道的旧 Release、更新 `latest.json`。

## 自动更新

每次发布后工作流会更新根目录的 [`latest.json`](latest.json)：每个渠道（`stable` / `beta`）最新的一次发布，包含标签、版本、构建号、zip / dmg 地址、zip 的 SHA-256、大小、最低系统版本和提交说明，以及各平台安装包的 `assets`：

```json
"assets": {
  "windows-x64":        { "name": "…zip", "url": "…", "sha256": "…", "size": 0, "installer": { "name": "…-setup.exe", "url": "…", "sha256": "…", "size": 0 } },
  "linux-x64":          { "name": "…AppImage", "url": "…", "sha256": "…", "size": 0, "tarball": { "name": "…tar.gz", "…": "…" }, "deb": { "name": "…deb", "…": "…" } },
  "windows-x64-cuda":   { "name": "…-upgrade-cuda.zip", "…": "…", "installer": { "name": "…-setup.exe", "…": "…" } },
  "windows-x64-vulkan": { "…": "…" },
  "linux-x64-cuda":     { "name": "…AppImage", "…": "…", "tarball": { "name": "…-upgrade-cuda.tar.gz", "…": "…" }, "deb": { "…": "…" } },
  "linux-x64-vulkan":   { "…": "…" }
}
```

macOS 版 Friday 每小时读一次这个文件，比自己新就下载 zip、校验 SHA-256 和 Developer ID 签名，Agent 空闲时自动换包重启，正在工作时在侧边栏底部提示「点击更新到 …」；设置 › 通用 里可以关掉自动更新或只接收正式版。Windows / Linux 版按「系统-架构」（如 `windows-x64`；1.x 是「系统-架构-后端」，如 `windows-x64-cuda`）取 `assets` 里自己的包、再按安装方式取文件（便携版 zip / 安装程序 /
AppImage / deb / tar.gz），校验 SHA-256 后换包（安装程序和 deb 装的要管理员确认，只在用户点了时装）；这次构建缺某个平台时对应的键不存在，那个平台就保持原版本。
v2.0 起 1.x 的老键（`-cuda` / `-vulkan`）指向同一套包，便携 zip / tar.gz 换成 `package.json` 对得上的 `-upgrade-<后端>`：1.x 解开包后会核对它和自己的名字一致。

## 签名与公证

macOS 包的签名和公证依赖本仓库的 Actions 配置（Settings › Secrets and variables › Actions）：

| 名称 | 类型 | 说明 |
| --- | --- | --- |
| `MACOS_CERTIFICATE_P12` | secret | Developer ID Application 证书（含私钥）导出的 .p12，base64 编码 |
| `MACOS_CERTIFICATE_PASSWORD` | secret | .p12 的密码 |
| `KEYCHAIN_PASSWORD` | secret | 运行器上临时钥匙串的密码（随便设一个） |
| `CODE_SIGN_IDENTITY` | variable | 签名身份，如 `Developer ID Application: yuehong sun (6AXTRT5TV4)` |
| `DEVELOPMENT_TEAM` | variable | 团队 ID |
| `NOTARY_KEY_ID` | secret | App Store Connect API 密钥的 Key ID（`AuthKey_<Key ID>.p8` 文件名里那段） |
| `NOTARY_ISSUER_ID` | secret | App Store Connect API 的 Issuer ID |
| `NOTARY_KEY_P8` | secret | `AuthKey_<Key ID>.p8` 文件的原文 |

本机验证下载的包：`spctl -a -vvv -t exec Friday.app` 应输出 `source=Notarized Developer ID`，
`xcrun stapler validate Friday-<标签>-macOS-arm64.dmg` 应输出 `The validate action worked!`。
