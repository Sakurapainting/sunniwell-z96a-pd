# Z96A 开发说明

## 源码与构建

2026-11-24 的 BSP 打包变更对应 `PKGBUILDs` 提交 `9b44cb2`。对照历史实验时使用固定版本。 [BSP 包源码和安装说明](https://github.com/Sakurapainting/PKGBUILDs/tree/9b44cb26579194c644aa7ca959c3549717f969c5/core/linux-rockchip-z96a)

| 组件 | 源码 | CI |
| --- | --- | --- |
| Rockchip BSP 6.1 | [内核包、DTS 和补丁](https://github.com/Sakurapainting/PKGBUILDs/tree/z96a-niri/core/linux-rockchip-z96a) | [z96a-bsp-kernel.yml](https://github.com/Sakurapainting/PKGBUILDs/actions/workflows/z96a-bsp-kernel.yml) |
| Mainline 内核与板级包 | [内核包](https://github.com/Sakurapainting/PKGBUILDs/tree/z96a-niri/core/linux-aarch64) · [板级配置](https://github.com/Sakurapainting/PKGBUILDs/tree/z96a-niri/alarm/z96a-typec-post-install) | [z96a-build.yml](https://github.com/Sakurapainting/PKGBUILDs/actions/workflows/z96a-build.yml) |
| Niri 桌面配置 | [桌面配置与打包说明](https://github.com/Sakurapainting/PKGBUILDs/tree/z96a-niri/alarm/z96a-niri-profile) | [z96a-niri.yml](https://github.com/Sakurapainting/PKGBUILDs/actions/workflows/z96a-niri.yml) |

BSP CI 检查包内文件路径、内核配置和 DTB，并在临时 Arch Linux ARM 根文件系统中验证安装及 `mkinitcpio`。在对应的 Actions 运行页取包时，核对构建提交、包版本和验证结果。 [2026-11-24 Actions 启用截图](assets/github-actions-enabled.jpg)

## 当前实机与烧录验收

截至 2026-11-27，实机从 eMMC 默认启动 `6.1.172-10-z96a-bsp61`，保留 `movablecore=256M` 和禁用 NPU 的 `linux-bsp61-recovery` 恢复项。Actions `45936151382` 的 Niri LBA0 镜像通过整盘 SHA-256 回读、首次启动、按键冷启动、内屏 Niri 和 RKNN FP16 推理验收。`-9` 的 NPU 电源域启动故障及 `-10` 的修复和测试数据见 [11 月 27 日实机记录](../logs/2026-11-27-bsp-release.md)。

`6.1.172-10` 的内核包和对应完整源码见 [发布说明](releases/z96a-bsp-v6.1.172-10.md)。11 月 26 日的 `-8` 镜像验收、Smart0 试启和 SDDM 用户列表处理见 [11 月 26 日实机记录](../logs/2026-11-26-bsp-release.md)。

覆盖 BOOT/ROOT 前先制作并校验当前系统的回退备份。`backup-current-20261126` 是当时的 `-8` 快照，覆盖 p9 BOOT、p10 ROOT 和 eMMC 头尾各 128 个扇区。盘尾 GPT 与当时 Linux 使用的主 GPT 不一致，恢复分区时以核验过的主 GPT 和实际分区位置为准。可将备份另存到独立物理盘，防止主机磁盘故障。

### 制作独立备份副本

本机已验收的快照在 `local/bsp-release/backup-current-20261126/`。在独立磁盘上选择可写的私有目录（例如挂载好的 Linux 文件系统，建议预留至少 32 GiB 可用空间）。用 `findmnt -T` 与 `lsblk` 核对目标确实在另一块物理盘；同一盘上的另一个目录、分区或子卷不算独立副本。以下命令只复制文件，不操作板端或块设备。先设好 `BACKUP_DEST`，在项目根目录运行：

```bash
set -euo pipefail
umask 077
src="$PWD/local/bsp-release/backup-current-20261126"
: "${BACKUP_DEST:?请先指定独立磁盘上已挂载的私有目录}"
test -s "$src/backup-verified"
test -d "$BACKUP_DEST" && test -w "$BACKUP_DEST"
test "$(findmnt -n -o MAJ:MIN -T "$src")" != "$(findmnt -n -o MAJ:MIN -T "$BACKUP_DEST")"
dest="$BACKUP_DEST/z96a-bsp-niri-backup-20261126"
test ! -e "$dest" && test ! -L "$dest"
stage=$(mktemp -d --tmpdir="$BACKUP_DEST" .z96a-bsp-niri-backup.XXXXXXXX)
cp --sparse=always -- "$src"/{BOOT.img,ROOT.img,gpt.bin,gpt-tail.bin,SHA256SUMS,README.md,backup-verified,source.state,BOOT.fsck.log,ROOT.fsck.log,gpt-check.log} "$stage/"
sync -f "$stage"
(cd "$stage" && sha256sum --strict -c SHA256SUMS)
mv -nT -- "$stage" "$dest"
test ! -e "$stage"
sync -f "$BACKUP_DEST"
```

备份可能含有用户数据和凭据，应保存在私有目录。板上测试使用过两种裸镜像。BOOT/ROOT 版从 `0x2E800` 写入，整盘版从 LBA 0 写入。烧录前要核对镜像对应的起点和覆盖范围。

## 历史流程：保留主线默认时试启 BSP

1. 记录 `/boot` 的挂载位置和剩余空间。备份并校验 mainline 的 Image、initramfs、DTB 和 extlinux 配置。
2. 通过 UART 验证能进入 U-Boot 并手动启动原有的 `linux` 条目。接线前单独确认设备 RX 电平和串口适配器 TX 电平。
3. 按 BSP 包说明，把同一次构建的 Image、DTB、initramfs 部署到独立路径。保留 extlinux 默认条目 `linux`，手动选择 `linux-bsp61` 测试。
4. 记录 `uname -r`、串口登录结果、`/sys/bus/sdio/devices` 和实体 HDMI 输出。需要回退时从 U-Boot 选择 `linux`。 [完整操作步骤与回退说明](https://github.com/Sakurapainting/PKGBUILDs/blob/9b44cb26579194c644aa7ca959c3549717f969c5/core/linux-rockchip-z96a/README.md#recovery-gate)

[2026-11-24 实验记录](../logs/2026-11-24-bsp61.md)
