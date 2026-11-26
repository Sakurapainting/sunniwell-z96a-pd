# Z96A BSP 常规启动和烧录产物 · 2026-11-25

## 实机启动配置

板端 `/boot` 为 `/dev/mmcblk0p9`，根分区为 `/dev/mmcblk0p10`。原本普通启动选择 `linux`，BSP 槽里仍是旧文件。先将已成功运行 RKNN 的 `6.1.172-7-z96a-bsp61` Image、initramfs 和 NPU 试验 DTB 写入三个现有 BSP 文件路径，设置 `default linux-bsp61` 和 `timeout 50`。旧 BSP 文件及原 extlinux 配置备份于板端 `/var/lib/linux-rockchip-z96a/boot-backup.HPpMRICk`。主线 Image、initramfs 和 DTB 与 `-7` 上轮验收清单一致。过程记录在 `local/bsp-release/boot-default-7.log`。

## 打包流水线

`Sakurapainting/PKGBUILDs` 的 `z96a-niri` 分支将实测 NPU 配置纳入 `6.1.172-8`，默认启用核心和 IOMMU，供电接 RK809 `DCDC_REG4`，上限 600 MHz，单独的 `bus-npu` 调压节点禁用。原始 Rockchip BSP 固定在 `77168c8d5ab82399f65a80e9f807b50ba37cf483`。完整源码补丁包含板级 DTS、内核配置和 rtw88 v6.4 导入及其 SDIO 修正。补丁包同时收录打包源码中的四个编号补丁和构建校验文件。

Actions `45571004938` 已完成源码准备，导出阶段因 makepkg 使用 `/work` 下的 Git alternate 对象库而中止。导出迁到同一 Arch Linux ARM chroot 后，由最终提交 `c727849ceb6307e4c00c772c232a8ebe9f233ce5` 触发完整构建 `45572211711`。

## Rockchip 原始镜像布局

用当前板的实际 GPT 校验布局。`BOOT` 位于 `0x2E800`，524288 个扇区。`ROOT` 位于 `0xAE800`，实际 60356575 个扇区。镜像写入 `0x2E800` 至 `0x7AE7FF`（含），每扇区 512 字节，共 4026531840 字节。镜像中的 ext4 首启按 GPT 分区大小扩容。前置固件、GPT 和 Android 分区在写入区间之外。发行镜像从签名的 Arch Linux ARM 基础系统构建，首次在控制台设置 root 密码，桌面和 RKNN Runtime 单独安装。

附带的 `check-flash-layout.py` 已用本机 eMMC 只读导出的 GPT 验证。脚本也能拒绝根分区过小与 GPT 分区表 CRC 错误。FAT 文件树已独立演练过写入、`fsck.fat` 与回读比对。当前实机走现有系统的无损内核升级路径，完整镜像的 USB 烧录另行验证。

## Actions 验收结果

| 产物 | Actions 运行 | SHA-256 |
| --- | --- | --- |
| `linux-rockchip-z96a-6.1.172-8-aarch64.pkg.tar.xz` | `45572211711` 的 `build` | `4d7500fc2ceb6d693532475afe4329413a8a029984fce242903191e9e1e42ad7` |
| `z96a-bsp-patches.tar.xz` | `45572211711` 的 `build` | `7c01e60c17521c10011d6a95f5cc5905dbcafe09e769d30919c9f4679741c40d` |
| `z96a-archarm-6.1.172-8-z96a-bsp61-boot-root.img.zst` | `45576665803` 的 `flash-image` | `4480ef435e6a1d946e1398f7fd2276544301dfeeb061fe4bf99e4bfee2d25fe8` |
| 解压后的 `.img` | 同上 | `c2693f869a25576bfdb05db866a7deb11f75aa8c89516c08a8900ac832c72d75` |

`45572211711` 的内核任务通过包内容、安装及 initramfs 检查，镜像任务因普通用户无法读取 `mkinitcpio` 写出的 root 专用文件而失败。提交 `36fa393ac23d36899f240f03c94d33eb8c23e079` 通过 `sudo install` 将原 initramfs 以受控所有者和 `0644` 模式复制到 BOOT 工作目录，并支持从指定的成功内核任务复用产物。镜像任务 `45576665803` 已完成，`layout.json` 记录原内核运行 ID `45572211711`。

本地核验三个压缩产物及解压后的镜像。补丁通过在 Rockchip 官方归档提取的原始受影响源码上执行 `git apply --check`。镜像的 BOOT 中回读的 Image 和 DTB 与已验收的内核包字节一致，extlinux 默认项为 `linux-bsp61`，ROOT 的 `e2fsck -fn` 通过。日志在 `local/bsp-release/45576665803/image-verification.log`。

## `6.1.172-8` 常规启动实机验收

16:26:41 执行正常重启。串口显示 U-Boot 读取 eMMC 的 `/extlinux/extlinux.conf`，自动选中 `linux-bsp61` 并加载三份 BSP 启动文件。Linux 启动后 boot ID 为 `4baa47d5-87ab-4958-9ae6-4eba0162cc7f`，Wi-Fi 自动通过 `wlan0` 连接，SDDM 与图形目标为 active，内屏 DRM eDP 为 connected/enabled，1920×1080。主线救援文件哈希与原值一致。替换前的 `-7` BSP 启动文件备份于板端 `/var/lib/linux-rockchip-z96a/boot-backup.5QgOzEU7`。旧 `-7` 模块目录也保留，供一次性试启使用。

RKNPU 0.9.8 和 `rk_iommu` 已绑定，NPU 使用 IOMMU group 0，出现 `/dev/rknpu`。本次热重启后，已知输出的 FP16 卷积无法完成。复验覆盖默认 heap、uncached heap、200/400/600 MHz 和一次驱动软件复位，均在约 6 秒报任务超时。NPU 共用 IRQ 的计数保持 0，超时记录一致为 `raw status 0x1000`、`require mask 0x300`、`task counter 0`。对照完整 DTB 后，两者节点相同，802 项属性差异均可由 phandle 的重编号解释。再次用原先成功的 `-7` 同一 Image、initramfs 和试验 DTB 一次性热启动，复验出现相同超时。两种热启动均有串口和 SSH 记录。需要断电冷启动进一步确认硬件状态。

## 同一 boot ID 恢复及 `-8` 正常重启复验

16:44 重新连接板子时，仍运行 16:34 启动的一次性 `6.1.172-7-z96a-bsp61`，boot ID 为 `895acfdd-aaac-404a-91b9-6b4c246370c4`。该会话于启动后 96 秒曾出现 NPU 超时，16:45 用相同的 RKNN Runtime 2.3.2、模型和默认 DMA heap 复测通过。FP16 卷积在 600 MHz 连续执行 15 秒，24,507 次均与 CPU 参考逐元素一致，共用 IRQ 从 0 增至 24,507，推理窗口无新增内核消息。在这两个测试之间，内核 journal 未记录系统休眠和恢复。此结果证实故障能在同一次开机中消失，触发和恢复条件待查。

16:46 执行 `systemctl reboot`。串口确认原厂 U-Boot 从 eMMC 的 `/extlinux/extlinux.conf` 选择 `linux-bsp61`，16:46:35 启动 `6.1.172-8-z96a-bsp61`，boot ID 为 `778baba2-7265-4bc1-bf3c-afb2725928a4`。Wi-Fi 自动连接，SDDM 与图形目标 active，eDP connected/enabled、模式 1920×1080。RKNPU 0.9.8、`rk_iommu` 和 IOMMU group 0 正常绑定。

| 常规 `-8`、默认 DMA heap、600 MHz | 持续时间 | 正确输出 | NPU 峰值负载 |
| --- | --- | --- | --- |
| FP16 卷积，启动后约 53 秒开始 | 15.000 秒 | 24,882 次，每次 14,400 个值，最大绝对偏差 0 | 8% |
| MobileNet V1，`dog_224x224.jpg` | 30.004 秒 | 4,665 次，首选类别 156，概率 0.935059 | 81% |
| ResNet18，`space_shuttle_224.jpg` | 30.007 秒 | 2,631 次，首选类别 812，softmax 概率 0.999680 | 94% |

两个图像模型各自测试前，RKNPU 的 `runtime_status` 均为 `suspended`。三项测试的 IRQ 增量与通过次数一致，内核推理窗口均无新增消息。板子保留在默认 `-8`。同一 `-8` 已有一次热重启后推理持续超时和一次热重启后推理通过的记录，仍待确认跨启动稳定性。原始记录在 `local/bsp-release/verify-current-7.log`、`boot-default-8.next.uart.log`、`verify-default-8.next.log`、`verify-default-8-vision.log` 和 `compare-npu-boot-markers.log`。

## 跨重启复现及 DMA 物理地址对照

正常重启两次，以串口和不同 boot ID 确认 U-Boot 均自动引导 `-8`。首条 FP16 推理分别于启动后 24.55 秒和 26.52 秒开始，均出现约 6 秒的任务超时。NPU 中断计数为 0，`raw status=0x1000`、`require mask=0x300`、`task counter=0`。第一次的 boot ID 是 `f9897c2e-a7a8-41ed-acee-7493468a26d6`，约 74.49 和 150.44 秒重复超时，240.18 秒在同一会话恢复。恢复后的 7,827 次卷积输出与 CPU 逐元素完全一致，IRQ 等量增加。

| `conv-check` 的 IOMMU 跟踪 | `map` 事件数 | 物理地址 ≥ 4 GiB | 结果 |
| --- | ---: | ---: | --- |
| 默认启动首条推理，boot ID `d1c1875c-e8de-4e44-9da0-de01c821b115` | 23 | 23 | 超时，0 IRQ |
| 同一会话 116 秒 | 26 | 26 | 超时，0 IRQ |
| 同一会话 134 秒 | 23 | 17 | 超时，0 IRQ |
| 前一次默认启动约 240 秒，恢复后 | 22 | 0 | 7,827 次通过 |
| 一次性 `mem=3500M` 启动首条推理 | 22 | 0 | 8,486 次通过 |
| 一次性 `movablecore=256M` 启动首条推理 | 21 | 0 | 8,354 次通过 |

故障和通过样本的 RKNPU 与 IOMMU runtime resume 都返回 0，SCMI 都完成设定 600 MHz，跟踪中没有 IOMMU page fault。全内存启动的高地址 RAM 为 `0x1f0000000–0x1ffffffff`（256 MiB）。两次单变量试启均复用与 `/boot` 字节一致的 Image、initramfs 和 DTB。`mem=3500M` 试启将 Linux 可用物理地址限制于 4 GiB 以下，NPU 再进入 runtime suspend 后复测 8,171 次通过。`movablecore=256M` 把该高地址银行划入 `ZONE_MOVABLE`，`MemTotal` 保持 `3984808 kB`，NPU 再进入 runtime suspend 后复测 7,778 次通过。此次仍用默认 DMA heap；MobileNet V1 4,644 次、ResNet18 2,603 次均通过 30 秒参考分类验收，IRQ 分别等量增加，峰值 NPU 负载分别为 81% 和 90%，测试内核日志无新增。

尝试将 RKNN 进程的 heap 切换到 `/dev/dma_heap/reserved` 时，BSP 的 `rknpu_mem_sync_ioctl` 触发内核空指针 Oops。随后已重启，所有通过结果均来自默认 heap。该测试用法已弃用。高地址对照表明规避分配可消除已复现的推理故障，具体硬件或驱动为何不能处理该段物理地址仍待查。

板端 `/boot/extlinux/extlinux.conf` 的 `linux-bsp61` 项现追加 `movablecore=256M`，保留为本机规避方案。此次仅改这一行，原配置备份为 `/var/lib/linux-rockchip-z96a/extlinux-before-movable.nc2PVTw2/extlinux.conf`。诊断记录位于 `local/bsp-release/reboot-rknpu-20261125-165442/` 和 `local/bsp-release/reboot-rknpu-20261125-170005/`。

`movablecore=256M` 加入默认项后，连续三次执行正常重启并通过。串口和 SSH 确认每次都自动引导 `6.1.172-8-z96a-bsp61`，boot ID 各不相同，`MemTotal` 均为 `3984808 kB`。

| 轮次 | 首次推理开始时间（开机后） | 5 秒首轮正确次数 | 运行时休眠后再次正确次数 |
| --- | ---: | ---: | ---: |
| 1 | 27.55 秒 | 7,781 | 7,763 |
| 2 | 26.39 秒 | 7,943 | 7,961 |
| 3 | 24.90 秒 | 7,821 | 8,111 |

六次共 47,380 次卷积，每次 14,400 个值，最大绝对偏差均为 0，IRQ 每轮增量等于模型通过次数，内核推理窗口均无新增消息。板端最后一次启动时间为 17:12:03，boot ID `5f40d673-9d99-47d8-a67e-2e8494b51e71`，SDDM 和图形目标 active，Wi-Fi 上线，eDP connected/enabled、模式 1920×1080。记录在 `local/bsp-release/reboot-rknpu-20261125-171017/`。这轮只覆盖正常热重启；已有 Actions 镜像和补丁包是此前的版本。

## 手动关机开机复验

上一轮系统日志在 17:17:01 记录 `systemd-poweroff.service` 结束。用户手动开机，SSH 确认系统于 17:19:58 启动，boot ID 为 `659c73b6-2cd1-4470-bc06-aa49feb0b01f`。内核为 `6.1.172-8-z96a-bsp61`，实际命令行含 `movablecore=256M`，`MemTotal` 为 `3984808 kB`。`/dev/rknpu` 存在，驱动绑定在 IOMMU group 0。SDDM 和图形目标 active，eDP connected/enabled，首选模式为 1920×1080。

开机后 64.89 秒执行第一轮，5 秒内 8,410 次通过。RKNPU 回到 runtime suspend 后，于 76.12 秒执行第二轮，5 秒内 8,108 次通过。每次核对 14,400 个 FP16 输出，最大绝对偏差均为 0，IRQ 增量分别为 8,410 和 8,108，两轮测试窗口内核日志均无新增。原始记录见 `local/bsp-release/manual-reboot-rknpu-20261125-172045.log`。

## 带 NPU 启动参数的 Actions 镜像

提交 `3e5c45984766ff91a9ded29ed9085e9d99eefd60` 在镜像的 `linux-bsp61` 启动项加入 `movablecore=256M`。Actions `45584762782` 复用 `45572211711` 的 BSP 内核包，`flash-image` 成功。产物位于 `local/bsp-release/45584762782/image/`。

回读 BOOT 确认默认项及实际启动参数，Image 和 DTB 与内核包逐字节一致。FAT 与 ext4 检查、压缩包与解压镜像 SHA-256 均通过。用实机 eMMC 只读导出的 GPT 检查，镜像覆盖 `0x2E800` 至 `0x7AE7FF`，范围符合 BOOT 和 ROOT 布局。压缩镜像 SHA-256 为 `fd475082a56d9e10cefd673831a77b259b017016ccfaa781cfae99fb5b7fabc8`。核验输出见 `local/bsp-release/45584762782/image-verification.log`。实际 USB 写入及从新镜像启动复验仍待进行。

## 预装诊断工具与 RKNN 离线自检

提交 `5abc73d8b6e4b7d77af775399b0fe31223fc3c0c` 的 Actions `45588703349` 复用 `45572211711` 的 `6.1.172-8` 内核包，构建产物位于 `local/bsp-release/45588703349/image/`。rootfs 新增 `fastfetch`、`iw`、`dosfstools`、`usbutils`，以及 `z96a-rknn-selftest`、RKNN Runtime 2.3.2 和 RK3568 FP16 卷积模型。

CI 在 ARM64 rootfs 运行了 `fastfetch --version` 和自检帮助，并验证自检程序的动态库依赖。下载产物后回读 BOOT 与 ROOT，检查分区布局、文件系统和镜像校验和。ROOT 中的 RKNN Runtime 与模型和实机验收文件逐字节一致。压缩镜像 SHA-256 为 `61c2407ecedf49642eafff5581c657be3f98ab998442b20f7973a1b2289e798d`。核验输出见 `local/bsp-release/45588703349/image-verification.log`。这版镜像的首次实机推理仍待烧录后执行。

## USB 烧录准备

串口中断 U-Boot 自动引导后执行 `rockusb 0 mmc 0`。U-Boot 报告 LUN 0 对应 eMMC 0，主机枚举 `2207:350a`，Rockchip 官方 `rkdeveloptool`（`304f073752fd25c854e1bcf05d8e7f925b1f4e14`）列出一台 Loader。解压后的镜像再次通过 SHA-256 校验。

`rfi` 在主机打开 `/dev/bus/usb/001/006` 时返回 `EACCES`。设备节点仅 root 可写，当前命令环境设置了 `NoNewPrivs`，无法用 sudo 提权。板子留在 Loader，等待该 USB 节点的读写权限。eMMC 尚未写入。记录在 `local/bsp-release/flash-45584762782/`。

用户授权该节点的读写权限后，`rfi` 识别出 61,071,360 扇区的 Samsung 存储。USB 读取的前 128 扇区与同板 Linux 下导出的文件逐字节一致，镜像布局检查通过。BOOT 完整读回 268,435,456 字节。17:37:45 主机记录 Genesys USB 2.0 集线器 `1-1` 和下游 Rockusb `1-1.1` 同时断开，ROOT 读回中止。已留存 2,391,867,392 字节的 ROOT 前缀。经 UART 重进 `rockusb 0 mmc 0` 后，同一 USB 路径仍报告描述符超时和地址设置失败。等待直连后的 Rockusb 枚举，先核对 GPT 和断点重叠扇区再续读。eMMC 尚未写入。

换到 `1-1.2` 后，通过 U-Boot CPU reset 和 `rockusb 0 mmc 0` 重新枚举 `/dev/bus/usb/001/017`。新节点授权后，GPT 与断点前 4 MiB 比对通过。17:48:01 主机再次记录 `1-1` 集线器与 Rockusb 同时断开，另已保留自 LBA 5,386,368 开始的 532,480,000 字节，待新连接验明首尾再合并。已通过串口退出 Rockusb，板子停在 U-Boot。FTDI 当前位于主机直连路径 `1-2`，可交换两根线的主机侧 USB 口。eMMC 尚未写入。

交换主机接口后，FTDI 位于 `1-1.2`，Rockusb 经 U-Boot CPU reset 后在主机直连口 `1-2` 枚举为 `/dev/bus/usb/001/027`。等待新节点授权。eMMC 尚未写入。

节点授权后，直连读取未出现掉线，但同一原厂 U-Boot Rockusb 在 LBA 65535 返回有效数据，从 LBA 65536 起的测试扇区只返回 `0xCC`。两份所谓的分区读回均无对应文件系统签名，之前的断点比对只是填充数据相同。已中止读取，将这些文件隔离在 `rejected-uboot-rockusb-dumps`。Rockchip 参考 U-Boot 的 `cmd/rockusb.c` 也实现了超过读取边界时以 `0xCC` 填充的行为。

通过瑞芯微工具 `rd 3` 软件切入 Maskrom，仍位于直连口 `1-2`，新节点为 `/dev/bus/usb/001/028`。从 Rockchip `rkbin` 固定提交 `3e288fe814e059dd06833495f845cab04ac20a5c` 构建了 RK3568 USB Loader `rk356x_loader_v1.26.114.bin`，SHA-256 `cd7d7c0a09c1dfbe4b4dcc5241da16f749a2745f270005b6169a17fa10a82d5c`。待授权后用 `db` 下载到 RAM，验证 FAT32 和 ext4 签名后再做完整备份。eMMC 尚未写入。

18:36，直连口的临时 udev 规则在重新枚举后生效。`rkdeveloptool db` 成功下载 RAM Loader，USB 由无字符串描述符的 ROM 设备重新枚举为 `RockChip USB-MSC`，`rkdeveloptool ld` 仍因 USB 版本字段把它列为 Maskrom。Libusb 成功打开节点并认领接口，但 31 字节批量 OUT 无应答。`rfi`、GPT/BOOT/ROOT 读取和 `rd` 都未返回有效响应。UART 重复输出 `powerOn 64581` 与 `...62659`。USB 总线复位尝试后主机报告描述符与地址设置错误，设备从 USB 总线移除。没有写 eMMC。

预装 RKNN 离线自检的 Actions `45588703349` 镜像已解压。压缩包与解压镜像的 SHA-256 均通过，解压镜像 SHA-256 为 `7bfa8a4cc214e81582db81d24feb9b526be09b6d93e958c3ae98c5036a6b56ff`，与实机 GPT 的布局检查通过。恢复 USB 后先要求实际读出 BOOT/ROOT 文件系统数据并通过完整备份，再写这一版镜像。

## 原系统只读备份

重启后原厂 U-Boot 和已安装的 `6.1.172-8-z96a-bsp61` 正常启动，boot ID `1e95b86b-f5b2-44ba-a0bd-713a3815f5cd`。经 SSH 采集 GPT，内容同此前的 128 扇区备份。卸载 BOOT 后经 SSH 读出全量 268,435,456 字节，并与板端独立计算的 SHA-256 `6907babd27692429a76362b9b58c749d77a934f7a74392c849928719b9e966c9` 比对。保留未修改原始副本。FAT 检查仅报告已有 dirty 标志，修复临时副本后 `fsck.fat -vn` 通过，字节比对只差偏移 65 的标志位。原系统的 BOOT 已重新挂载。

停掉尚在登录界面的 SDDM 并将 journald 切换至 `/run` 后，ROOT 成功重新挂载为 `ro`。板端 `dumpe2fs` 确认 ext4 clean，分区大小 30,902,566,400 字节。SSH 首块读取中 Wi-Fi 失联，未留下已校验的 ROOT 数据。经 UART 在运行的 Linux 上初始化 `fcc00000.usb` 的设备端控制器并绑定临时 configfs Mass Storage，序列号 `Z96A-ROOT-BACKUP`，唯一 LUN 为 `/dev/mmcblk0p10` 且 `ro=1`。板端同时开始独立计算 ROOT 的 SHA-256。待重新接直连 USB 线后读取并核验 ROOT。没有开始烧录。

更换烧录线后，直连口 `1-2` 在 Linux Mass Storage gadget 模式多次报告读描述符 `-71` 和 `-110`，主机始终未注册块设备。限制高速、EP0 64 字节及临时强制 USB2 peripheral 均未恢复枚举。此结果只覆盖运行中 Linux 的 gadget 模式。已在同一次启动内通过 UART 临时关闭 Wi-Fi 省电，重绑 `rtw_8822cs`，恢复 `172.16.31.31` 的 SSH。ROOT 保持只读，板端给出全分区 SHA-256 `32c418cf6ce243d305238d74c2f5fd4bf8550bc6e9ff96fcdd9e0525a38b21f3`。备份改走 SSH，逐块（64 MiB）验证板端和本机的 SHA-256，完成后再做整盘 SHA-256 和只读 ext4 检查。

20:23，ROOT 全部 461 个分块、本机整分区 SHA-256 与板端独立 SHA-256 一致，`e2fsck -fn` 通过，未修改文件系统。BOOT、ROOT 和前 128 LBA 原始备份及校验清单位于 `local/bsp-release/flash-45584762782/backup-before-flash/`。后经 UART 正常重启进入原厂 U-Boot，实机 BOOT/ROOT 前 8 扇区的 CRC32 与备份一致。直连接口 `1-2` 枚举为 `2207:350a`，确认 eMMC 61,071,360 扇区且 USB 读出的 GPT 与备份一致。20:23:58 开始从 `0x2E800` 写入 Actions `45588703349` 的 7,864,320 扇区。

## 本次暂停的位置

20:28:05，`rkdeveloptool wl` 报告写入成功，共 4,026,531,840 字节。原厂 Rockusb 在写入前后读出的前 128 LBA 逐字节一致。通过 UART 用新 BOOT 上的 Image 和 DTB 一次性启动 `6.1.172-8-z96a-bsp61`，内核将 `/dev/mmcblk0p10`（179:10）挂载为 `ro,norecovery`，`init=/usr/bin/bash`，未运行 systemd 或执行首启扩容。回读脚本识别串口提示符后，设备号守卫仍返回失败，未生成完整镜像的回读校验；USB 写入不要重复执行。

用户要求暂停并拆线。确认无烧录或后续验收进程后，在只读环境执行 `sync` 和 SysRq poweroff，UART 返回 `reboot: Power down`（`local/bsp-release/flash-45584762782/pause-poweroff-uart.log`）。明天接回电源和 UART，拦截 U-Boot 自动启动，以相同临时参数进入只读环境。先用 `/proc/self/mountinfo` 的设备号字段核对 ROOT，校验从 LBA `0x2E800` 开始的 4,026,531,840 字节 SHA-256 为 `7bfa8a4cc214e81582db81d24feb9b526be09b6d93e958c3ae98c5036a6b56ff`；通过后再放行默认 BSP 首启，运行镜像内的 `z96a-rknn-selftest`，检查推理结果、NPU 负载、IRQ 和内核报错。若已正常首启，ROOT 扩容会改变原始字节，不能将此后的原始整段哈希当作烧录前镜像的回读校验。现有系统的完整备份已验收并保留。

## 2026-11-26 烧录与 RKNPU 实机验收

- 只接 UART 和电源，拦截原厂 U-Boot。用新 BOOT 的 Image 和 DTB 进入一次性只读环境，ROOT 挂载为 `ro,norecovery`，首启与扩容尚未发生。设备号守卫改用 `/proc/self/mountinfo` 的第三列消除 `findmnt` 输出末尾的空白。实机读取从 LBA `0x2E800` 开始的 4,026,531,840 字节，SHA-256 `7bfa8a4cc214e81582db81d24feb9b526be09b6d93e958c3ae98c5036a6b56ff`，与 Actions `45588703349` 原始镜像一致。前 128 LBA 的 SHA-256 仍为 `a0d29abc580751d3bf7a2756e27e6062ec30960206cff25b12f424fa970459f0`。未重复烧录。记录在 `local/bsp-release/flash-45584762782/usb-write-verified`。
- 完成正常首启并通过 UART 登录。extlinux 默认项 `linux-bsp61` 启动 `6.1.172-8-z96a-bsp61`，实际内核命令行带 `movablecore=256M`；NPU-disabled BSP 恢复项保留。ROOT 扩容后 `findmnt` 报告 28.2 GiB，BOOT 依旧是 `/dev/mmcblk0p9`。boot ID `862d5743-ca38-4931-b8ec-0d481850c010`。
- `/dev/rknpu` 存在，`fde40000.npu` 绑定 RKNPU，IOMMU group 0。镜像自带 RKNN Runtime `2.3.2`、驱动 `0.9.8`。FP16 卷积自检分别跑 5、5、15 秒，成功 8,192、8,238、26,422 次，共 42,852 次，每次 14,400 个值与 CPU 参考逐项比对，三个窗口的最大绝对误差均为 `0.000000`。
- 15 秒测试中与 NPU 共用的 IRQ 33 从 16,430 增至 42,852，增量与推理次数相同。测试期间采到 NPU 负载 5%–10%，每次测试窗口没有新增内核消息，空闲后 `runtime_status=suspended`。启动探测仍有此前记录的按名称查找 `npu_irq` 失败，实际走按索引绑定的中断。测试日志为 `firstboot-acceptance-20261126.log`、`rknn-irq-acceptance-20261126.log`，位于 `local/bsp-release/flash-45584762782/`。
- 板子留在新系统运行。未重新验证桌面、eDP 画面或外围。本机首启生成的 root 密码保存在该阶段目录下的 `firstboot-root-password`，文件权限为 `0600`，不写入日志、不提交。
