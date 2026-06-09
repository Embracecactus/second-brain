---
title: TuyaOpen - 涂鸦 AI+IoT 开发框架
category: iot/tuya
tags:
  - tuya
  - iot
  - ai
  - embedded
  - rtos
  - lvgl
  - cmake
  - esp32
  - smart-home
created: 2026-06-09
status: active
source: https://github.com/tuya/tuyaopen
license: Apache-2.0
---

# TuyaOpen - 涂鸦 AI+IoT 开发框架

## 项目概述

TuyaOpen 是涂鸦智能开源的 AI+IoT 开发框架，支持多芯片平台和类 RTOS 操作系统，能够无缝集成多模态 AI 能力（音频、视频、传感器数据处理），帮助开发者快速创建智能互联设备。框架提供从底层驱动到云端服务的完整开发栈，支持语音识别（ASR/KWS/TTS/STT）、主流 LLM 集成（Deepseek/ChatGPT/Claude/Gemini）以及 Google Home / Amazon Alexa 生态兼容。

## 技术栈

- **编程语言**: C (核心 SDK), Python (构建工具链 / CLI), C++ (可选)
- **构建系统**: CMake 3.16+, Kconfiglib (配置系统), Ninja
- **GUI 框架**: LVGL (Light and Versatile Graphics Library)
- **网络协议**: MQTT, HTTP/HTTPS, TLS, LwIP, Bluetooth, Wi-Fi, Ethernet
- **AI/语音**: ASR, KWS, TTS, STT, LLM 集成
- **云服务**: 涂鸦云 (Tuya Cloud) - 远程控制 / 监控 / OTA 升级
- **依赖管理**: Python venv + pip, Git submodules
- **代码规范**: clang-format
- **容器化**: Docker (Ubuntu 基础镜像)

## 支持的硬件平台

| 平台 | 说明 | 调试串口 |
|------|------|----------|
| Ubuntu | 可直接在 Linux 主机运行 | - |
| Tuya T2 | T2-U 模块 | Uart2/115200 |
| Tuya T3 | T3-U / T3-2S / T3-3S / T3-E2 等 | Uart1/460800 |
| Tuya T5 | T5-E1 / T5-E1-IPEX 等 (AI 主力平台) | Uart1/460800 |
| ESP32/ESP32C3/ESP32S3 | 乐鑫系列 | Uart0/115200 |
| LN882H | 凌通系列 | Uart1/921600 |
| BK7231N | 博通集成系列 (CBU/CB3S/CB3L 等) | Uart2/115200 |

各平台以独立 Git 仓库维护，通过 `platform_config.yaml` 声明仓库地址和分支。

## 架构与设计决策

### SDK 分层架构

```
+------------------------------------------+
|           Application Layer              |
|  (examples/, apps/, user projects)       |
+------------------------------------------+
|         TuyaOpen Core Components         |
|  tuya_cloud_service | tuya_ai_basic      |
|  tuya_p2p | tal_* | lib* | peripherals  |
+------------------------------------------+
|       Platform Abstraction Layer (PAL)   |
|  platform/T2 | T3 | T5AI | ESP32 | ...  |
+------------------------------------------+
|          Hardware / RTOS                 |
+------------------------------------------+
```

### 核心组件 (`src/` 目录)

- **tuya_cloud_service** - 涂鸦云服务对接层
- **tuya_ai_basic** - AI 基础能力封装 (语音/LLM)
- **tuya_p2p** - P2P 通信能力
- **tal_*** - Tuya Abstraction Layer (抽象层):
  - `tal_wifi` / `tal_bluetooth` / `tal_cellular` - 无线连接
  - `tal_driver` - 驱动抽象
  - `tal_system` - 系统服务
  - `tal_network` / `tal_security` - 网络与安全
  - `tal_kv` - Key-Value 存储
  - `tal_cli` - 命令行接口
- **lib*** - 协议与库:
  - `liblvgl` - LVGL 图形库
  - `libmqtt` - MQTT 客户端
  - `libhttp` - HTTP 客户端
  - `libtls` - TLS 安全通信
  - `liblwip` - LwIP TCP/IP 协议栈
  - `libcjson` - JSON 解析
- **peripherals** - 外设驱动
- **common** - 公共工具与基础组件
- **micropython** - MicroPython 支持

### 关键设计决策

1. **Kconfig 配置系统**: 使用 Kconfiglib 实现内核级配置管理，支持 `menuconfig` 交互式配置，生成 `tuya_kconfig.h` 宏定义头文件。配置入口在 `src/Kconfig`，各子模块独立维护 Kconfig。

2. **平台抽象层 (PAL)**: 通过 `platform/` 目录隔离硬件差异，每个平台提供 `toolchain_file.cmake` 和 `platform_config.cmake`，实现跨平台编译。

3. **组件化构建**: CMake 自动扫描 `src/` 下的组件目录，每个组件编译为 OBJECT 库，最终链接为统一的 `tuyaos` 静态库。

4. **Python CLI 工具链 (`tos.py`)**: 基于 Click 框架的统一命令行工具，提供 `build` / `flash` / `monitor` / `config` / `new` 等子命令，替代传统 Makefile 工作流。

5. **虚拟环境隔离**: `export.sh` 自动创建 Python venv 并激活，确保构建依赖的版本一致性。

6. **多框架支持**: 除原生 C 开发外，还支持 Arduino (`arduino-TuyaOpen`) 和 Luanode (`luanode-TuyaOpen`) 开发模式，通过 `TOS_FRAMEWORK` 变量切换。

## 构建与运行

### 环境搭建

```bash
# 1. 克隆仓库 (含子模块)
git clone --recursive https://github.com/tuya/tuyaopen.git
cd TuyaOpen

# 2. 激活开发环境 (自动创建 venv, 安装依赖)
source ./export.sh

# 3. 检查环境
tos.py check
```

### 项目构建流程

```bash
# 创建新项目
tos.py new

# 配置目标平台和功能
tos.py config

# 编译
tos.py build

# 烧录
tos.py flash

# 串口监控
tos.py monitor

# 清理
tos.py clean
```

### Docker 构建

```bash
docker build -t tuyaopen .
docker run -it tuyaopen /bin/bash
```

### 项目结构示例 (T5-ai-project)

```
T5-ai-project/
  CMakeLists.txt          # 项目 CMake 配置
  app_default.config      # Kconfig 默认配置
  src/
    t5-ai-project.c       # 主程序入口 (user_main)
    ui/                   # LVGL UI 代码
      lvlg_t5ai.c
      screens/
```

## 代码模式

### 应用入口点模式

框架通过条件编译区分 Linux 和 RTOS 平台的入口：

```c
// Linux 平台: 直接调用 user_main
#if OPERATING_SYSTEM == SYSTEM_LINUX
void main(int argc, char *argv[]) {
    user_main();
    while (1) { tal_system_sleep(500); }
}

// RTOS 平台: 创建独立任务线程
#else
static void tuya_app_thread(void *arg) {
    user_main();
    tal_thread_delete(ty_app_thread);
}

void tuya_app_main(void) {
    THREAD_CFG_T thrd_param = {1024 * 4, 4, "tuya_app_main"};
    tal_thread_create_and_start(&ty_app_thread, NULL, NULL,
                                 tuya_app_thread, NULL, &thrd_param);
}
#endif
```

### LVGL 初始化模式

```c
void user_main(void) {
    tal_log_init(TAL_LOG_LEVEL_INFO, 4096, (TAL_LOG_OUTPUT_CB)tkl_log_output);
    board_register_hardware();
    lv_vendor_init(DISPLAY_NAME);
    lv_vendor_disp_lock();
    lvlg_t5ai_init(NULL);          // 自定义 UI 初始化
    lv_vendor_disp_unlock();
    lv_vendor_start(5, 1024*8);    // 启动 LVGL 任务 (优先级5, 8KB栈)
}
```

### Kconfig 配置示例 (T5 AI Board + LVGL)

```kconfig
CONFIG_BOARD_CHOICE_T5AI=y
CONFIG_TUYA_T5AI_BOARD_EX_MODULE_35565LCD=y
CONFIG_ENABLE_LIBLVGL=y
CONFIG_LVGL_ENABLE_TOUCH=y
CONFIG_ENABLE_LVGL_DEMO=y
CONFIG_LV_USE_DEMO_MUSIC=y
```

## 关键学习与洞察

1. **子模块依赖风险**: 框架深度依赖 Git submodules 和外部平台仓库，各平台版本通过 `platform_config.yaml` 中的 commit hash 锁定。更新子模块时需注意兼容性。

2. **Kconfig 驱动的功能开关**: 所有可选功能（LVGL / 蓝牙 / MQTT / MicroPython 等）均通过 Kconfig 控制编译，`app_default.config` 定义项目级默认配置。

3. **组件自动发现机制**: CMake 通过 `list_components()` 函数自动扫描 `src/` 目录，新增组件只需在对应目录添加 `CMakeLists.txt` 和 `Kconfig`。

4. **平台仓库独立演进**: 各芯片平台（T2/T3/T5AI/ESP32/BK7231X 等）作为独立仓库维护，主仓库通过 `tos.py update` 命令同步。

5. **安全内置**: 框架内置设备认证、数据加密、TLS 通信等安全能力，通过 `tal_security` 和 `libtls` 组件提供。

## 相关资源

- [TuyaOpen 开发者文档](https://tuyaopen.ai/docs/about-tuyaopen)
- [快速开始 - 环境搭建](https://tuyaopen.ai/docs/quick_start/enviroment-setup)
- [T5 AI 开发板](https://tuyaopen.ai/docs/hardware-specific/t5-ai-board/overview-t5-ai-board)
- [涂鸦 AI Agent 管理](https://developer.tuya.com/en/docs/iot/ai-agent-management?id=Kdxr4v7uv4fud)
- [Arduino for TuyaOpen](https://github.com/tuya/arduino-TuyaOpen)
- [Luanode for TuyaOpen](https://github.com/tuya/luanode-TuyaOpen)
- [GitHub 仓库](https://github.com/tuya/tuyaopen)
- [贡献指南](https://tuyaopen.ai/docs/contribute/contribute-guide)

## 相关笔记

- [[esp-idf-v5-guide]] — ESP-IDF v5 开发指南（ESP32 平台）
- [[esp32c3]] — ESP32-C3 智能尾灯项目
- [[zephyr]] — Zephyr RTOS 项目笔记（RTOS 对比）
- [[lvgl]] — LVGL T5AI 项目（LVGL 集成）
- [[xingshan]] — BS2X CFBB 嵌入式项目（IoT 平台对比）
- [[redbook]] — 小红书技术内容创作素材库
