# ImmortalWrt 24.10.x — 奇虎 360T7（custom U-Boot layout，刷不死 U-Boot 布局）

> **给谁用**：手上是 **Qihoo 360T7（奇虎 360 T7）**，且 U-Boot 被魔改成
> **"刷不死"恢复版**（custom U-Boot layout，`fip`/`FIP` 分区里保留厂商/大神 U-Boot +
> 192.168.1.1 中文恢复网页）的用户——本仓库让这类机器用上**新版 ImmortalWrt**，
> 同时**完整保留刷不死恢复能力**。

## 本仓库是什么

一个 **构建 overlay**（不是完整固件源码，只有几十 KB）：

```
files/    mt7981b-qihoo-360t7-ubootmod.dts     ubootmod 设备树（保留厂商 U-Boot 引导链）
patches/  900-qihoo-360t7-ubootmod-device.mk   filogic.mk 设备定义（KERNEL_IN_UBI 等）
config/   24.10.6.config                        该版本经验证的完整构建配置（与官方 CI 同种子）
scripts/  build.sh                              一键构建（拉官方 tag → 套 overlay → make）
          release.sh                            一键发布 GitHub Release
.github/  workflows/build.yml                   Actions 自动构建+发布
LICENSE   GPL-2.0-only（与 ImmortalWrt/OpenWrt 一致）
```

构建时基于**官方 ImmortalWrt 源码 tag** + 官方 CI 配置种子（`config.buildinfo` 同方法展开），
软件包集合与官方发布一致；差异仅在分区布局（ubootmod 经典 UBI 三卷：kernel/rootfs/rootfs_data）。

## Releases（直接下载固件）

每个版本一个 tag/Release，含 3 个镜像 + `sha256sums.txt`：

| 文件 | 用途 |
|---|---|
| `*-squashfs-factory.bin` | **完整 UBI 镜像**——刷不死 U-Boot 网页（192.168.1.1）上传这个 |
| `*-squashfs-sysupgrade.bin` | 运行中系统内 `sysupgrade` 升级（**保留配置**） |
| `*-initramfs-kernel.bin` | FIT 内核镜像（调试/恢复辅助；刷不死网页**不吃**它） |

当前已发布：`v24.10.6`（ImmortalWrt 24.10.6，内核 6.6，opkg 包管理）。

## 刷机（刷不死 U-Boot 布局）

1. **备份**：在当前系统 SSH 里 `tar czf /tmp/backup.tar.gz /etc/config`，把文件存到电脑。
2. **FACTORY 直刷（全新安装）**：按住 reset + 通电 5~10 秒进刷不死网页（192.168.1.1，
   电脑网线直连 LAN 口并设 192.168.1.2/24）→ 上传 `*-squashfs-factory.bin` → 重启 → 完整系统。
3. **sysupgrade（版本间升级，保留配置）**：把 `*-squashfs-sysupgrade.bin` 传到运行系统
   （scp/LuCI 文件管理），然后 `sysupgrade /tmp/<该文件>`。

> 升级/回退的跨版本兼容性细节（哪些方向需要 `-f`）见下方"版本间升级/回退"。

## ⚠️ 红线

**永远不要刷 `fip` / `preloader` 分区**。那会替换掉你的刷不死 U-Boot，直接失去恢复能力。
本仓库固件不触碰这两个分区。

## 版本间升级/回退（24.10 ↔ 23.05 等）

- **新版本镜像里含当前 board 名** → 直接 `sysupgrade` 即可（保留配置）。
- **旧版本镜像的兼容列表可能不含当前 board 名**（命名不一致）→ 需 `sysupgrade -f`
  （UBI 布局相同，强制安装安全）。
- 兜底：任何时候都能用刷不死网页上传上一版本的 `factory.bin` 回退。

## 自己构建（一键）

```sh
git clone <本仓库地址>
cd <仓库目录>
./scripts/build.sh 24.10.6 2        # 2 = CPU 核数；2 核约 8~10 小时
# 产物: ~/iwrt-build/iwrt-24.10.6/bin/targets/mediatek/filogic/
```

发布 Release（可选）：

```sh
GITHUB_TOKEN=<你的token> ./scripts/release.sh 24.10.6
```

**新增小版本（如将来出 24.10.7）**：等官方发布 tag 后，把 `config/24.10.6.config` 复制为
`config/24.10.7.config`（通常无需改动，个别情况按官方 config.buildinfo 微调），提交后
打 tag `v24.10.7`（或等定时任务自动检测）→ 自动构建并发布。

## GitHub Actions

- **tag 触发**：打 `v24.10.7` 之类的 tag 即自动构建 + 发布 Release。
- **手动触发**：Actions 页面 → Build & Release → Run workflow → 填版本。
- **定时检测**：每天北京时间 00:00 自动检查"官方已发版 + 本仓库有对应 config + 尚未发布"，
  命中即自动构建发布（所以"官方一发新版，只需补一份 config 文件，固件就自动出来"）。
- 构建在 **self-hosted runner**（低配 Linux 机器）上执行；`config/<版本>.config` 不存在时
  直接报错，保证"一版本一配置"的纪律。

## 许可证

GPL-2.0-only，与 ImmortalWrt / OpenWrt 一致（见 `LICENSE`）。
