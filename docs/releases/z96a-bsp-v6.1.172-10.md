# Sunniwell Z96A Arch Linux ARM BSP 6.1.172-10

面向 Sunniwell Z96A（Rockchip RK3568、aarch64）的 BSP 内核测试版。本次附件是内核安装包与对应源码，不是 eMMC 烧录镜像。

## 附件

- `linux-rockchip-z96a-6.1.172-10-aarch64.pkg.tar.xz`：Arch Linux ARM 内核包。安装和引导切换是两步；先阅读源码包中的 `packaging/README.md`，保留已知可启动的引导项。不要把此包作为裸镜像交给烧录工具。
- `linux-rockchip-z96a-6.1.172-10-source.tar.gz`：应用本版补丁后的完整内核源树，附二进制包中的实际内核配置、合并补丁、打包配置、构建与验证脚本。不含原厂启动固件、RKNN Runtime 或预编译模型。GitHub 自动生成的源码归档是本项目的文档快照，不是上述内核源码。
- `SHA256SUMS`：上述两份附件的 SHA-256 校验值，下载后运行 `sha256sum -c SHA256SUMS`。

## 构建与验收

- Rockchip BSP 基线 `77168c8d5ab82399f65a80e9f807b50ba37cf483`；rtw88 回移植基线 `6995e2de6891c724bfeb2db33d7b87775f913ad1`。
- [打包源码提交 `9b8690f`](https://github.com/Sakurapainting/PKGBUILDs/tree/9b8690f98fae24c67fbaaffa44cb305d4f6f52b0/core/linux-rockchip-z96a)；[Arch Linux ARM 原生构建与安装验收 `45936151382`](https://github.com/Sakurapainting/PKGBUILDs/actions/runs/45936151382)。
- 同版内核在一台 Z96A 上通过默认 BSP 冷启动、内屏 Niri 和 RKNN FP16 卷积测试，15 秒内 24,923 次推理输出全部匹配。没有验证其他硬件批次、Windows RKDevTool 或 KDE。

## 镜像状态

整盘 LBA0 镜像仍待核实原厂启动链和 RKNN Runtime 的再分发许可。BOOT/ROOT 镜像虽不含原厂启动链，但预装 RKNN Runtime，也暂不随本版分发。
