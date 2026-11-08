# PassWall2 开发约定

## 仓库与目录

- 本地项目目录：`/root/.acode/openwrt-passwall2`，后续项目命令在此执行。
- fork（origin）：`https://github.com/ayers-ltd/openwrt-passwall2`。
- 上游（upstream）：`https://github.com/Openwrt-Passwall/openwrt-passwall2`，默认分支 `main`。
- 此仓库是 OpenWrt LuCI 软件包 feed，不是 OpenWrt 完整固件源码；不要直接在仓库根目录运行固件编译。
- 主要代码在 `luci-app-passwall2/`：`luasrc/` 为 LuCI 页面与 Lua 逻辑，`root/` 为路由器配置、服务及脚本，`po/` 为翻译，`Makefile` 为软件包定义。
- 任务脚本、日志、下载、分析与截图统一保存到 `/root/.acode/mywork/.tasks/<任务名>/`；不要在本仓库创建 `.task/`、`.tasks/` 或保存 SDK 临时目录。

## 修改与验证

- 优先保持上游代码风格和 OpenWrt 兼容性；运行时 shell 使用 BusyBox ash/POSIX 兼容语法，不能假定路由器有 Bash、GNU 工具或 systemd。
- 修改 shell、Lua、UCI 配置、翻译和软件包依赖时，检查对应调用方；不要对未修改文件批量格式化。
- 提交前执行 `git diff --check`，并对改动文件执行适用的语法检查。GitHub Actions 编译是主要打包验证方式，不在本机安装完整交叉编译环境。
- 提交和推送前检查 `git status`、当前分支、远程目标及差异，仅包含本次任务文件；不要向 upstream 推送。

## GitHub Actions 测试编译

- 默认使用 `.github/workflows/test-build.yml`（名称 `Test build`）。相关源码或此 workflow 的 main 分支推送、PR 会自动触发，也支持手动触发。
- 流程使用 Ubuntu 24.04 与官方 OpenWrt 24.10.4 x86_64 SDK，通过固定 SHA256 校验 SDK，并将 checkout 的代码复制到 SDK 内构建，确保验证实际提交或 PR 合并结果。
- 此流程只测试 LuCI 软件包及中文语言包的 IPK 打包；不构建完整固件、不保证代理核心已编译，不覆盖 APK、其他架构或路由器运行效果。
- 产物在 Actions artifacts 中保留 7 天，包含 IPK、构建日志、最终 `.config`、源码提交和 feed 版本。失败时优先查看失败 step、`build.log` 与配置；环境初始化失败时可能没有完整产物。
- SDK 固定版本，但附加 PassWall 依赖 feed 跟踪 main，存在上游依赖变化；排查时记录 artifact 中的 feed 提交。
- SDK 使用分段下载和按 SHA256 区分的 Actions 缓存，缓存命中后仍执行校验；升级 SDK 时同步更新 URL、文件名和 SHA256。
- 当前 geoview 要求 Go >= 1.25，而 24.10 feed 的 Go 版本不足；测试流程从 OpenWrt packages 的固定提交复制 `lang/golang` 并重建 feed 索引。升级时核对 `GOLANG_FEED_COMMIT`，版本记录在 `build-info.txt`。

常用命令（在项目目录执行）：

```bash
gh workflow run test-build.yml --repo ayers-ltd/openwrt-passwall2 --ref <已推送分支>
gh run list --repo ayers-ltd/openwrt-passwall2 --workflow test-build.yml --limit 5
gh run view <run-id> --repo ayers-ltd/openwrt-passwall2
gh run view <run-id> --repo ayers-ltd/openwrt-passwall2 --log-failed
gh run download <run-id> --repo ayers-ltd/openwrt-passwall2 --dir /root/.acode/mywork/.tasks/<任务名>/artifacts
```

- 触发后核对 run 的分支与提交，跟踪至结束；只有目标提交的构建 job 成功并有预期软件包，才可报告“编译通过”。排队、执行中、取消或只通过静态检查均不可称为编译通过。
- 上游的 `Auto compile with openwrt sdk.yml` 是发布流程，会创建/更新 Release、编译多架构依赖，并通过 `PACKAGES_DISPATCH_TOKEN` 触发外部仓库；普通测试使用 `Test build`。仅在用户要求发布时运行发布流程。
- CI 打包成功不能代替真实路由器上的 DNS、透明代理、防火墙与节点连接测试；汇报时明确已验证范围。
