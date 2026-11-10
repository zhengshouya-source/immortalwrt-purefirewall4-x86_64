# 回归稳定 · 纯净 Firewall4

> **x86_64 ImmortalWrt 25.12.2 官方稳定维护版 · Daede 纯血 / Honk 新代 eBPF 双引擎 + Docker 全能主路由固件**
>
> 极致精简 · 100% 纯开源纯血 · Linux 6.12 LTS · 原生内核级 WireGuard · 全主流网卡覆盖

---

## 一句话定位

路由器是家庭 / 企业的网络中枢，核心诉求永远是 **长期稳定、久经考验、亚毫秒级低延迟、0 丢包**。

本固件不走"功能堆砌"路线，而是聚焦底层硬骨头：打通 Clang/LLVM BPF 编译链、原生开启 Linux 6.12 内核 BTF、攻克 Honk 阶段元数据缺失、注入多核并发扫描防卡死补丁，让数据包内核级零拷贝转发与全链路物理优化落到发烧友心中的理想状态。

底座采用官方 `openwrt-25.12` 维护分支最新提交 + Linux 6.12 LTS + GCC 14.3.0 黄金编译链，专为 x86_64 小主机 / 工控机（Intel 2.5G/千兆、Realtek 螃蟹 2.5G/5G、主流万兆光电卡）及 PVE/ESXi/Hyper-V 全平台虚拟化量身打造。

---

## 特性速览

| 维度 | 说明 |
| --- | --- |
| 底座 | ImmortalWrt 25.12.2 官方正式稳定版，全量对齐官方最新补丁，出厂即最新、零更新提示 |
| 内核 | Linux 6.12 LTS，硬核开启 `CONFIG_KERNEL_DEBUG_INFO_BTF=y` 原生 BTF |
| 工具链 | GCC 14.3.0 + Binutils 2.44 |
| 网络调优 | Socket 收发缓冲由 208 KB 跃升至 64 MB，单连接理论吞吐上限提升约 315 倍 |
| 防火墙 | 100% 现代纯净 Firewall4 (fw4) + Nftables，双向 FullCone NAT1 |
| 分流 | Daede 经典纯血 / Honk 新代 Rust eBPF，双引擎按需二选一 |
| 异地组网 | Linux 官方内核态 WireGuard，加解密直通内核，千兆/2.5G 线速 |
| 存储 | FUSE 3 全栈 (kmod-fuse + libfuse3 + fuse3-utils) + Btrfs |
| 空间 | 1024 MB 根分区，实测留存 900+ MB，Docker 随心部署 |
| 引导 | UEFI + Legacy BIOS (MBR) 双引导，输出 `.img.gz` 与 `.efi.img.gz` 双镜像 |

---

## 设计哲学

折腾的尽头是 **回归稳定** 与 **极致精简**。

市面上已有许多优秀的多功能全包固件，极大满足了玩法探索；本固件选择另一条取向——**回归稳定 · 极致精简 · 纯血性能**：

- **精简即极速、克制即稳定**：拒绝花哨无用的杂质插件与后台开销，深度精选裁剪核心组件，极低内存占用、零后台垃圾进程，把每一分 CPU 算力与硬件资源极致释放给数据包转发与 Docker 容器。
- **官方底座最坚固**：直接内置 openssl 3.5.9、curl 8.23.0 及最新 LuCI 界面，底层固若金汤，彻底告别崩溃与符号冲突。
- **100% 纯开源、零闭源风险**：拒绝一切第三方闭源插件与不可审计的二进制程序，纯血 Daede 与 Honk 双 eBPF 引擎各司其职。

---

## 核心技术亮点

### 1. 突破 208 KB 远古套接字瓶颈

- **208 KB 从何而来**：Linux 内核全局套接字最大收发缓冲硬编码默认值是 `212992` 字节（正好 208 KB），是 20 多年前拨号时代为防止极低内存硬路由 OOM 遗留的保守值。
- **为什么必须突破**：开启透明代理时，所有代理流量都会在本地建立 Socket 中继。按 BDP（带宽时延积 = 带宽 × 往返延迟）定律，在 150 ms 跨国高延迟下，208 KB 会把单连接最大速度锁死在约 1.38 MB/s（仅约 11 Mbps），千兆 / 2.5G 宽带单线程测速与单视频拉流严重受限。
- **本固件 64 MB 满血释放**：构建脚本自动注入 `99-custom-network-tuning.conf`，将 `rmem_max` / `wmem_max` 由 208 KB 跃升至 64 MB，单连接理论吞吐上限暴增约 315 倍，终结代理环境隐形限速。

### 2. 纯血原生 BTF · Daede 零拷贝代理内核

- 从源码层开启 `CONFIG_KERNEL_DEBUG_INFO_BTF=y`，Daede 直接接入 Linux 6.12 原生内核，原生 eBPF CO-RE 字节码直接载入，UDP 内核级零拷贝直连转发、TCP 高效分流，超低 CPU 负载。
- **三重闭环防污染隔离**：
  1. 内置 `down.dllkids.xyz` 软件源与官方 ECDSA 验签公钥，使 `daed` 核心可一键在线热更新；
  2. 独家 `@daed` 命名空间源标签沙盒隔离 + `package-manager-call` 智能过滤，LuCI 软件包页面自动屏蔽带 `@` 的隔离源，官方 feeds 软件 100% 保持原版纯正，杜绝同名魔改包（Docker、Lucky 等）覆盖冲突；
  3. 开机自愈脚本智能校验安装状态（`apk info -e`），仅当已装 `daed` 才在 `/etc/apk/world` 绑定 `@daed` 标记，实现 LuCI 界面一键在线热更。

### 3. Honk 新代 Rust eBPF + Doona 控制面板

- daeuniverse 生态新一代核心，Rust 重构数据面，继承 eBPF 零拷贝直连与精准分流，更低内存占用、更极致内存安全。
- 汲取 sing-box 现代设计精髓：原生兼容 Clash API、出站组 / 策略路由逻辑，配合现代化高颜值 Doona 静态 Web 面板，极客性能与直观易用兼得。
- 深度依托本固件 Linux 6.12 LTS 原生 BTF，无需外部 vmlinux-btf 补丁，内核原生载入 eBPF。

### 4. 全量对齐官方最新补丁

- 构建流程自动解除 `feeds.conf.default` 的 commit 锁定，全面跟踪拉取官方 `openwrt-25.12` 最新维护分支，刷机即最新、零更新提示。
- OpenSSL 固化为 `3.5.9`、cURL 固化为 `8.23.0`、LuCI 完全对齐官方镜像站最新状态，安全修复在编译期完成，不占 Overlay、无需在线热升级底层库。

### 5. FUSE 3 用户态文件系统全栈

深度集成 `kmod-fuse` / `libfuse3` / `fuse3-utils`，无缝联动 Docker 部署 Alist / Rclone / WebDAV，将阿里云盘、百度网盘、夸克、115、局域网 NAS 等一键挂载为本地目录，变身家庭轻 NAS 与流媒体库。

### 6. 纯净 Firewall4 + Nftables 极简链表

- UPnP 选用 `miniupnpd-nftables`，DNS 采用 `dnsmasq-full (nftset)` 原生集合分流，流量卸载基于 `kmod-nft-offload` / `kmod-nf-flow`，全链路统一调度。
- Daede / Honk 运行于 eBPF (netkit/TC) 零拷贝层，不依赖陈旧 iptables 规则链，规则链极简干净、零冲突，转发延迟压至极限。

### 7. 原生内核态 WireGuard 组网天花板

- 直接集成 Linux 6.12 LTS 原生 `kmod-wireguard`，加解密（ChaCha20-Poly1305 / Curve25519）由 CPU 硬件指令集直接完成，极低 CPU 负载，千兆 / 万兆对等流量轻松跑满，抖动极小。
- 呈现为标准纯净虚拟网卡 `wg0`，受 Firewall4 统一调度，不与 Daede / Honk 透明代理打架（无私有路由表 / fwmark 污染）。
- 两端具备公网 IPv6 时，搭配 IPv6 DDNS 端到端纯国内直连，PPPoE 重拨数秒内秒级重握手，不受境外骨干抖动影响。

### 8. 国内外网页极致秒开

结合 Daede / Honk (eBPF) 分流、全局 TCP BBR、网卡队列优化，国内主流网页 / 4K 流媒体 / 海外请求与 API 调用均实现毫秒级直连、0 延迟感知。

### 9. 双向原生 FullCone NAT (NAT1)

Firewall4 引擎原生集成并自动挂载 FullCone NAT 规则链，PS5 / Xbox / Switch、Steam 联机、P2P / PT 下载直接达到最高级别 NAT1。

### 10. 全局 TCP BBR 拥塞控制

内核编译级集成 `kmod-tcp-bbr=y`，开机自动全局生效（`net.ipv4.tcp_congestion_control = bbr`），长距离 / 高延迟 / 丢包网络下显著提升吞吐与响应。

### 11. 全系网卡驱动深度调优

Intel (igc/igb/e1000e)、Realtek (r8125/r8126/r8168) 均采用官方独立优化驱动；万兆卡 (ixgbe/mlx4/atlantic/bnx2x) 完整配置多队列接收与中断亲和性；显式关闭硬件分流 (`flow_offloading_hw=0`)，杜绝小包发热与丢包。

---

## 网卡与硬件兼容全景（免驱即插即用）

| 类别 | 内核驱动 | 代表型号 |
| --- | --- | --- |
| Intel 2.5G | `kmod-igc` | I225-V / I225-LM / I226-V |
| Realtek 2.5G | `kmod-r8125` | RTL8125 / RTL8125B / RTL8125BG |
| Realtek 5G | `kmod-r8126` | RTL8126 |
| Intel 千兆 | `kmod-igb` / `kmod-e1000e` / `kmod-e1000` | I350 / I210 / I211 / I219 / I217-218 / 82540 系列 / 虚拟机兼容 |
| Realtek 千兆 | `kmod-r8168` / `kmod-r8169` (+固件) | RTL8111 / RTL8168 |
| Intel 万兆 | `kmod-ixgbe` / `kmod-ixgbevf` / `kmod-i40e` / `kmod-iavf` | X520 / X540 / X550 / 82599 / X710 / XL710 |
| Mellanox 光卡 | `kmod-mlx4-core` / `kmod-mlx5-core` | ConnectX-3 / CX4 / CX5 |
| Aquantia 电口 | `kmod-atlantic` | AQC107 / AQC113 |
| Broadcom 万兆 | `kmod-bnx2x` (+固件) | BCM57810S |
| 虚拟化 | VMXNET3 / VirtIO / Hyper-V | ESXi / PVE(KVM) / Hyper-V |

**存储 / 磁盘**：原生 NVMe (`kmod-nvme`)、SATA AHCI (`kmod-ata-ahci`)、Btrfs (`kmod-fs-btrfs` / `btrfs-progs`)、FUSE 3、VirtIO 虚拟磁盘。

**物理复位**：`kmod-button-hotplug` + `kmod-input-core`，短按安全重启、长按 5 秒恢复出厂。

**引导模式**：UEFI + Legacy BIOS 双引导，无论现代小主机、老旧主机还是 PVE/ESXi 虚拟机均秒启。

---

## 开机自动初始化（99-custom-settings）

刷机开机后全自动静默完成 6 大底层优化：

1. **物理网口全量自适应识别**：扫描 `/sys/class/net/eth*`；多网口取数值最大的最后一个口绑 WAN（DHCP），其余桥接 `br-lan`；单网口强制唯一口为 LAN 并移除 WAN，杜绝失联变砖。
2. **纯净单网桥**：清空匿名 `br-lan`，重建唯一具名 `br_lan`，插任意 LAN 口稳定获取 `192.168.1.x`。
3. **纯净 DNS 防劫持**：删除默认 `wan6`，强制阿里 `223.5.5.5` + 腾讯 `119.29.29.29`，`peerdns='0'` 拒绝上游污染。
4. **FullCone NAT1 + 软件流量分载双开**：自动开启 NAT1 与 `flow_offloading=1`，显式关闭 `flow_offloading_hw=0`。
5. **Dropbear SSH & TTYD 免密直连**：清空 root 密码，TTYD 浏览器免密获取 root 命令行。
6. **官方源 / 第三方源双轨纯化**：清洗 404 伪源，保留天灵国内 CDN 镜像 + 内置 `down.dllkids.xyz` 源与公钥，`@daed` 标签沙盒隔离，国内 0 报错、秒级刷新。

---

## 集成插件矩阵

| 类别 | 组件 |
| --- | --- |
| 防火墙 | Firewall4 (fw4) + Nftables + miniupnpd-nftables + dnsmasq-full(nftset) + FullCone NAT1 |
| 代理分流（二选一） | **Daede** 经典纯血 / **Honk + Doona** 新代 Rust eBPF |
| 文件挂载 | FUSE 3（Alist / Rclone / 云盘挂载轻 NAS） |
| 容器 | Docker daemon + docker-compose + runc + 可视化面板 |
| 异地组网 | 内核态 WireGuard（kmod-wireguard + tools + luci-proto + rpcd-mod） |
| 万能工具 | Lucky（端口转发 / DDNS / WebDAV / STUN） |
| 终端诊断 | Ttyd / ethtool / lm-sensors / sysfsutils / bc / autocore |
| 物理按键 | Button Hotplug（短按重启 / 长按恢复出厂） |
| 磁盘存储 | Btrfs / Ext4 / fdisk / parted / e2fsprogs |
| UI 主题 | Argon & Argon Config |

---

## 安装与默认信息

- **默认后台 IP**：`192.168.1.1`
- **默认账号**：`root`
- **默认密码**：无（直接回车登录 Web 与 SSH）
- **固件版本**：ImmortalWrt 25.12.2（Linux 6.12 LTS）
- **目标架构**：x86_64（Squashfs + BIOS/UEFI 双引导）
- **剩余空间**：900+ MB（1024 MB 根分区）
- **镜像选择**：
  - `*-efi.img.gz` → 现代 UEFI 小主机 / PC / PVE( OVMF ) / ESXi UEFI
  - `*-combined.img.gz` → 传统 Legacy BIOS / PVE(SeaBIOS)
  - 物理机用 Rufus / BalenaEtcher 写入；PVE 用 `qm importdisk` 导入

---

## 下载 / Download

首个稳定版已发布（标签 `v25.12.2`）：

- **Release 页面**：https://github.com/zhengshouya-source/immortalwrt-purefirewall4-x86_64/releases/tag/v25.12.2
- **镜像文件**：`immortalwrt-x86-64-generic-squashfs-combined-efi.img.11-1.gz`（UEFI / combined-efi，约 142 MB）
- **sha256**：`e77a4b3c7ee8fe9fa731cdb1244f477d3272bbd5ac6b391d155f5e1593a4ea61`

> 刷机前务必校验 sha256，确认镜像完整、未被篡改。当前发布为 UEFI（combined-efi）镜像；传统 Legacy BIOS 镜像如需可另行构建。

---

## 使用须知：Daede / Honk 二选一

两者底层均直接挂载内核 eBPF / cgroup 零拷贝透明代理，**网络栈接管逻辑互斥，切勿同时运行**。若 Daede 在运行，Honk 维护页会主动抛出 `proxy_conflict` 保护性报错并阻止接管。

日常遵循 **二选一**：
- 想用 Honk：在 Daede 界面点【停止】并取消自启，Honk 报错自动消除并满血接管；
- 想用回 Daede：关闭 Honk 即可，无需卸载任何包。

---

## 全链路避坑：光猫协同（木桶理论）

把路由器优化到极致，网络仍偶发跳 Ping / 丢包，问题常在上游 **光猫**，且 **改桥接根本没用**：

1. **物理层闪断 (Link Flap)**：2.5G 光猫 EEE 节能与流控协商缺陷，改桥接依旧毫秒级微闪断，竞技游戏瞬间跳 Ping。
2. **后台流氓进程 / LXC 容器**：改桥接只改 WAN 转发类型，光猫定制系统后台仍 24h 狂奔，遥测探针与偷跑容器啃光弱芯内存，缓冲区溢出丢包。
3. **TR069 探针高频打点**：独立管理通道常在线，死占软中断；Conntrack/NAT 表被塞满即静默丢包。
4. **内部微交换架构积热**：数据必经光猫 Switch Fabric，70~80℃ 闷罐降频反压。

**三大黄金法则（缺一不可）**：
1. 极致纯净软路由（64 MB 缓冲 + eBPF 零拷贝 + NAT1）；
2. 光猫底层彻底净化（强杀探针 / 容器、锁死 2.5G 满血双工，单改桥接无效）；
3. 手动指定物理最近、无拥堵的最优 BRAS 拨号。

> 建议搭配光猫终极调校白皮书协同食用，达成全链路 0 丢包 · 亚毫秒级低延迟。

---

## 构建工程调优（setup_menu.sh）

- 源码构建期直接注入 64 MB 缓冲，破除 BDP 枷锁；
- 菜单规范排序：Daede(10) → Honk(20) → Lucky(30) → rtp2httpd(40) → IPTV回看(50) → UPnP(60) → Docker(70)，Dockerman 重命名为 Docker；
- 内嵌 Clang/LLVM 路径兼容、Feeds 排重、`git pull` 防卡死兜底、Honk 缺失文件智能补齐、`99-fix-apk-repositories` 伪源清洗与 `@daed` 隔离、JSON 菜单语法自检门禁；
- `include/scan.mk` 内嵌 `mv -f` 根治只读缓存卡死；出包阶段深度提纯，按 BIOS/UEFI 智能保留纯净镜像；
- 全量 Kconfig 字典排序与依赖树整理，无语法废行与冲突项。

---

## 致谢

- 底层硬骨头借助现代前沿 AI 深度工程协同打磨；
- 代理内核 Daede / Honk 均源自 [daeuniverse](https://github.com/daeuniverse) 生态；
- 配套光猫调校白皮书为本项目姐妹篇，建议双端协同。
