# OpenWrt x86_64 自编译固件

基于 [coolsnowwolf/lede](https://github.com/coolsnowwolf/lede) 源码，通过 GitHub Actions 自动编译 x86_64 固件。默认不带 Docker，需要 Docker 请自行 fork 修改配置。

## 固件默认参数

| 项目 | 默认值 |
| --- | --- |
| 源码 | `coolsnowwolf/lede` master（每次构建拉取最新） |
| 内核 | 6.18.x（随上游更新，当前验证为 6.18.62） |
| LAN 地址 | `192.168.10.100/24` |
| 网关 / DNS | `192.168.10.1` |
| DHCP 地址池 | `192.168.10.10 - 192.168.10.99`（避开路由器自身 .100） |
| 登录账号 | `root`，默认密码为空 |
| IPv6 | 已开启，LAN `ip6assign 60` |
| 内核分区 / 根分区 | 32M / 800M |
| 镜像格式 | GRUB、EFI、VMDK（VMDK 为必需产物，缺失即停止发布） |
| LuCI 语言 | 简体中文 + English |
| 默认主题 | MMWX（内置 Argon、GWRT 可切换） |
| 音频组件 | 已全部移除（无声卡 / HDMI / USB 音频驱动） |

配置以 `files/etc/config/network` 与 `x86_64-full.config` 为准。

## 预装插件

### 网络与代理
- **OpenClash**：vernesong 官方源码，未使用 feed 内副本；只负责代理，DNS 劫持默认关闭。
- **MosDNS**：DNS 分流核心，`dnsmasq:53 → MosDNS:5335 → OpenClash:7874` 链路，国内直连、国外走 fake-ip。
- **EasyTier**：异地组网，含 `easytier` 内核、`luci-app-easytier` 界面与 Web Console（VPN → EasyTier）。
- **UPnP**：`miniupnpd-nftables` + `luci-app-upnp`，适配 firewall4/nftables。
- **socat**：端口转发。

### 系统与运维
- **ttyd**：Web 终端。
- **bash / nano / curl**：基础工具。
- **KMS**：`vlmcsd` + `luci-app-vlmcsd`。
- **AdGuardHome**：已编译进固件（含核心二进制），默认不启用；需要按客户端统计、家长控制或 DoH 时在 LuCI 手动开启并接管 53。
- **EQoS**：限速（使用 kenzok8 通用版，避开 NSS 依赖）。
- **TurboAcc**：网络加速面板，默认只开启 BBR；流量分载（Flow Offload）与 SQM/EQoS/OpenClash 冲突，需手动开启。
- **cpulimit**：CPU 占用限制。
- **netdata**：系统监控。
- **SQM**：QoS 队列。
- **diskman**：磁盘管理（含 btrfs-progs、lsblk、mdadm）。
- **luci-app-advanced**：高级设置。
- **netspeedtest**：网速测试（sirpdboy 原版及依赖）。
- **poweroff**：关机按钮。
- **定时重启**：`luci-app-autoreboot`（取自 ImmortalWrt），默认不启用；需要时在「系统 → 定时重启」开启并按周/日/月设置时间。
- **uhttpd**：LuCI Web 服务。

### IPv6
- `ipv6helper`、dnsmasq DHCPv6 支持、`6in4`、`6rd`、`6to4`、`iptables` / `ip6tables` 全套。

### 虚拟化客户机
- 当前 x86_64 6.18 内核已直接内置 VirtIO PCI/网卡/磁盘/SCSI/内存气球/控制台驱动。
- Hyper-V 网络、存储、内存气球等客户机驱动同样已内置，无需安装旧版 `kmod-hyperv-*` 软件包。
- `qemu-ga`：QEMU Guest Agent。
- x86 目标默认包已含 `kmod-vmxnet3`（VMware）、`kmod-e1000e`、`kmod-igb/igc` 等。

### 主题
- **MMWX**（默认）、**Argon**（jerrykuku 源）、**GWRT**（`rothdren-lion/gwrt-theme`）。

### 未预装 / 已停用
- Docker（`luci-app-dockerman`）、PassWall、SmartDNS、NPS/FRPS、Transmission/qBittorrent/Aria2、rclone、VSSR、PPTP 服务端、Watchcat、ServerChan 等均未启用。
- mwan3、syncdial、frpc、DDNSTO、minieap、mentohust 已移除，相关 feeds 不再引入。
- 音频组件已全部禁用（`kmod-sound-*`、`kmod-usb-audio`），用于减小镜像体积。
- VSSR 与 OpenClash 副本下载命令已移除。
- 不安装 Open vSwitch feed 包（会产生自依赖）。

## 构建说明

- **触发方式**：仅手动触发，在 Actions → Build OpenWrt_x86_64 → Run workflow 中运行，不再定时编译。
- **配置预检**：手动运行勾选 `config_only`，只检查 feeds、额外包和 `make defconfig` 后的最终配置，不下载、不编译、不发布。
- **构建环境**：`ubuntu-22.04`，超时 6 小时；先释放磁盘、安装依赖并恢复下载缓存。
- **下载阶段**：首次 `make download -j8` 最长 35 分钟，失败后降为 `-j4` 重试 25 分钟。
- **产物校验**：必须实际生成 x86 镜像且包含 `*.vmdk`，否则不创建 Release。
- **关键包校验**：MMWX、GWRT、OpenClash、MosDNS、网速测试、EQoS、TurboAcc、定时重启、EasyTier、UPnP、ttyd、KMS 若被 `defconfig` 丢弃，直接中断构建。
- **日志**：每次运行都会上传 `build-logs` Artifact，含 `config.log`、`download.log`、`build.log`、最终配置、feeds 配置和源码提交版本。
- **缓存**：编译使用 ccache，`dl` 目录使用 Actions 缓存；缓存键包含配置与 diy 脚本哈希。
- **清理**：仅保留最近 1 天的 Actions 运行记录和最近 30 个 Release。
- **上游检查**：`Update Checker` 保持手动触发，检测 LEDE 新提交后派发主工作流。

## 注意事项

1. **空密码有风险**：首次登录后请立即在「系统 → 管理权」设置 root 密码；同时建议只在内网开放 LuCI。
2. **浏览器**：请使用 Google Chrome 访问 LuCI，其他浏览器可能显示异常。
3. **默认主题与 GWRT**：默认 MMWX；GWRT 的状态页仪表盘只在其自身主题启用时渲染，其他主题保持原生状态页（脚本已打补丁）。
4. **EasyTier 版本**：使用官方预编译包（当前 2.7.0，编译时从 GitHub Release 下载），并移除了 feeds 中同名 Rust 版本，避免重复包。
5. **dnsmasq 补丁**：强制提供 `kmod-ipt-ipset`，解决 IPset 子选项与 `dnsmasq-full` 的循环依赖，供 OpenClash 使用。
6. **网速测试补丁**：移除已废弃的 `python3-pkg-resources` 依赖。
7. **EQoS 来源**：使用 kenzok8 软件源的通用版本；若切回官方源，需确认 x86 下不会被 NSS 依赖挡掉。
8. **外部源冲突**：kenzok8、mosdns、neobird、linkease 等额外 feeds 可能与上游包重名，出现覆盖警告属正常；新增插件后建议先跑 `config_only`。
9. **内核随上游变化**：构建拉取 master，内核补丁版本会变；如需固定版本，请在配置中锁定或改用固定分支。
10. **Docker**：默认未启用，且根分区 800M 对镜像运行偏紧，启用前请同步调大 `CONFIG_TARGET_ROOTFS_PARTSIZE`。
11. **VMDK**：发布强校验 VMDK；若上游移除该格式支持，构建会直接失败而不是发布缺件版本。
12. **虚拟机使用**：PVE 建议磁盘控制器选 VirtIO SCSI；ESXi 可直接使用 vmxnet3 网卡。
13. **DNS 链路**：`files/etc/uci-defaults/99-dns-chain` 首次启动会开启 MosDNS 的「DNS 重定向」，由其自动把 dnsmasq 上游改为 `127.0.0.1#5335`（无需手改 dnsmasq，停止服务时自动还原），同时关闭 OpenClash 的本地 DNS 劫持；MosDNS 的国外上游为 `127.0.0.1:7874`，因此**必须先启用 OpenClash**，否则国外域名无法解析。AdGuardHome 默认 `enabled=0`（未执行 `disable`，保留开机自启能力），需要时在 LuCI 开启，但需先把 MosDNS 的「DNS 重定向」关掉并把 AGH 提到 53。MosDNS 默认已开启 16000 条缓存、最小 TTL 60s、并发 3 与 GeoIP 每周一自动更新。
14. **TurboAcc**：`files/etc/uci-defaults/99-turboacc` 首次启动只开启 BBR，并显式关闭 `sw_flow/hw_flow/sfe_flow`。若在 LuCI 中开启流量分载，SQM、EQoS 限速和 OpenClash 透明代理会失效或异常；需要加速直连吞吐时，请先停用 SQM/EQoS。构建时也排除了 `SHORTCUT_FE`、`PDNSD` 等 Qualcomm 专用或与 MosDNS 重复的组件。
15. **单臂路由拓扑**：LAN 与网关同在 `eth0`（旁路由），公网端口映射实际由上级路由完成，UPnP 页面保留但无法直接映射公网端口。
16. **DHCP 池**：`files/etc/config/dhcp` 将地址池限定在 `.10-.99`，避免与路由器地址 `.100` 冲突；修改 LAN 网段后需同步调整 `start/limit`。

## 目录结构

```
.
├── .github/workflows/
│   ├── x86_64-full.yml      # 主构建流程
│   └── update-checker.yml   # 上游提交检查
├── files/                   # 固件默认配置（network、dhcp、shadow、uci-defaults/99-dns-chain、99-turboacc）
├── scripts/
│   ├── check-config.sh      # 关键包与配置校验
│   └── download-packages.sh # 依赖下载与重试
├── diy-part1.sh             # feeds 更新前：追加软件源
├── diy-part2.sh             # feeds 更新后：外部插件、补丁、默认项
├── x86_64-full.config       # 固件与插件选择
└── README.md
```

## 参考项目

**上游与工作流**
- [coolsnowwolf/lede](https://github.com/coolsnowwolf/lede)：固件源码。
- [P3TERX/Actions-OpenWrt](https://github.com/P3TERX/Actions-OpenWrt)：Actions 构建流程骨架。
- [P3TERX/ssh2actions](https://github.com/P3TERX/ssh2actions)：构建期间 SSH 调试。
- [jlumbroso/free-disk-space](https://github.com/jlumbroso/free-disk-space)、[klever1988/cachewrtbuild](https://github.com/klever1988/cachewrtbuild)：磁盘释放与编译缓存。
- [softprops/action-gh-release](https://github.com/softprops/action-gh-release)、[GitRML/delete-workflow-runs](https://github.com/GitRML/delete-workflow-runs)、[dev-drprasad/delete-older-releases](https://github.com/dev-drprasad/delete-older-releases)：发布与清理。

**插件与主题**
- [vernesong/OpenClash](https://github.com/vernesong/OpenClash)
- [EasyTier/luci-app-easytier](https://github.com/EasyTier/luci-app-easytier)
- [kenzok8/openwrt-packages](https://github.com/kenzok8/openwrt-packages)（EQoS、AdGuardHome、advanced 等）
- [sbwml/luci-app-mosdns](https://github.com/sbwml/luci-app-mosdns)
- [immortalwrt/luci](https://github.com/immortalwrt/luci)（定时重启）
- [coolsnowwolf/luci](https://github.com/coolsnowwolf/luci)（TurboAcc，取自 master 分支）
- [sirpdboy/NetSpeedTest](https://github.com/sirpdboy/NetSpeedTest)
- [esirplayground/luci-app-poweroff](https://github.com/esirplayground/luci-app-poweroff)
- [nickilchen/luci-app-socat](https://github.com/nickilchen/luci-app-socat)
- [jerrykuku/luci-theme-argon](https://github.com/jerrykuku/luci-theme-argon)
- [Wyatt323/luci-theme-mmwx](https://github.com/Wyatt323/luci-theme-mmwx)
- [rothdren-lion/gwrt-theme](https://github.com/rothdren-lion/gwrt-theme)
