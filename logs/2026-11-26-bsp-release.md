# 2026-11-26 · Z96A 镜像验收与 Niri 内屏排查

接续 11 月 25 日完成的最小镜像烧录。 [前一日记录](2026-11-25-bsp-release.md)

## 烧录与 RKNPU 实机验收

- 只接 UART 和电源，拦截原厂 U-Boot。用新 BOOT 的 Image 和 DTB 进入一次性只读环境，ROOT 挂载为 `ro,norecovery`，首启与扩容尚未发生。设备号守卫改用 `/proc/self/mountinfo` 的第三列消除 `findmnt` 输出末尾的空白。实机读取从 LBA `0x2E800` 开始的 4,026,531,840 字节，SHA-256 `7bfa8a4cc214e81582db81d24feb9b526be09b6d93e958c3ae98c5036a6b56ff`，与 Actions `45588703349` 原始镜像一致。前 128 LBA 的 SHA-256 仍为 `a0d29abc580751d3bf7a2756e27e6062ec30960206cff25b12f424fa970459f0`。未重复烧录。记录在 `local/bsp-release/flash-45584762782/usb-write-verified`。
- 完成正常首启并通过 UART 登录。extlinux 默认项 `linux-bsp61` 启动 `6.1.172-8-z96a-bsp61`，实际内核命令行带 `movablecore=256M`；NPU-disabled BSP 恢复项保留。ROOT 扩容后 `findmnt` 报告 28.2 GiB，BOOT 依旧是 `/dev/mmcblk0p9`。
- `/dev/rknpu` 存在，`fde40000.npu` 绑定 RKNPU，IOMMU group 0。镜像自带 RKNN Runtime `2.3.2`、驱动 `0.9.8`。FP16 卷积自检分别跑 5、5、15 秒，成功 8,192、8,238、26,422 次，共 42,852 次，每次 14,400 个值与 CPU 参考逐项比对，三个窗口的最大绝对误差均为 `0.000000`。
- 15 秒测试中与 NPU 共用的 IRQ 33 从 16,430 增至 42,852，增量与推理次数相同。测试期间采到 NPU 负载 5%–10%，每次测试窗口没有新增内核消息，空闲后 `runtime_status=suspended`。启动探测仍有此前记录的按名称查找 `npu_irq` 失败，实际走按索引绑定的中断。测试日志为 `firstboot-acceptance-20261126.log`、`rknn-irq-acceptance-20261126.log`，位于 `local/bsp-release/flash-45584762782/`。
- 板子留在新系统运行。未重新验证桌面、eDP 画面或外围。

### 内屏黑屏初查

串口证实系统已启动。镜像未安装 SDDM、Niri 或 Mesa，`display-manager.service` 不存在；`graphical.target` 虽为 active，但没有桌面会话。`getty@tty1` 和串口 getty 均 active，未见 `/proc/fb`。Rockchip DRM 与 Panfrost 已登记，`card0-eDP-1` 当前为 `unknown` / `disabled` 且模式列表为空，背光 `actual_brightness=200`（最大 200）。这些信息确认无图形会话且 eDP 尚未启用，不能据此确定面板或链路是否另有故障。本次只读检查，未安装桌面或修改配置。

通过 UART 将从已验证的旧 ROOT 备份中读取的 `modetest` 和依赖临时放到新系统 `/run/z96a-edp-check/`，未安装软件包。主动 DRM 查询把 `eDP-1` 探测为 `connected`，读到 BOE EDID 和 1920×1080 的 60.02/39.27 Hz 两档。精确指定 60.00 Hz 没有匹配模式；改用 EDID 首选模式索引 `#0` 后，10:14 成功启动约三分钟的 SMPTE 彩条。DRM 报告 `eDP-1 connected/enabled`，VP1 `active=1`，以 XR24 扫描 1920×1080 @ 60.02 Hz。现场面板可见性与色彩等待用户确认。原始记录在 `local/bsp-release/flash-45584762782/edp-test-20261126/visual-preferred.log`。没有改启动配置、NPU 或安装桌面。

### 恢复 SDDM 和 Niri

用户确认实机 SMPTE 彩条画面和颜色正常。在已烧录的系统中使用镜像自带同步数据库的版本安装 SDDM `0.21.0-7`、Niri `26.04-1`、Mesa `26.3.1-3`、Xorg `21.1.26-1` 及桌面依赖，共 148 个新包，没有升级原有包。初始化官方 Arch Linux ARM 密钥环并保留签名校验；为兼容 BSP 不支持 Landlock，仅本次 pacman 命令传入 `--disable-sandbox-filesystem`，没有修改 pacman 配置。下载使用从已验证旧备份提取的单个 Wi-Fi 配置，实机仅放在 `/run/NetworkManager/system-connections/`，未持久化。

新建常规用户 `alarm`；精简后的 Niri 配置通过 `niri validate`，保留 `Super+Enter` 启动 Foot、`Super+D` 启动 Fuzzel，不覆盖显示模式和色彩。手动启动 SDDM 后，Xorg 的 greeter 在 `seat0` / `tty2` 成为前台会话，eDP-1 `connected/enabled`，VP1 以 XR24 扫描 1920×1080 @ 60.02 Hz。随后启用 `sddm.service` 开机启动，默认 `graphical.target` 不变。当前未重启，Niri 的实机登录、Panfrost 渲染和现场色彩仍待确认。

Image、initramfs、extlinux 和 BSP 主/恢复 DTB 的 SHA-256 全部与改动前一致；当前仍运行 `6.1.172-8-z96a-bsp61`，RKNPU 绑定 IOMMU group 0。保持 greeter 活动时运行 3 秒 FP16 自检：4,825 次通过，每次比对 14,400 个值，最大绝对误差为 `0.000000`。记录为 `local/bsp-release/flash-45584762782/desktop-start-acceptance-20261126.log`。本次未重烧，没有修改 Actions 镜像。

用户确认已登录 Niri。UART 复核 `seat0` 上 `alarm` 为前台 Wayland 会话，Niri 报告 BOE `eDP-1` 正使用首选 1920×1080 @ 60.024 Hz、缩放 1.25；Niri 进程持有 Rockchip DRM `card0` 以及 Panfrost `renderD129`（启动日志列出的 Rockchip `renderD128` 与实际打开的 Panfrost 节点分属不同驱动）。前台保留桌面时，3 秒 RKNN FP16 自检通过 4,801 次，每次比对 14,400 个值，最大绝对误差为 0，IRQ 33 同增 4,801，采样峰值负载 9%，测试窗口无新增内核消息。SDDM 已启用，BSP 启动文件校验全部通过。本次未重启；冷启动进入桌面仍待复验。记录为 `local/bsp-release/flash-45584762782/desktop-niri-acceptance-20261126.log`。

### 冷启动复验

串口 `systemctl poweroff` 正常结束，BOOT 和 ROOT 已卸载，内核输出 `reboot: Power down`。首次拔插电源仅短按未见屏幕或 UART 输出；用户报告拔充电线后充电口绿灯仍亮。之后依次拔线、长按、接回电源并短按，UART 被动采集到 SPL、原厂 U-Boot 的 `wait power key press down` 和 Linux 启动。尚不能确定第一次未启动的原因，也不将长按视为每次开机必需。

本次没有中断 U-Boot 或设置临时引导变量。boot ID 确认已更换。extlinux 仍默认 `linux-bsp61`，启动 `6.1.172-8-z96a-bsp61` 带 `movablecore=256M`，所有 BOOT 文件哈希与恢复桌面前一致。SDDM 自动启动，`seat0` 的 `tty2` 为前台 greeter，eDP-1 `connected/enabled`。此时 3 秒 RKNN FP16 自检通过 5,077 次，每次比对 14,400 个值、最大误差 0，IRQ 33 从 0 增至 5,077，自检窗口无新增内核日志。现场登录画面和本次 Niri 登录待用户确认。原始记录在 `local/bsp-release/flash-45584762782/` 的 `coldboot-desktop-poweroff-20261126.uart.log`、`coldboot-desktop-powerkey-retry-20261126.uart.log` 和 `coldboot-desktop-acceptance-20261126.log`。

用户确认冷启动后内屏表现正常。板端核对同一新 boot ID 中 `alarm` 为前台 Niri Wayland 会话，1920×1080 @ 60.024 Hz，进程持有 Panfrost `renderD129`。此时 3 秒 RKNN FP16 自检通过 4,977 次、最大误差 0，IRQ 从 5,077 增至 10,054，自检窗口无新增内核日志。记录为 `local/bsp-release/flash-45584762782/coldboot-desktop-session-20261126.log`。

### 独立 Niri 烧录镜像

`PKGBUILDs` 的 `codex/z96a-bsp-niri-image` 分支新增 `image_profile=niri`，保留 `minimal` 默认选项。桌面镜像采用实机验证过的最小 SDDM/Niri/Foot/Fuzzel 配置，不打包用户、密码或无线配置；首启设 root 密码后可自行创建桌面用户。Actions `45747812441` 复用 `45572211711` 已验收的 BSP 内核包，仍构建从 LBA `0x2E800` 起写入的 BOOT/ROOT 裸镜像。未重新烧录实机。

Actions `45747812441` 的 `flash-image` 成功。`z96a-rockchip-flash-image-niri`（artifact `15084471267`）已下载到 `local/bsp-release/45747812441/image/` 并解压。本地用烧录前备份的实机 GPT、FAT/ext4 检查和文件回读验收：区间 `0x2E800`–`0x7AE7FF`，4,026,531,840 字节；Image、主/恢复 DTB 和 extlinux 与已烧录的最小版逐字节一致。SDDM 开机链接、Niri 会话及配置存在，未预置 `alarm` 或 Wi-Fi 连接；镜像中 runtime 和模型与实机验收文件逐字节一致。压缩镜像 SHA-256 `efa85e84a88da3e1d468b4f77687bfb0c880de33e0237d10625b1e7717741eb1`，裸镜像 SHA-256 `e95cc9cbf39ef15eaa8641b4e30406a6228610ece73029a8b1c371fbd5f252d0`。记录为 `local/bsp-release/45747812441/image-verification.log`。独立桌面镜像尚未实机烧录，已通过冷启动验收的是当前板上系统。

## 已验收 BSP + Niri 同板恢复备份

用户确认只有这一块板，允许短暂停机。停止桌面、卸载 BOOT，将 clean 的 ROOT 重挂为只读，并将 eMMC 和两个分区设为块设备只读；使用仅在 `/run` 生效且固定主机密钥的临时 Wi-Fi/SSH 读取。完整镜像保存在 `local/bsp-release/backup-current-20261126/`，其中 BOOT 为 268,435,456 字节、SHA-256 `792ba5b5ee7558b0e86643faab8236c7fad620a175df84bdd8672c528f78945c`，ROOT 为 30,902,566,400 字节、SHA-256 `8c3ffd6f02acbf3bf51bef4cf8b89d57027107f3ee2800d7c285b5fdce5f0058`。ROOT 分块读取时只在板端 SHA-256 命中时复用本机已有字节；最后两张完整镜像均与板端独立计算的整分区 SHA-256 一致，`fsck.fat -vn` 和 `e2fsck -fn` 均通过。主机镜像为独立文件，不依赖旧备份。未烧写或修复 eMMC。

eMMC 前后各 128 个扇区均原样保存。主 GPT 与此前已验收的 128 扇区备份逐字节一致，p9/p10 起始 LBA `0x2E800`/`0xAE800` 与 Linux 一致；盘尾 GPT 的 CRC 独立有效，但 GUID 和分区布局与主 GPT 不一致，不能按盘尾 GPT 恢复。本次仅备份 BOOT/ROOT 和 GPT 区域，不是整盘原厂恢复镜像。

正常热重启后，实机 boot ID `5e4e8a06-a29d-49b5-8537-e354490cd15f`，仍使用默认 `6.1.172-8-z96a-bsp61`，SDDM active，eDP connected/enabled、1920×1080。临时网络和 SSH 文件已消失，`sshd.service` inactive；未改密码。RKNN Runtime 2.3.2 / 驱动 0.9.8，3 秒 4,845 次，每次比对 14,400 个值，最大绝对误差 0。

旧备份 `backup-before-flash/` 再次通过原 SHA-256 清单。含旧主线恢复入口及新备份不能替代的旧用户目录，原件保留。`zstd -1 -T2` 实测 BOOT/ROOT 共 12,837,459,859 字节（约 11.96 GiB），比原始约 29.03 GiB 少约 17.07 GiB。需要释放空间时，建议先另写压缩档并验证解压后逐镜像 SHA-256，再决定移除旧的裸镜像；本次未移除任何备份。

已生成旧系统独立无损归档 `local/bsp-release/backup-before-flash-zstd/`。BOOT/ROOT 的 `.zst` 共 12,837,459,859 字节（11.96 GiB）。从落盘的压缩档完整解压到校验管道，两个裸镜像的 SHA-256 与烧录前清单一致；盘头 GPT 和压缩档 SHA-256 均通过。归档含压缩与原始校验清单、只读验证脚本和恢复说明，未上传。两个旧裸镜像尚未删除，新备份和板端不受影响。因现在同时保有旧裸镜像和压缩档，主机剩余约 17 GiB；若后续只移除两份旧裸镜像，可释放约 29 GiB，比归档前净省约 17 GiB。

2026-11-26 12:18（北京时间），再次校验旧压缩档及完整解压后的原始 SHA-256，并重新校验当前备份的 BOOT、ROOT、主/副 GPT 区域。全部通过后，仅删除 `backup-before-flash/` 下旧的 `BOOT.img` 和 `ROOT.img`，保留压缩档、原校验记录和 GPT。旧目录的 `RAW-IMAGES-ARCHIVED.txt` 指向归档。`df` 可用空间实测从 18,116,644,864 增至 37,730,877,440 字节，实际释放约 18.3 GiB（低于两个裸文件的合计逻辑大小）。没有访问或修改板端。

## 清理已确认废弃的本地文件

只清理本机：逐字节确认 `rejected-uboot-rockusb-dumps/` 内 4 个失败回读全是 `0xCC` 后删除这 4 个大文件；保留 USB 日志、GPT 探针和失败证据摘要。删除刷机暂存目录中被替代的 Actions `45584762782` 裸镜像，其原 `.img.zst` 与解压后的 SHA-256 均复验通过。删除两套早期 Wi-Fi/Panfrost 构建的对象、静态库、命令缓存和临时 Kallsyms；源树、工具、内核配置、成品和实验记录保留。完整压缩并回读比对 279,849,552 字节的旧 UART 日志，替换原先截断的 `.gz`，去掉未压缩副本；文档内有逐字节相同副本的根目录截图也去重。`df` 实测可用空间从 37,729,599,488 增至 40,210,558,976 字节，释放约 2.31 GiB。清单与输出保存在 `local/cleanup-20261126/`；两套恢复备份、已烧并验收的 Actions `45588703349` 最小镜像、待验收的 `45747812441` Niri 镜像、BSP 补丁包和 RKNN 工具未改动。未访问板端。

## 独立 Niri 镜像 USB 烧录与首启

- 使用已验收的本机回退备份 `local/bsp-release/backup-current-20261126/`。刷前重新核对 BOOT、ROOT、GPT 头尾与 SHA-256 清单，UART 读出的 GPT 头尾也一致。没有另制副本，原备份保留。
- UART 进入原厂 U-Boot 的 `rockusb 0 mmc 0`（未用 Maskrom）。主机直连口 `1-2` 枚举 `2207:350a`，只有一台目标 Loader；核对 Samsung eMMC 为 61,071,360 个扇区，USB 读取的头 128 LBA 与备份逐字节一致。12:57:58–13:01:23（北京时间），`rkdeveloptool wl 0x2E800` 写入 Actions `45747812441` Niri 镜像的 7,864,320 个扇区（4,026,531,840 字节），只覆盖 BOOT / ROOT 范围，未写 GPT、U-Boot、trust 或原厂 Android 分区。
- USB 刷后 GPT 匹配。首次正常启动之前，用刚写入的 Image/DTB 一次性启动只读环境（`init=/usr/bin/bash`，ROOT `ro,norecovery`），对全部写入范围计算 SHA-256 `e95cc9cbf39ef15eaa8641b4e30406a6228610ece73029a8b1c371fbd5f252d0`，与 Actions 原始镜像一致；GPT 头尾亦匹配备份。记录 `local/bsp-release/flash-45747812441/usb-write-verified`。
- 随后放行正常首启。默认 extlinux 项 `linux-bsp61`、内核 `6.1.172-8-z96a-bsp61`、`movablecore=256M`，保留 `linux-bsp61-recovery`；ROOT 扩至 28.2 GiB。创建本地桌面账号 `alarm` 并沿用用户指定密码，未安装额外桌面包。SDDM 自动启动，`seat0` 前台为 SDDM，内屏 eDP `connected/enabled`、1920×1080，VOP VP1 约 60 Hz。新镜像的 Niri 登录及面板实际颜色待现场确认。
- 保持 SDDM 活动，镜像自带 RKNN Runtime 2.3.2 / RKNPU 0.9.8 实跑 5 秒 FP16 卷积：8,156 次，每次 14,400 个值与 CPU 参考逐项比对，最大绝对误差为 0。IRQ 33 增量 8,156，采样负载最高 10%，测试窗口无新增内核消息。首启与测试输出在 `local/bsp-release/flash-45747812441/firstboot.log`、`test.log`。
- 首启时实机 SDDM 在 `/etc/passwd` 新增 `alarm` 之前启动，现场报登录界面的用户列表为空。UART 确认 `alarm`（UID 1000、`/bin/bash`）有可用密码，且 SDDM 账号可经 NSS 查到该用户，没有隐藏用户配置。确认没有已登录的图形会话后仅重启 `sddm.service`（未重启整机、未改密码），前台 greeter 从 `c1` 切换到 `c5`，eDP 仍为 `connected/enabled`。记录 `local/bsp-release/flash-45747812441/sddm-user-refresh.log`；屏幕用户卡片仍待现场确认。
- 用户确认进入桌面。UART 验证 `alarm` 为 `seat0` 前台 Niri Wayland 会话；Niri 打开 Rockchip DRM `card0` 和 Panfrost `renderD129`，内屏 BOE `eDP-1` 运行首选 1920×1080 @ 60.024 Hz、缩放 1.25，VOP VP1 用 XR24 扫描。保留桌面前台时 RKNN 自检 5 秒通过 8,045 次，每次比对 14,400 个值，最大误差 0；IRQ 33 从 8,156 增至 16,201（增量 8,045），采样负载 9%–11%，窗口内无新增内核消息。记录 `local/bsp-release/flash-45747812441/niri-session-acceptance.log`。面板肉眼色彩和该镜像的断电冷启尚未复验。
- 用户现场确认新镜像登录 Niri 后内屏显示正常。至此 USB 烧录、默认 BSP 首启、Niri/Panfrost、内屏画面和桌面运行时 RKNN 推理均通过。本轮未再次重启，独立 Niri 镜像的断电冷启未做。
- 开始独立 Niri 镜像的关机再开机验收。用户保存工作并确认可以关机后，通过 UART 执行 `systemctl poweroff`；BOOT 与 ROOT 正常卸载，内核最后报告 `reboot: Power down`。关机前 boot ID `fe591a03-150b-4a03-9644-63bdcee447b4`。板子停在关机状态，等待重新接电开机。记录 `local/bsp-release/flash-45747812441/coldboot-poweroff.log`。
- 用户按长按再短按开机并报告已进入 Niri 桌面。串口随后恢复响应，确认此次断电冷启 boot ID `42a20699-8dc6-494e-88e6-3f90a9cadacc`，默认 `6.1.172-8-z96a-bsp61`（root=`PARTLABEL=ROOT`），SDDM / graphical.target active 且 SDDM enabled；`seat0` 为 `alarm` 的活动 Niri Wayland 会话，eDP `connected/enabled`、1920×1080。保持桌面运行做第二次冷启后 RKNN 自检：5 秒通过 8,766 次，每次 14,400 个输出最大误差为 0；IRQ 33 增量 8,766，采样负载 5%，窗口内无新增内核消息。冷启日志 `local/bsp-release/flash-45747812441/coldboot-login-probe.log`、推理日志 `coldboot-test.log`。独立 Niri 镜像的断电冷启动验收通过。

## LBA0 镜像登录 Niri 后内屏黑屏

- Actions `45780674791` 的 LBA0 镜像已完成前 32 MiB 回读比对，并从 eMMC 启动至 `6.1.172-8-z96a-bsp61`。用户看到 SDDM，但报告登录后黑屏，随后经 IPC 打开的 Foot 在实屏不可见。
- UART 确认 Niri 为活动会话，进程实际持有 Rockchip `card0` 与 Panfrost `renderD129`。内屏 eDP-1 首选 `1920x1080 @ 60.024 Hz`、缩放 1.25，背光驱动报告 200/200。通过 UART 回收并校验的 `local/flash-stage/sddm-black-screen-Niri-A.png` 清晰包含 Foot 与 Niri 退出确认提示，提示未确认，会话未退出。截图不是面板实拍。
- Niri IPC 对 eDP 关开后恢复 `connected/enabled`，VOP2 在关输出期间打印 `Smart1-win0 is invisible`，这条日志本身不能归因。之后仅临时切到 EDID 的 `1920x1080 @ 39.269 Hz`，内核确认 VOP VP1 使用 93.5 MHz；新开的黄色测试窗口在截图 `local/flash-stage/sddm-black-screen-Niri-B.png` 可见，等待现场判断内屏是否同步显示。没有改配置、改密码、重启或重刷。
- 另查到镜像先创建 `alarm`、后放入 `/etc/skel` 的 Niri 配置；板端 `/home/alarm` 的配置与 `/etc/skel` 哈希不同。尚无证据表明这是黑屏根因。
- 用户确认 39.269 Hz 实屏仍全黑。串口恢复 EDID 首选 60.024 Hz，并用 Niri IPC 临时载入 `/etc/skel` 中的已验收精简配置副本（`/run/user/1000/z96a-known.kdl`），未覆盖 `~/.config`。黄色窗口在 60 Hz 下已映射为全屏；本次用户未确认该窗口的实屏可见性。
- 对照同板上版正常实机的 `local/bsp-release/flash-45747812441/firstboot.uart.log` 与 `local/flash-stage/boot-lba0-uart.log`：U-Boot 版本相同，落盘 BSP DTB 逐字节相同（SHA-256 `0b256dc7fb92efe633ea3a17eda9521dc297129350a2964fddec0e3e7cce2855`）。原厂分区仍保留时，U-Boot 完成 eDP 链路训练，VPLL 701 MHz，Linux 的 eDP VP1 主平面是 Smart0；LBA0 产物启动时 U-Boot 无法加载原厂 resource，VPLL 初始 24 MHz，Linux VP1 默认选 Smart1。两项差异都可能影响显示，不足以单独归因。
- 派生一次性试启 DTB，语义 diff 只有 VOP VP0/VP1 的四个 `rockchip,plane-mask` / `rockchip,primary-plane` 属性，使平面分配与旧镜像一致。未改时钟、NPU、正式内核、DTB、extlinux 或分区。`/home/alarm/.z96a-edp-check/edp-plane-ab.dtb` 已经 UART 临时传入并核验 SHA-256 `ec6ba33ec98c4c709003018352c5987c353948c6d79f217596812467be61f2f7`，尚未试启。
- 用户同意后由前台会话发起正常重启。经 UART 中断 U-Boot，原样加载 BOOT 的 Image / initramfs，只从 ROOT 读取试验 DTB（173,404 字节，U-Boot CRC32 `c98cf4b6` 与本地一致），以原启动参数一次性引导，未保存 U-Boot 环境。启动到 `6.1.172-8-z96a-bsp61`；VOP2 确认内屏 VP1 使用 Smart0，运行设备树 plane mask `0x15`、primary plane `4`。SDDM greeter 已在 tty2，eDP connected/enabled，首选模式像素时钟仍为请求 140.240 MHz、实际 140.166667 MHz。正式 DTB 和 extlinux 哈希仍与原镜像一致。待用户在内屏登录 Niri 验证。串口实录 `local/flash-stage/edp-plane-ab/trial-20261126-191312.uart.log`、`trial-postboot.uart.log`。
- 用户报告 Smart0 试启后已进入 Niri，但 Super+Enter 和 Super+T 未打开终端。UART 查明正在使用 Niri 自带配置：没有 `Mod+Return`，`Mod+T` 指向镜像未安装的 Alacritty。只修改板上 `alarm` 的两条快捷键为 Foot，保留备份 `~/.config/niri/config.kdl.before-z96a.aifp53NZ`，`niri validate` 通过并热加载。IPC 试开 Foot 后有已映射的窗口，eDP 仍为 1920x1080 @ 60.024 Hz。面板确认及两个物理按键组合待用户复核。
- 本地镜像构建脚本将创建 `alarm` 移至桌面 `/etc/skel` 配置就绪以后，补查 home 配置内容和属主，并从最终 ROOT 镜像检查两份配置；Niri 镜像两种快捷键均指向 Foot。`bash -n`、Niri 26.04 配置校验及 ext4 读回夹具通过，尚未提交或重跑 Actions。

### Smart0 默认启动

- 2026-11-26：用户确认一次性 Smart0 试启可进 Niri，并确认修正后的两个终端快捷键。板上保留 `alarm` 原密码。通过 UART 从只读 ROOT 的一次性维护引导修改 BOOT：只让 extlinux 默认 BSP 项改用 `/boot/dtbs/z96a-bsp61-edp-smart0.dtb`，备份旧配置至 `/boot/extlinux/extlinux.pre-smart0.conf`。原 DTB、原 Image / initramfs、NPU 禁用恢复项、U-Boot 环境与分区布局不变。写入后校验 DTB/配置 SHA-256、同步并卸载 BOOT。
- 2026-11-26：不干预后续自动启动，串口确认 U-Boot 从默认项载入新 DTB，`6.1.172-8-z96a-bsp61` 将 eDP 的 VP1 主平面设为 Smart0（mask `0x15`、ID `4`），SDDM 已启动并出现串口登录提示。实录 `local/flash-stage/edp-plane-ab/permanent-recovery.uart.log`。本次永久配置未重刷镜像。

`6.1.172-9` 镜像的构建验收与上板结果见 [11 月 27 日记录](2026-11-27-bsp-release.md)。
