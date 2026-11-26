# Sunniwell Z96A 开发资料

记录 Sunniwell Z96A（RK3568）的 Arch Linux ARM 适配、内核构建和板上调试。

- 源码、CI、启动验证和回退路径见[开发说明](docs/development.md)。
- VOP 时钟故障、RK809 时钟绑定与 SDIO 枚举见 [2026-11-24 BSP 6.1 调试记录](logs/2026-11-24-bsp61.md)。
- 根分区一次性启动、时钟自动绑定与 RTL8822CS Wi-Fi 验证见 [2026-11-25 BSP 上板验收](logs/2026-11-25-bsp61.md)。
- 本地 `PKGBUILDs/` 是打包仓库的独立检出。打包源码和 CI 位于 [Sakurapainting/PKGBUILDs 的 `z96a-niri` 分支](https://github.com/Sakurapainting/PKGBUILDs/tree/z96a-niri)。

## 添加记录

开发说明和可公开的参考资料放在 `docs/`。实验记录放在 `logs/`，按 `YYYY-MM-DD-主题.md` 命名，注明测试版本、变更、观察结果及待验证项。
保留能支撑结论的日志行，发布前去掉用户名、主机名、网络地址、设备标识和凭据。原始采集文件和本地安装包保留在工作目录，由 `.gitignore` 排除。
