# Friday.build

Friday 的构建产物与发布页。二进制由 GitHub Actions 从源码仓库自动构建、用 Developer ID 证书签名后发布到本仓库的 Releases。

## 版本与渠道

- **正式版**：源码仓库 `main` 分支的每次提交触发，Release 标签 `vX.Y.Z`，标题「Friday 1.0.3 正式版」。
- **预发布（Beta）**：其他分支的每次提交触发，Release 标签 `vX.Y.Z-beta`，标记为预发布，标题「Friday 1.1.2 Beta（v1.1）」。
- **版本号 X.Y.Z**：分支名是 `vX.Y`（如 `v1.1`）就用它做前缀，其余分支用工程 `MARKETING_VERSION` 的前两位；第三位从 0 开始，每次打包自动 +1（正式版和预发布共用同一个序列，按本仓库已有标签计数）。
- 构建号是工作流的运行序号；版本、构建号、Release 标签、渠道一起写进 app 的 Info.plist（`CFBundleShortVersionString`、`CFBundleVersion`、`FridayReleaseTag`、`FridayReleaseChannel`）。
- 正式版和预发布各只保留最新的一个 Release，旧的连安装包一起删除；标签保留。

## 自动更新

每次发布后工作流会更新根目录的 [`latest.json`](latest.json)：每个渠道（`stable` / `beta`）最新的一次发布，包含标签、版本、构建号、zip / dmg 地址、zip 的 SHA-256、大小、最低系统版本和提交说明。Friday 应用每小时读一次这个文件，比自己新就下载 zip、校验 SHA-256 和 Developer ID 签名，Agent 空闲时自动换包重启，正在工作时在侧边栏底部提示「点击更新到 …」。设置 › 通用 里可以关掉自动更新或只接收正式版。
