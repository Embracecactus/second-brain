---
tags: [learning, notes, esp32, nrf, zephyr]
category: learning
created: 2026-06-09
---
Now I have a thorough understanding of the entire project. Let me generate the Obsidian note.

---
---
tags:
  - embedded
  - learning-notes
  - MCU
  - FPGA
  - linux
  - zephyr
  - esp-idf
  - docker
  - h618
  - arm64
category: learning
created: 2026-06-09
status: active
---

# 嵌入式开发学习笔记合集 (wenan)

## 项目概述

这是一个嵌入式系统开发的综合学习笔记仓库，涵盖了从 8-bit/32-bit MCU 开发、FPGA 入门、到 ARM64 Linux 系统移植与应用开发的全流程实践经验。笔记以具体硬件平台为线索，记录了开发环境搭建、编译烧录、驱动调试、系统移植等关键环节的操作步骤和踩坑记录。

## 技术栈

### 硬件平台
- **ESP32-S3 / ESP32-C3** -- Espressif WiFi/BLE SoC
- **CH32V208** -- WCH RISC-V MCU
- **NRF7002DK (nRF5340)** -- Nordic WiFi + BLE 开发板
- **FRDM-MCXA346** -- NXP Cortex-M33 MCU
- **H618 AI TV Box** -- Allwinner H618 ARM64 盒子
- **M5StickC-Plus** -- M5Stack 语音识别模块
- **FPGA (Xilinx xc7a35t)** -- 入门级 FPGA 开发板

### 软件框架与工具
- **Zephyr RTOS** -- 跨平台 RTOS，支持 NRF、NXP 等多芯片
- **ESP-IDF** -- Espressif 官方 IoT 开发框架
- **Vivado** -- Xilinx FPGA 开发工具
- **Docker** -- 统一编译环境容器化
- **CMake / West** -- 构建系统
- **LVGL** -- 嵌入式 GUI 库
- **uniapp + Vue 3** -- 跨平台 App 开发（智能灯光控制）
- **OpenClaw** -- AI Agent 框架，接入飞书
- **U-Boot / Linux Kernel / genimage** -- Linux 系统移植工具链

### 语言
- C / C++ / Verilog / Python / CMake / Bash

## 架构与关键设计决策

### 1. Docker 容器化编译环境
所有 MCU 开发均采用 Docker 镜像封装编译工具链，确保环境一致性：
- Zephyr 使用 `zephyr-ci` 自定义镜像（基于官方镜像定制，替换清华源加速）
- ESP-IDF 使用 `espressif/idf` 官方镜像
- 镜像推送到阿里云 ACR 个人版仓库，实现跨机器复用

### 2. Zephyr RTOS 多平台统一开发
采用 Zephyr 的 `west` 工具管理项目，统一 NRF 和 NXP 平台的开发流程：
- 通过 `west init` + `west update` 精确拉取所需模块（非全量拉取）
- 使用 Device Tree Overlay (`app.overlay`) 扩展硬件配置
- 通过 `prj.conf` 管理 Kconfig 配置项

### 3. H618 Linux 系统全栈移植
从零构建完整 Linux 系统，分为四个阶段：
- **Stage 0**: ARM Trusted Firmware (BL31)
- **Stage 1**: U-Boot 移植（自定义 defconfig + DTS）
- **Stage 2**: Linux Kernel 移植（打 Armbian patch，自定义 defconfig + DTS）
- **Stage 3**: Rootfs 制作（基于 Ubuntu 22.04 ARM64）
- **Stage 4**: 镜像打包（genimage 工具）

### 4. 设备树 (Device Tree) 驱动架构
在 NXP MCXA346 平台深入分析了 Zephyr 的 DTS 驱动机制：
- DTS 解析 -> YAML binding 匹配 -> 代码生成 (`devicetree_generated.h`)
- 通过 `DT_CHOSEN(zephyr_console)` 宏在驱动中获取设备节点
- 启动流程: `reset.S` -> `z_prep_c()` -> `z_cstart()` -> `board_early_init_hook()`

## 关键学习与踩坑记录

### Zephyr 平台
- **堆栈溢出问题**: LVGL 运行时 BUS FAULT，通过 `addr2line` 定位到 `lv_anim.c`，增大 `CONFIG_MAIN_STACK_SIZE=8192` 解决
- **USAGE FAULT (Illegal load of EXC_RETURN)**: idle 线程栈溢出，需进一步增大系统栈
- **SmartDMA 设备树问题**: SDK 中缺少 SmartDMA 节点，需在 `app.overlay` 根节点手动添加，并添加 `interrupt-parent = <&nvic>` 属性
- **P1_29 引脚冲突**: 该引脚连接了 RESET/SW1 按键，使用后会导致 MCULINK 下载需按 ISP/SW2 按键上电
- **摄像头集成**: 需修改 `drivers.cmake` 添加 smartdma_mcxa 驱动，修改 `board.c` 添加时钟使能，修改 `fsl_inputmux_connections.h` 添加宏定义

### ESP-IDF 平台
- **IRAM 不足**: WiFi + BLE + ASR 组合工程出现 `.iram0.text` 溢出，通过 `idf.py menuconfig` 取消 WiFi IRAM 缓存选项解决
- **NFC ST25DV**: I2C 速率过低会导致传输卡住在 200 字节处
- **栈溢出**: `main` 函数中定义 5000 字节局部变量导致 `app_main` 栈溢出，应用 `idf.py size` 检查静态分配
- **FreeRTOS 事件组**: `xTaskCreate` 创建的任务函数必须包含 `while(1)`，否则程序崩溃
- **sdkconfig 管理**: 每次重新配置需删除 `sdkconfig` 和 `build` 目录，使用 `idf.py save-defconfig` 保存

### H618 Linux 平台
- **桌面环境休眠**: 安装 XFCE4 后系统会休眠导致 SSH/串口断连，需禁用 systemd 休眠目标
- **Netplan 配置**: Ubuntu 22.04 网卡命名从 `eth0` 变为 `end0`，需修改 netplan 配置
- **Docker 安装**: 需开启 `CONFIG_IP_NF_*`、`CONFIG_EXT4_FS_*` 等内核特性
- **eMMC 启动**: 需使用 PARTUUID 自适应启动脚本

## 代码片段

### Zephyr Shell 控制 RGB LED
```c
// 通过 Device Tree 获取 LED 设备规格
const struct led_dt_spec spec_red = LED_DT_SPEC_GET(DT_NODELABEL(red_led));

// Shell 命令注册
SHELL_CMD_REGISTER(rgb_led, &led_cmds, "Control RGB LED", NULL);
```

### FPGA Verilog LED 闪烁 (计数器分频)
```verilog
reg [24:0] counter;
always @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
        counter <= 25'd0;
        led <= 1'b0;
    end else begin
        counter <= counter + 1'b1;
        if (counter == 25'd12_500_000) begin
            counter <= 25'd0;
            led <= ~led;
        end
    end
end
```

### Zephyr 启动流程关键路径
```
reset.S: bl z_prep_c
  -> z_prep_c(): BSS清零, 数据拷贝, 中断初始化
    -> z_cstart(): z_sys_init_run_level(EARLY), board_early_init_hook()
```

### ESP-IDF ADC 按键配置
```c
button_adc_config_t btn_adc_cfg = {
    .unit_id = ADC_UNIT_1,
    .adc_channel = CONFIG_ADC_BUTTON_CHANNEL,
    .button_index = 0,
    .min = 100,
    .max = 400,
};
iot_button_register_cb(adc_btn, BUTTON_LONG_PRESS_START, &args, callback, NULL);
```

## 构建与运行指南

### Zephyr (NXP FRDM-MCXA346)
```bash
# 启动 Docker 编译环境
docker run -ti -v $PWD:/workdir zephyr-ci:v1.0
cd workdir && west init . && west update hal_nxp cmsis cmsis_6
# 编译
cd frdm-mcxa346 && west build -b frdm_mcxa346
```

### Zephyr (NRF7002DK)
```bash
west build -b nrf7002dk/nrf5340/cpuapp   # CPUAPP
west build -b nrf7002dk/nrf5340/cpunet   # CPUNET
# 烧录
nrfjprog --program build/zephyr/zephyr.hex --chiperase --verify --reset
```

### ESP-IDF (ESP32-S3)
```bash
docker run -it -v $PWD:/project docker.1ms.run/espressif/idf:latest
idf.py create-project myproject && cd myproject
idf.py set-target esp32s3 && idf.py build
# 烧录
python -m esptool --chip esp32s3 write_flash 0x0 build/myproject.bin
```

### H618 Linux 系统构建
```bash
# 0. ATF: make CROSS_COMPILE=aarch64-none-linux-gnu- PLAT=sun50i_h616 DEBUG=1 bl31
# 1. U-Boot: make ARCH=arm CROSS_COMPILE=aarch64-none-linux-gnu- sun50i-h618-ai-tv_defconfig && make -j32
# 2. Kernel: make ARCH=arm64 CROSS_COMPILE=aarch64-none-linux-gnu- sun50i-h618-ai-tv_defconfig && make -j32
# 3. Rootfs: cd 3.rootfs && make all
# 4. Pack: cd pack && genimage --input input --output images --config root/genimage.cfg
```

## 项目文件结构

```
wenan/
├── FPGA/
│   ├── README.md                    # FPGA 基础介绍
│   └── FPGA口播视频脚本.md          # FPGA 入门教程视频脚本
├── 学习笔记/
│   ├── ch32v208.md                  # CH32V208 RISC-V MCU 开发
│   ├── esp32-README.md              # ESP32-S3 ST25DV NFC 开发
│   ├── esp32bikelight.md            # ESP32-C3 车灯项目 + 深度学习边缘计算
│   ├── esp32st25dv.md               # ESP32-S3 ST25DV NFC 详细笔记
│   ├── h618-system-README.md        # H618 Linux 系统移植全记录
│   ├── h618-app-README.md           # H618 应用开发(Docker/桌面/VNC/摄像头)
│   ├── h618-web-readme.md           # H618 轻量化浏览器
│   ├── h618-retroarch-docker.md     # H618 RetroArch 模拟器 Docker
│   ├── h618-RetroPie-setup.md       # H618 RetroPie 设置
│   ├── m5stick.md                   # M5StickC-Plus 语音识别
│   ├── nrf-7002dk.md                # NRF7002DK Zephyr 开发
│   ├── nxp-zephyrREADME.md          # NXP MCXA346 Zephyr 深度学习笔记
│   ├── rk3528-system.md             # RK3528 系统
│   ├── 推送镜像到阿里云README.md    # Docker 镜像推送阿里云 ACR
│   └── 车灯app.md                   # 智能灯光控制系统 uniapp 应用
├── h618-tv-box-install-openclaw.md  # H618 安装 OpenClaw AI Agent
└── 20251210.txt
```

## 相关概念链接

- [[Zephyr RTOS]] -- 跨平台实时操作系统
- [[Device Tree]] -- 嵌入式 Linux/Zephyr 设备描述机制
- [[ESP-IDF]] -- Espressif IoT 开发框架
- [[Docker 容器化开发]] -- 统一编译环境
- [[ARM64 Linux 移植]] -- 从 U-Boot 到 Rootfs 的全流程
- [[FPGA 入门]] -- Verilog HDL 开发基础
- [[LVGL]] -- 嵌入式图形库
- [[FreeRTOS]] -- ESP-IDF 底层 RTOS
- [[OpenClaw]] -- AI Agent 框架
- [[NFC ST25DV]] -- I2C NFC 标签芯片
- [[Docker 镜像加速]] -- 国内镜像源配置
- [[阿里云 ACR]] -- 容器镜像托管服务

## 相关笔记

- [[zephyr]] — Zephyr RTOS 项目笔记
- [[studyzephyr]] — Zephyr RTOS 学习项目
- [[esp32-box-lite]] — ESP32-S3-BOX-Lite 开发板
- [[esp32s3-nfc]] — ESP32-S3 + ST25DV NFC
- [[esp32c3]] — ESP32-C3 智能尾灯项目
- [[h618-buildroot]] — H618 完整开发笔记
- [[fpga]] — FPGA LED 流水灯项目
- [[xmind-notes]] — XMind 知识库概览
- [[wch]] — WCH CH32V RISC-V MCU 项目
- [[ncs]] — nRF Connect SDK
- [[frdm-mcxa346]] — FRDM-MCXA346 开发板
