# OpenCore EFI — i5-14600K / RX 6600 XT / macOS 26 Tahoe

本 EFI 适用于 **Intel 第 14 代平台 + AMD RX 6600 XT + 博通 BCM94360CD** 的黑苹果，基于 OpenCore，安装在本机 ESP，目前可正常启动并日常使用。

> 当前系统：**macOS 26.5 Tahoe (25F71)** · OpenCore **1.0.8** · SMBIOS **MacPro7,1**

---

## 硬件配置

| 组件 | 型号 | 状态 |
|------|------|------|
| CPU | Intel Core **i5-14600K**（Raptor Lake，6P+8E / 14C20T）| ✅ |
| 主板 | **ASUS TUF GAMING B760M-PLUS WIFI II D5**（B760 / DDR5）| ✅ |
| 内存 | 64GB（32GB×2）KingBank DDR5 4800MHz | ✅ |
| 显卡 | **AMD Radeon RX 6600 XT** 8GB（Navi 23）| ✅ 原生免驱 |
| 存储 1 | 致钛 ZHITAI TiPro7000 1TB NVMe（PCIe 4.0 ×4）| ✅ 启动盘 Macintosh HD |
| 存储 2 | SOLIDIGM SSDPFKKW020X7 2TB NVMe | ✅ |
| 声卡 | Realtek **ALC897**（板载）| ✅ alcid=11 |
| 有线网卡 | Realtek **RTL8125** 2.5GbE（板载）| ✅ en0 |
| 无线网卡 | **BCM94360CD**（Fenvi T919，PCIe）| ⚠️ Tahoe 下需 OCLP-Mod root patch（见下）|
| 蓝牙 | **BCM20702B0**（网卡自带，USB）| ✅ 原生免驱 |
| USB-C | 1× USB 3.2 **Gen 2×2（20Gbps）** | 非雷雳；macOS 限 10Gbps（见下）|
| SMBIOS | MacPro7,1 | - |

> ⚠️ **主板兼容性说明**：本配置针对 **ASUS TUF B760M-PLUS WIFI II D5** 定制。换主板需调整：声卡 `alcid`、`UTBMap`（USB 端口映射）、板载网卡驱动、部分 SSDT。
> 板载 Realtek 无线已在 DeviceProperties 中禁用（改用博通 Fenvi 卡）。

### 功能状态

| 功能 | 状态 |
|------|------|
| 显卡硬件加速 / Metal | ✅ |
| 睡眠 / 唤醒 | ✅ |
| 有线网络（2.5G）| ✅ |
| 蓝牙 | ✅ |
| 音频（板载 / 显示器 / USB）| ✅ |
| USB 端口映射 | ✅ |
| OTA 软件更新 | ✅（revpatch=sbvmm）|
| iMessage / FaceTime | ✅（需生成有效序列号）|
| **WiFi** | ⚠️ Tahoe 下需 OCLP-Mod root patch（默认不工作）|
| **AirDrop / 随航 / 接力** | ⚠️ 依赖 WiFi 的 AWDL，WiFi 恢复后才可用 |
| **USB 20Gbps** | ⚠️ macOS 不支持 Gen 2×2，最高 10Gbps |

---

## 仓库结构

```
.
├── release/   # RELEASE 版 EFI（静默启动，日常使用）→ BOOT/ + OC/
├── debug/     # DEBUG 版 EFI（屏幕输出 OpenCore 日志，排错用）
└── README.md
```

两套**仅二进制构建类型不同**（OpenCore.efi + Acidanthera 系 kext 的 RELEASE/DEBUG），`config.plist`、ACPI、USB 映射等完全一致，可互换。

### 部署 / 切换（ESP 分区操作）
```bash
mv /Volumes/ESP/EFI /Volumes/ESP/EFI-bak      # 先备份当前
cp -R release /Volumes/ESP/EFI                # 日常用 RELEASE
# 排错时改用： cp -R debug /Volumes/ESP/EFI
```
> 操作前务必保留可启动备份（如 Ventoy U 盘），以便回滚。

---

## EFI 概要

| 项 | 值 |
|----|----|
| OpenCore | 1.0.8（release=RELEASE / debug=DEBUG）|
| SMBIOS | MacPro7,1 |
| boot-args | `keepsyms=1 alcid=11 ipc_control_port_options=0 revpatch=sbvmm amfi=0x80` |
| SIP（csr-active-config）| `0x0803`（允许第三方 kext + 不受限文件系统）|
| SecureBootModel | `Disabled` |
| Misc→Debug→Target | `0`（静默，直接进 logo）|
| 启动主题 | OpenCanopy `Acidanthera\GoldenGate` |
| ShowPicker / Timeout | 显示选择器 / 5 秒 |

### ACPI（SSDT）

| 文件 | 作用 |
|------|------|
| `SSDT-AWAC.aml` | 修复 AWAC 时钟，启用 RTC |
| `SSDT-EC-USBX.aml` | 嵌入式控制器 + USB 供电 |
| `SSDT-PLUG-ALT.aml` | CPU 电源管理（XCPM plugin-type）|

### Kexts

**基础**
| Kext | 版本 | 作用 |
|------|------|------|
| Lilu | 1.7.3 | 补丁框架，最先加载 |
| VirtualSMC | 1.3.8 | SMC 模拟 |
| AppleMCEReporterDisabler | 1.2 | MacPro7,1+消费 CPU 防 MCE panic |

**硬件驱动**
| Kext | 版本 | 作用 |
|------|------|------|
| AppleALC | 1.9.8 | 声卡（layout 11 / ALC897）|
| LucyRTL8125Ethernet | 1.2.300 | 2.5G 有线网卡 |
| USBToolBox + UTBMap | 1.2.0 / 1.1 | USB 端口映射（本主板定制）|
| NVMeFix | 1.1.4 | NVMe 电源管理 |
| AirportBrcmFixup | 2.2.1 | 博通 WiFi 补丁（配合 root patch）|

**CPU**
| Kext | 版本 | 作用 |
|------|------|------|
| CpuTopologyRebuild | 1.1.0 | 14 代大小核拓扑 |
| CPUFriend + DataProvider | 1.3.1 / 1.0.0 | CPU 调频数据 |
| SMCProcessor / SMCSuperIO | 1.3.8 | CPU 温度 / 风扇监控 |

**系统兼容**
| Kext | 版本 | 作用 |
|------|------|------|
| RestrictEvents | 1.1.7 | VMM 伪装保 OTA（revpatch=sbvmm）|
| AMFIPass | 1.4.1 | 配合 `amfi=0x80` |

**已禁用 / 未使用**：WhateverGreen、NootRX（RX 6600 XT 原生免驱）；BrcmPatchRAM 系（BCM94360CD 原生蓝牙）。

> WiFi 所需的 `IO80211FamilyLegacy.kext` / `IOSkywalkFamily.kext` **不在本 EFI 内**——它们由 OCLP-Mod 的 root patch 注入（见「WiFi 恢复」）。

### Kernel Block
| Identifier | 作用 | 状态 |
|------------|------|------|
| `com.apple.iokit.IOSkywalkFamily` | 屏蔽原生 Skywalk，走旧 WiFi 路径 | 启用（MinKernel 23.0.0）|

### DeviceProperties
| PCI 路径 | 作用 |
|----------|------|
| `…/Pci(0x1,0x0)/…/Pci(0x0,0x0)` | GPU 属性（RX 6600 XT）|
| `PciRoot(0x0)/Pci(0x1F,0x3)` | 声卡（ALC897 注入）|
| `PciRoot(0x0)/Pci(0x1C,0x3)/Pci(0x0,0x0)` | **禁用板载 Realtek 无线**（已换博通）|

### Drivers
`OpenRuntime.efi`（必需）· `OpenCanopy.efi`（图形选择器）· `HfsPlus.efi`（必需）· `ResetNvramEntry.efi` · `ToggleSipEntry.efi`

---

## ⚠️ WiFi / AirDrop 恢复（Tahoe）

macOS 26 Tahoe 移除了对 BCM4360/BCM94360CD 旧博通网卡的原生驱动，**WiFi 默认不工作**。需用 **OCLP-Mod（laobamac fork，3.1.0+）** 的 root patch 注入旧驱动恢复。

**前置条件（本 EFI 已全部满足）**：SecureBootModel=Disabled · csr=0x0803 · AMFIPass 1.4.1 · boot-args 含 `amfi=0x80` · 已 Block `IOSkywalkFamily`。

**步骤**：
1. 进系统打开 **OCLP-Mod** → Build/Install OpenCore（让它注入 WiFi kext）
2. OC 菜单 **Reset NVRAM**
3. OCLP-Mod → **Post-Install Root Patch → Start Root Patching** → 重启
4. **每次 macOS 更新后需重新打 root patch**

---

## USB / 雷雳 说明

- 本机**无雷雳/USB4 控制器**，唯一 Type-C 为 **USB 3.2 Gen 2×2（20Gbps）** 纯 USB 口。
- **macOS 不支持 USB 3.2 Gen 2×2**，该口最高只能 **10Gbps（Gen 2×1）**——系统限制，非 EFI 问题。
- USB 端口由 USBToolBox + UTBMap 映射（≤15 口），含 2 个 Type-C（connector=9）。

---

## 常用配置修改

**1. 生成新序列号**（分享/避免冲突，务必先做）
用 [GenSMBIOS](https://github.com/corpnewt/GenSMBIOS) 或 OCAuxiliaryTools 重新生成 `PlatformInfo→Generic` 的 `SystemSerialNumber` / `MLB` / `SystemUUID` / `ROM`。

**2. 声卡 layout-id**：声音异常时改 boot-args 的 `alcid=`（ALC897 常用 1/2/3/11/13/28）。

**3. 启用/禁用 Kext**：`config.plist→Kernel→Add`。注意顺序：Lilu 第一 → VirtualSMC → 其余依赖 Lilu 的。

**4. 启动菜单**：`Misc→Boot→ShowPicker`；或开机按住 Option/Esc。

---

## 故障排查

- **启动黑屏/卡住**：改用 `debug/` 版（或临时加 `-v`）看日志；关注 GPU(`EXITBS`) / 显示驱动(`IOConsoleUsers`) 卡点。
- **WiFi 扫不到**：确认已跑 OCLP-Mod root patch（更新后需重跑），且 AMFIPass / 注入的 WiFi kext 生效。
- **有线不通**：确认 `LucyRTL8125Ethernet.kext` 启用。
- **Win/Mac 时间不同步**（双系统）：Windows 管理员运行
  `reg add "HKLM\System\CurrentControlSet\Control\TimeZoneInformation" /v RealTimeIsUniversal /d 1 /t REG_DWORD /f`

---

## 更新说明

- **OpenCore / Kext 来源**：dortania **build-repo**（与 heipg.cn 编译同源、版本一致，已校验字节一致）；CpuTopologyRebuild / USBToolBox / LucyRTL8125 取自各自官方仓库。
- **更新 OC**：整套替换 `OpenCore.efi` + `BOOTx64.efi` + `Drivers/`（OpenRuntime 必须与 OpenCore 同版本），用 `ocvalidate` 验证 config。
- **更新 Kext**：保持 Lilu 与其插件版本兼容，建议同批更新。

## 相关资源
- [OpenCore 安装指南](https://dortania.github.io/OpenCore-Install-Guide/) · [OpenCorePkg](https://github.com/acidanthera/OpenCorePkg/releases) · [build-repo 构建](https://github.com/dortania/build-repo) · [OCLP-Mod](https://github.com/laobamac/OCLP-Mod)

---

## 更新记录

### 2026-06-07
- macOS 26.5 Tahoe (25F71) + OpenCore 1.0.8 配置定稿
- RELEASE 静默构建（开机直接进 logo，无 `OC: Starting...` 日志）
- boot-args：`keepsyms=1 alcid=11 ipc_control_port_options=0 revpatch=sbvmm amfi=0x80`
- `Misc→Debug→Target=0`；`Timeout=5`；启动主题 `Acidanthera\GoldenGate`
- 声卡 ALC897（alcid=11）；USB 映射 USBToolBox + UTBMap；板载 Realtek 无线已禁用，改用博通 Fenvi
- 仓库分 `release/` + `debug/` 两套完整 EFI
- ⚠️ WiFi 需用 OCLP-Mod 在 Tahoe 上 root patch 恢复
