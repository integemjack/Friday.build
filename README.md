# Friday.build

Friday 的构建产物与发布页。二进制由 GitHub Actions 从源码仓库自动构建、用 Developer ID 证书签名后发布到本仓库的 Releases。

## 版本与渠道

- **正式版**：源码仓库推送 `v<版本>` 标签（如 `v1.1`）触发，Release 标签同名，标题「Friday 1.1 正式版」。
- **Beta**：源码仓库 `main` 的每次提交触发，Release 标签 `v<版本>-beta.<构建号>`（如 `v1.0-beta.12`），标记为预发布，标题「Friday 1.0 Beta 12」。
- 版本号取自工程的 `MARKETING_VERSION`，构建号是工作流的运行序号；两者和 Release 标签、渠道一起写进 app 的 Info.plist（`CFBundleShortVersionString`、`CFBundleVersion`、`FridayReleaseTag`、`FridayReleaseChannel`）。

## 自动更新

每次发布后工作流会更新根目录的 [`latest.json`](latest.json)：每个渠道（`stable` / `beta`）最新的一次发布，包含标签、版本、构建号、zip / dmg 地址、zip 的 SHA-256、大小、最低系统版本和提交说明。Friday 应用每小时读一次这个文件，比自己新就下载 zip、校验 SHA-256 和 Developer ID 签名，Agent 空闲时自动换包重启，正在工作时在侧边栏底部提示「点击更新到 …」。设置 › 通用 里可以关掉自动更新或只接收正式版。
