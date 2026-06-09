---
tags: [nordic, ncs, zephyr]
category: mcu/nrf
created: 2026-06-09
---
Here is the generated Obsidian markdown note:

```markdown
---
title: nRF Connect SDK (NCS)
category: mcu/nrf
tags:
  - nrf
  - ncs
  - zephyr
  - nordic-semiconductor
  - rtos
  - bluetooth
  - matter
  - lte
created: 2026-06-09
status: active
version: v3.1.1
---

# nRF Connect SDK (NCS)

## 项目概述

nRF Connect SDK (NCS) 是 Nordic Semiconductor 官方提供的嵌入式开发 SDK，基于 Zephyr RTOS 构建，支持 nRF52、nRF53、nRF54、nRF7120、nRF91 等全系列芯片。NCS 集成了蓝牙、Matter、LTE-M/NB-IoT、Wi-Fi、Thread 等无线协议栈，是 Nordic IoT 产品的统一开发平台。

## 技术栈

| 组件 | 版本/说明 |
|---|---|
| Zephyr RTOS | v4.1.99 (sdk-zephyr fork) |
| NCS 版本 | v3.1.1 |
| 构建系统 | CMake + west (Zephyr meta-tool) |
| 工具链 | ncs-toolchain-x86_64-windows (bundle c1a76fddb2) |
| SDK 版本 | Zephyr SDK 0.17.1 |
| 密码学库 | mbedtls, oberon-psa-crypto, tinycrypt |
| Bootloader | MCUboot |
| 配置系统 | Kconfig + Devicetree |

## 架构与关键设计

### West Workspace 结构

```
ncs/v3.1.1/
├── .west/config          # west manifest 指向 nrf/west.yml
├── zephyr/               # Zephyr RTOS (sdk-zephyr fork, ncs-v3.1.1 分支)
├── nrf/                  # NCS 主仓库 (manifest repo)
│   ├── applications/     # 完整应用示例
│   ├── samples/          # 功能示例代码
│   ├── subsys/           # 子系统 (BT, DFU, net 等)
│   ├── lib/              # 通用库
│   ├── drivers/          # 外设驱动
│   ├── boards/           # 板级支持包
│   └── soc/              # SoC 定义
├── nrfxlib/              # Nordic 私有库 (闭源二进制)
│   ├── nrf_modem/        # LTE 调制解调器库
│   ├── nrf_802154/       # 802.15.4 协议栈
│   ├── mpsl/             # 多协议调度层
│   ├── crypto/           # 加密库
│   └── nrf_rpc/          # 跨核 RPC
├── bootloader/mcuboot/   # MCUboot 安全启动
├── modules/              # 集成模块 (crypto, fs, hal 等)
└── tools/                # 构建/调试工具
```

### 核心设计模式

1. **West Manifest 管理**: `nrf/west.yml` 定义所有依赖仓库及版本，通过 `west update` 统一拉取和同步
2. **Kconfig + Devicetree 双配置**: Kconfig 控制软件功能开关，Devicetree 描述硬件拓扑
3. **Sysbuild 构建**: 支持多镜像构建 (应用 + MCUboot + TF-M)，通过 `sysbuild` 统一管理
4. **多协议架构**: MPSL (Multi-Protocol Service Layer) 实现蓝牙、Thread、802.15.4 等协议的时分复用

## 核心知识点

### 支持的 SoC 系列
- **nRF52 系列**: nRF52840, nRF52833, nRF52820 (BLE, 802.15.4)
- **nRF53 系列**: nRF5340 (双核: 应用核 + 网络核)
- **nRF54 系列**: nRF54LM20, nRF54LV10 (新一代低功耗)
- **nRF7120**: Wi-Fi SoC
- **nRF91 系列**: nRF9160/nRF9151 (LTE-M/NB-IoT + GNSS)

### 关键子系统

| 子系统 | 路径 | 说明 |
|---|---|---|
| Bluetooth | `nrf/subsys/bluetooth/` | BLE Mesh, Direction Finding, Fast Pair |
| DFU | `nrf/subsys/dfu/` | MCUboot + SMP OTA 升级 |
| Networking | `nrf/subsys/net/` | OpenThread, Wi-Fi, L2 连接 |
| CAF | `nrf/subsys/caf/` | Common Application Framework (按键/LED/传感器管理) |
| Event Manager | `nrf/subsys/app_event_manager/` | 应用事件发布/订阅框架 |
| Edge Impulse | `nrf/lib/edge_impulse/` | 端侧 ML 推理集成 |

### 完整应用示例 (applications/)
- `nrf_desktop` — HID 外设 (鼠标/键盘)
- `nrf5340_audio` — LE Audio 演示
- `matter_weather_station` — Matter 智能家居
- `serial_lte_modem` — AT 指令透传网关
- `machine_learning` — 端侧机器学习

### nrfxlib 闭源组件
Nordic 提供预编译的二进制库，位于 `nrfxlib/`:
- `nrf_modem` — LTE 调制解调器 AT/套接字接口
- `nrf_802154` — IEEE 802.15.4 射频驱动
- `mpsl` — 多协议调度 (BLE + Thread + Zigbee 共存)
- `nrf_wifi` — Wi-Fi 驱动
- `lc3` — LC3 音频编解码器

## 重要代码片段

### west.yml manifest 核心结构
```yaml
manifest:
  version: "0.13"
  remotes:
    - name: ncs
      url-base: https://github.com/nrfconnect
    - name: zephyrproject
      url-base: https://github.com/zephyrproject-rtos
  defaults:
    remote: ncs
  projects:
    - name: zephyr
      repo-path: sdk-zephyr
      revision: ncs-v3.1.1
      import: true
```

### NCS CMake 构建入口
```cmake
# nrf/CMakeLists.txt
set(NRF_DIR ${CMAKE_CURRENT_LIST_DIR} CACHE PATH "NCS root directory")
include(cmake/extensions.cmake)
include(cmake/version.cmake)
include(cmake/version_app.cmake)
zephyr_include_directories(include)
```

### Kconfig 配置要点
```
# nrf/Kconfig.nrf
config ROM_START_OFFSET
    default 0 if PARTITION_MANAGER_ENABLED

config WARN_EXPERIMENTAL
    default y
```

## 构建/运行方法

### 环境准备
```bash
# 1. 安装 west
pip install west

# 2. 初始化 workspace (从 manifest 拉取)
west init -m https://github.com/nrfconnect/sdk-nrf --mr v3.1.1
west update

# 3. 安装工具链 (Windows 已下载至 downloads/ 目录)
# 或使用 nrf-util:
# nrfutil toolchain-manager install --ncs-version v3.1.1
```

### 编译示例
```bash
# 编译 BLE 外设示例 (nRF52840 DK)
west build -b nrf52840dk/nrf52840 nrf/samples/bluetooth/peripheral_hr

# 编译 Matter 应用 (nRF5340 DK, 含 TF-M)
west build -b nrf5340dk/nrf5340/cpuapp nrf/applications/matter_weather_station

# 烧录
west flash

# 清理
west build -t pristine
```

### 常用 west 命令
```bash
west list                  # 列出所有仓库及版本
west update                # 同步所有仓库到 manifest 版本
west boards                # 列出所有支持的开发板
west build -t menuconfig   # Kconfig 图形配置
west build -t guiconfig    # Kconfig GUI 配置
```

## 相关笔记链接

- [[Zephyr RTOS]]
- [[nRF5340 双核架构]]
- [[BLE 蓝牙协议栈]]
- [[Matter 协议入门]]
- [[MCUboot 安全启动]]
- [[CMake 构建系统]]
- [[Kconfig 配置系统]]
- [[Devicetree 设备树]]
- [[nRF9160 LTE开发]]
- [[nRF Connect for VS Code]]
```

## 相关笔记

- [[zephyr]] — Zephyr RTOS 项目笔记
- [[studyzephyr]] — Zephyr RTOS 学习项目
- [[nrf-project]] — Nordic Zephyr 应用集合
- [[zephyr-nxp-notes]] — NXP Zephyr 开发笔记
