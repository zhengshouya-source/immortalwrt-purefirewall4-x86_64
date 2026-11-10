# 刷机与双引擎配置指南

> 适用固件：`ImmortalWrt 25.12.2` 纯净 Firewall4（x86_64 · Linux 6.12 LTS）
> 配套仓库：[immortalwrt-purefirewall4-x86_64](https://github.com/zhengshouya-source/immortalwrt-purefirewall4-x86_64)

本指南覆盖：刷机 → 初始配置 → Daede / Honk 双引擎（二选一）→ WireGuard 组网 → Docker + FUSE 轻 NAS → 常见问题。

---

## 0. 准备工作

| 项目 | 说明 |
| --- | --- |
| 固件镜像 | Release 中 `immortalwrt-x86-64-generic-squashfs-combined-efi.img.11-1.gz`（UEFI） |
| 校验 | 刷机前务必核对 sha256：`e77a4b3c7ee8fe9fa731cdb1244f477d3272bbd5ac6b391d155f5e1593a4ea61` |
| 写入工具（物理机） | Rufus / BalenaEtcher（把解压后的 `.img` 写入目标 SSD / U 盘） |
| 虚拟化（PVE） | `qm importdisk` 导入磁盘镜像 |
| 接线 | WAN 接光猫 / 上联；LAN 口接交换机或电脑 |

> ⚠️ 刷机有变砖风险，请先备份原路由配置，并确认已能进原系统后台或拥有物理机 Console / IPMI 救砖手段。

---

## 1. 刷机

### 1.1 物理机（N100 / N5105 / J4125 / N305 等 UEFI 小主机）

```bash
# 解压镜像（在本机执行）
gunzip immortalwrt-x86-64-generic-squashfs-combined-efi.img.11-1.gz
```

1. 用 Rufus / BalenaEtcher 把解压后的 `.img` 写入目标磁盘；
2. 将该磁盘设为机器首选启动项（UEFI 模式）；
3. 开机，等待约 30~60 秒让 `99-custom-settings` 完成首次自适应。

### 1.2 PVE 虚拟机

```bash
# 在 PVE 节点 shell 中，把镜像导入某 VM 的存储（以 vmbr0 / local-lvm 为例）
qm importdisk <VMID> immortalwrt-x86-64-generic-squashfs-combined-efi.img.11-1 local-lvm
```

- **BIOS**：选 `OVMF (UEFI)`（对应 efi 镜像）或 `SeaBIOS`（对应 combined-efi 也可，建议 UEFI）；
- 把导入的 `unused disk` 挂载为 **IDE / VirtIO / SCSI**，设置启动顺序；
- 网卡建议用 `VirtIO`（半虚拟化），驱动已内置。

### 1.3 首次启动会发生什么

开机脚本 `99-custom-settings` 自动完成：

- **网口自适应**：多网口取数值最大的最后一个口绑 WAN（DHCP），其余桥接 `br-lan`；单网口强制唯一口为 LAN 并移除 WAN（防失联变砖）；
- **纯净单网桥**：重建唯一具名 `br-lan`，插任意 LAN 口稳定获取 `192.168.1.x`；
- **DNS 防劫持**：删除默认 `wan6`，强制阿里 `223.5.5.5` + 腾讯 `119.29.29.29`，`peerdns=0`；
- **NAT1 + 软分载双开**：FullCone NAT1 与 `flow_offloading=1` 自动开启，`flow_offloading_hw=0` 显式关闭；
- **TTYD 免密**：浏览器打开 TTYD 即可直获 root 命令行。

---

## 2. 初始配置（必做）

1. 浏览器访问 `http://192.168.1.1`，账号 `root`、**无密码**直接登录；
2. **立刻修改 root 密码**（系统 → 管理权 → 密码）——出厂无密码仅方便首次接入，长期暴露风险大；
3. 确认 **网络 → 接口**：
   - `br-lan` 存在且唯一，IP 为 `192.168.1.1`；
   - `wan` 已获取 IP（DHCP 或 PPPoE，按你的上联方式）；
4. 如需 PPPoE 拨号：在 `wan` 接口改为 `协议 = PPPoE`，填入运营商账号密码；
5. 测速 / 游戏前，建议先确认 FullCone NAT1 已生效（见文末常见问题）。

---

## 3. Daede 配置（经典纯血 eBPF）

Daede 出厂已内置，并可通过 `@daed` 隔离源一键在线热更。

1. **更新守护核心**：LuCI → Daede → 界面右上角【升级】按钮（已绑定 `@daed` 通道，国内 0 报错秒级更新）；
2. **导入订阅**：Daede → 订阅 → 粘贴你的订阅链接（或手动添加节点）；
3. **路由规则**：
   - `direct`（直连）：国内地址、局域网、含 `geosite:cn` 等；
   - `proxy`（代理）：海外流量，可选按分组（如 `proxy-us` / `proxy-hk`）；
4. **启动并自启**：点【启动】，并勾选开机自启；
5. 验证：终端 `curl ip.sb` 看出口 IP 是否切换；国内站点应直连、海外走代理。

---

## 4. Honk 配置（新代 Rust eBPF + Doona）

> 仅当你想用 Honk 时配置；**与 Daede 互斥**，详见第 5 节切换。

1. LuCI → Honk → Doona 面板（现代化 Web UI）；
2. **订阅 + 出站组**：导入订阅，按 sing-box 风格配置 `outbound groups`（如 `auto` / `proxy` / `direct`）；
3. **策略路由**：设置规则集（域名 / IP / 地理站点），Doona 实时生效；
4. **启动**：在 Honk 界面点启动，Doona 显示 `running` 即接管成功。

---

## 5. Daede / Honk 二选一切换

两者均直接挂载内核 eBPF / cgroup 零拷贝透明代理，**网络栈接管逻辑互斥，切勿同时运行**。

- **从 Daede 切到 Honk**：Daede 界面点【停止】并取消自启 → Honk 维护页 `proxy_conflict` 报错自动消除 → Honk 满血接管；
- **从 Honk 切回 Daede**：关闭 Honk → Daede 直接启动即可，**无需卸载任何包**。

> 若 Honk 仍报 `proxy_conflict`，确认 Daede 进程已彻底停止（`ps | grep daed`），再重启 Honk。

---

## 6. WireGuard 异地组网（内核态）

1. LuCI → 网络 → 接口 → 添加 `WireGuard` 接口（依赖 `luci-proto-wireguard` 已内置）；
2. 生成密钥对（界面可一键生成），把**本端公钥**发给对端、把**对端公钥 + 允许的 IP** 填入；
3. 防火墙：把 `wg0` 接口划到 `VPN` 区（或 `lan` 区，视互通需求），并放通对应转发；
4. 两端均有公网 IPv6 时，配合 IPv6 DDNS 可端到端纯国内直连，PPPoE 重拨数秒重握手；
5. `wg0` 是纯净虚拟网卡，受 Firewall4 统一调度，**不与 Daede / Honk 透明代理冲突**。

---

## 7. Docker + FUSE 轻 NAS

1. LuCI → Docker（Dockerman 重命名后的统一面板）确认 daemon 已运行；
2. 部署 **Alist / Rclone** 容器，挂载阿里 / 百度 / 夸克 / 115 / WebDAV 等；
3. 通过 **FUSE 3**（`kmod-fuse` + `libfuse3`）把云盘挂载为本地目录（如 `/mnt/cloud`），供 Infuse / Kodi / PotPlayer 直接访问；
4. 根分区 1024 MB、实测留存 900+ MB，容器空间充裕。

---

## 8. 常见问题

**Q1：Honk 提示 `proxy_conflict`？**
A：Daede 仍在运行。到 Daede 界面【停止】并取消自启，报错即消失。见第 5 节。

**Q2：单网口小主机刷完进不去后台？**
A：开机脚本已把唯一口强制为 LAN（移除 WAN），直连该口获取 `192.168.1.x` 即可。若仍失联，用 TTYD / 串口 Console 检查 `br-lan`。

**Q3：如何确认 FullCone NAT1 生效？**
A：网络 → 防火墙 → 常规设置，确认 WAN 区「FullCone NAT」已开启；或用 NAT 检测站点看是否 NAT1。

**Q4：代理下跨国大流量还是慢？**
A：检查是否真用上了本固件 64 MB 套接字缓冲（`cat /proc/sys/net/core/rmem_max` 应为 67108864），并确认走的 eBPF 直连而非用户态 TUN。

**Q5：网络偶发跳 Ping / 丢包？**
A：多半在上游光猫（改桥接无效）。按「三大黄金法则」净化光猫：强杀探针 / 偷跑容器、锁死 2.5G 满血双工、手动指定最优 BRAS 拨号，与本网卡 eBPF 零拷贝 + NAT1 协同。

---

## 9. 安全与维护建议

- 出厂无密码，首次登录务必设置 root 密码；
- 非必要不开放 WAN 侧管理端口；
- 镜像升级请重新刷写并保持配置备份（Squashfs 支持保留配置升级）；
- 官方源已国内 CDN 加速、零报错；Daede 走 `@daed` 隔离源热更，避免引入第三方同名魔改包。
