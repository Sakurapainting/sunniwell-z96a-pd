# Z96A 开发说明

## 源码与构建

2026-11-24 的 BSP 打包变更对应 `PKGBUILDs` 提交 `9b44cb2`。对照历史实验时使用固定版本。 [BSP 包源码和安装说明](https://github.com/Sakurapainting/PKGBUILDs/tree/9b44cb26579194c644aa7ca959c3549717f969c5/core/linux-rockchip-z96a)

| 组件 | 源码 | CI |
| --- | --- | --- |
| Rockchip BSP 6.1 | [内核包、DTS 和补丁](https://github.com/Sakurapainting/PKGBUILDs/tree/z96a-niri/core/linux-rockchip-z96a) | [z96a-bsp-kernel.yml](https://github.com/Sakurapainting/PKGBUILDs/actions/workflows/z96a-bsp-kernel.yml) |
| Mainline 内核与板级包 | [内核包](https://github.com/Sakurapainting/PKGBUILDs/tree/z96a-niri/core/linux-aarch64) · [板级配置](https://github.com/Sakurapainting/PKGBUILDs/tree/z96a-niri/alarm/z96a-typec-post-install) | [z96a-build.yml](https://github.com/Sakurapainting/PKGBUILDs/actions/workflows/z96a-build.yml) |
| Niri 桌面配置 | [桌面配置与打包说明](https://github.com/Sakurapainting/PKGBUILDs/tree/z96a-niri/alarm/z96a-niri-profile) | [z96a-niri.yml](https://github.com/Sakurapainting/PKGBUILDs/actions/workflows/z96a-niri.yml) |

BSP CI 检查包内文件路径、内核配置和 DTB，并在临时 Arch Linux ARM 根文件系统中验证安装及 `mkinitcpio`。在对应的 Actions 运行页取包时，核对构建提交、包版本和验证结果。 [2026-11-24 Actions 启用截图](assets/github-actions-enabled.jpg)

## BSP 上板验证

1. 记录 `/boot` 的挂载位置和剩余空间。备份并校验 mainline 的 Image、initramfs、DTB 和 extlinux 配置。
2. 通过 UART 验证能进入 U-Boot 并手动启动原有的 `linux` 条目。接线前单独确认设备 RX 电平和串口适配器 TX 电平。
3. 按 BSP 包说明，把同一次构建的 Image、DTB、initramfs 部署到独立路径。保留 extlinux 默认条目 `linux`，手动选择 `linux-bsp61` 测试。
4. 记录 `uname -r`、串口登录结果、`/sys/bus/sdio/devices` 和实体 HDMI 输出。需要回退时从 U-Boot 选择 `linux`。 [完整操作步骤与回退说明](https://github.com/Sakurapainting/PKGBUILDs/blob/9b44cb26579194c644aa7ca959c3549717f969c5/core/linux-rockchip-z96a/README.md#recovery-gate)

[2026-11-24 实验记录](../logs/2026-11-24-bsp61.md)
