# 2026-11-27 · Z96A BSP 烧录与实机验收

`6.1.172-9` 延续前一天的 Smart0 和 Niri 配置修复。 [11 月 26 日记录](2026-11-26-bsp-release.md)

## `6.1.172-9` 构建验收

- `Sakurapainting/PKGBUILDs` 的 `z96a-bsp-niri-image` 分支提交 `5e89e8c09bb7792b6a0f825ce5e475c7562cbd46` 自动触发 Actions `45810214611`；`build`、`flash-image` 均通过。`6.1.172-9` 已把平面分配写入板级 DTS，包和最终镜像均检查 DTB，ROOT 检查桌面模板和 `alarm` 的 Niri 配置。已上传 BSP 包、BSP 补丁包、`z96a-rockchip-flash-image-niri`（BOOT/ROOT）和 `z96a-rockchip-flash-image-niri-lba0`（从 LBA0 写入，两版镜像均含 `.img.xz`）。

## `6.1.172-9` LBA0 烧录、启动故障与恢复

- 现场报告短按开机无明显反应。起初主机只识别 FTDI，UART 无输出、USB 无 Rockchip。随后 UART 收到 DDR、BL31、U-Boot 和 extlinux 输出；截获 U-Boot 提示符后 `mmc info` 和 `mmc part` 确认 eMMC 可读，BOOT/ROOT 起始 LBA 分别为 190464、714752。暂不判定开机键本身故障。
- 已下载 Actions `45810214611` 的 Niri LBA0 产物到 `local/artifacts/z96a-archarm-bsp61-niri-lba0-actions-45810214611/`。压缩包校验通过，解压 31,268,536,320 字节，SHA-256 `222023a0578ed5bdd794a5e93495ee6ca24739d3ad784ef64de7e9de52dc0d72`；GPT 主备表、固件前缀及 BOOT 的 extlinux 与清单一致。
- UART 运行 `rockusb 0 mmc 0`，确认 LUN 0、eMMC 用户区 61,071,360 扇区；USB 直连口 `1-2` 枚举设备 `2207:350a`（一次枚举节点 `/dev/bus/usb/001/005`）。`rkdeveloptool ld` 可列出 Loader，当前没有 USB 节点写权限，`rfi` 尚无法打开。待主机授权后先回读目标再写；本轮未写 eMMC。串口和镜像校验记录在 `local/flash-stage/flash-45810214611/`。
- 主机临时 USB ACL 生效后，`rkdeveloptool rfi` 确认 61,071,360 扇区，USB 回读头 128 LBA 的主 GPT 及分区表 CRC 匹配。10:26:20（北京时间）执行 `rkdeveloptool wl 0x0`，实际输入为已校验的 `6.1.172-9` Actions Niri 裸盘镜像。写后验收计划：USB 比对前 32 MiB，随后由 `-9` 内核在 ROOT 只读且不回放日志的环境计算整盘 SHA-256，再放行默认首启。
- 11:09:38（北京时间），`wl 0x0` 返回成功、进度 100%。写后在 USB 回读比较前 32 MiB，包括 GPT、固件区以及 LBA 59392 后的一段，字节一致；返回 U-Boot 后核对 BOOT 内的 `Image`、`initramfs.img`、正式 DTB 大小和 CRC32 全部匹配。此后直接从持续运行约 47 分钟的 Rockusb/U-Boot 环境一次性引导正式 DTB（只读校验），在 `rk_iommu_driver_init` 触发 NPU 电源域退出 idle 时 `failed to get ack on domain 'npu'`，进而 `panic_on_set_idle`。这是重启前的直接引导，尚未验证真正的复位或断电后默认启动。未进入 ROOT、未放行首启扩容；整盘 SHA-256 仍待用禁用 NPU 的恢复 DTB 只读验证。单独故障记录：`local/flash-stage/flash-45810214611/panic-after-rockusb-uart.log`。
- 11:14（北京时间），用户长按再短按后串口捕获到 DDR、U-Boot 及 eMMC 默认 extlinux 的完整启动前段；默认正式 DTB 的 `-9` 同样在 `rk_iommu_driver_init` 对 NPU 电源域退出 idle 等待 ACK 失败、随后 panic。故障不只发生在从 Rockusb 直接引导时；此现象解释了本次电源键操作后内屏无画面。默认失败串口：`local/flash-stage/flash-45810214611/panic-default-autoboot-uart.log`。对比镜像内正式 DTB 与前一轮成功的 Smart0 试启 DTB 的 1,017 个节点，节点名及所有属性值一致；不能将该 panic 归因于平面分配本身。现已提前挂起启动拦截，等下一次物理复位使用 NPU 禁用 DTB；尚未重刷或改写启动项。
- 用户再次长按后短按，UART 截获 U-Boot。通过 BOOT 中自带的 `z96a-bsp61-recovery.dtb` 启动同一套 `-9` 内核和 initramfs，进入 `ro,norecovery` 的 ROOT 维护 shell。整盘回读 SHA-256 为 `26a468e239f9465d830f0e48eef34f0ca2133799357ba735a7e995bbda833d00`，与原镜像不同。写后曾通过 USB 逐字节核对盘头 32 MiB。重启后，盘头变化定位到 `misc` 分区的 LBA 32772（110 个字节），其余盘头分段、BOOT、原始 ROOT 镜像范围之后的内容和备份 GPT 校验一致。ROOT 镜像范围的 SHA-256 也改变了。维护引导仍在挂载只读 ROOT 前运行了 initramfs 的文件系统检查，且已读到 ext4 超级块字段和校验和变化。后续使用 `e2fsck -fn` 检查 ROOT 通过（退出码 0）。这次回读未达到整盘逐字节验收。后续应在 USB 重启前完成整盘回读，或使用 `fsck.mode=skip` 控制只读验收引导中的检查，并单独记录固件对 `misc` 的改动。
- 通过 UART 从维护环境重启后拦截 U-Boot，逐一复核 `Image`、`initramfs.img`、恢复 DTB 的 CRC32，做了一次可写图形启动。首启 ROOT ext4 扩容完成，`6.1.172-9-z96a-bsp61` 的 SDDM 和 `graphical.target` 已启动，eDP 已启用并设置为 1920×1080，VP1 使用 Smart0。串口以预设 `alarm` 账户登录成功，确认 Niri 配置中 `Mod+Return` 和 `Mod+T` 都启动 Foot。本次恢复使用的设备树禁用了 NPU，eMMC 上的默认 extlinux 项仍会加载启用 NPU 的正式 DTB。此次恢复启动尚未现场确认屏幕和 Niri 快捷键。串口实录和命令回执见 `local/flash-stage/flash-45810214611/graphical-recovery-uart.log` 和 `recovery-live-probe.log`。
