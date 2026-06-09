---
tags: [nxp, zephyr, frdm, rtos]
category: mcu/nxp
created: 2026-06-09
---

# NXP Zephyr RTOS 开发笔记

## 项目概述

基于 NXP FRDM 开发板 (FRDM-MCXA346) 进行 Zephyr RTOS 开发。目标板搭载 MCXA346 MCU (ARM Cortex-M33)，通过 Zephyr 的 west 构建系统完成固件编译与烧录。项目涵盖了 GPIO LED 控制、Shell 命令行、LVGL GUI、ST7735S 屏幕驱动、OV5640 摄像头接入等外设开发。

---

## 环境搭建

### 方式一：原生 WSL2 环境

```sh
# 安装系统依赖
sudo apt update && sudo apt upgrade
sudo apt install --no-install-recommends git cmake ninja-build gperf \
  ccache dfu-util device-tree-compiler wget \
  python3-dev python3-pip python3-setuptools python3-tk python3-wheel xz-utils file \
  make gcc gcc-multilib g++-multilib libsdl2-dev libmagic1

# 验证工具版本
cmake --version
python3 --version
dtc --version
```

### 方式二：Docker 镜像（推荐）

```sh
# 拉取官方 Zephyr CI 镜像
# 最新标签查看: https://github.com/zephyrproject-rtos/zephyr-build/pkgs/container/zephyr-build
docker pull ghcr.io/zephyrproject-rtos/zephyr-ci:v0.28.7

# 国内加速
docker pull docker.1ms.run/zephyrprojectrtos/zephyr-build:v0.28.7

# 启动容器，挂载工作目录
docker run -ti -v $PWD:/workdir docker.1ms.run/zephyrprojectrtos/zephyr-build:v0.28.7
```

### 自定义 Docker 镜像构建（含 SDK）

基于 `zephyr-sdk-0.17.4` 构建：

```sh
git clone https://github.com/zephyrproject-rtos/docker-image.git

# 第一步：构建基础镜像（修改 Dockerfile.base 换清华源）
docker build -f Dockerfile.base \
   --build-arg UID=$(id -u) --build-arg GID=$(id -g) \
   -t zephyr-ci-base:v1.0 .

# 第二步：构建含 SDK 的镜像
docker build -f Dockerfile.ci \
    --build-arg BASE_IMAGE=zephyr-ci-base:v1.0 \
    -t zephyr-ci:v1.0 .

# 进入镜像
docker run -ti -v $HOME/Work/zephyrproject:/workdir zephyr-ci:v1.0
```

Dockerfile.base 关键修改项：
- Ubuntu 镜像源换为清华源 `mirrors.tuna.tsinghua.edu.cn`
- pip 源换为 `pypi.tuna.tsinghua.edu.cn`
- GitHub raw 文件使用代理 `edgeone.gh-proxy.org`

### Python 虚拟环境 + West 安装

```sh
# 创建并激活虚拟环境
python3 -m venv ~/project/zephyrproject/.venv
source ~/project/zephyrproject/.venv/bin/activate

# 换 pip 源
pip config set global.index-url https://mirrors.tuna.tsinghua.edu.cn/pypi/web/simple

# 安装 west
pip install west
```

### Zephyr SDK 安装

```sh
cd ~/project/zephyrproject
west sdk install
# 或手动解压 SDK 并运行 setup.sh
tar -xvf zephyr-sdk-0.17.4_linux-x86_64.tar.xz
cd zephyr-sdk-0.17.4/
./setup.sh  # 选择 Install host tools [y], Register CMake package [y]

# 安装 udev 规则（允许普通用户烧录）
sudo cp zephyr-sdk-0.17.4/sysroots/x86_64-pokysdk-linux/usr/share/openocd/contrib/60-openocd.rules /etc/udev/rules.d
sudo udevadm control --reload
```

### Zephyr 源码初始化

```sh
west init ~/project/zephyrproject
cd ~/project/zephyrproject
west update hal_nxp cmsis cmsis_6   # 仅拉取必要的模块
west zephyr-export                   # 导出 CMake 包
west packages pip --install          # 安装 Python 依赖
```

---

## 关键配置

### 工程目录结构

```
frdm-mcxa346/
  ├── frdm_mcxa346.overlay   # Device Tree overlay
  ├── CMakeLists.txt
  ├── prj.conf               # Kconfig 项目配置
  ├── Kconfig                # 自定义 Kconfig 菜单
  └── src/
      ├── main.c
      └── app_shell.c        # Shell 命令扩展
```

### CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.20.0)
set(BOARD "frdm_mcxa346")

find_package(Zephyr REQUIRED HINTS $ENV{ZEPHYR_BASE})
project(frdm_mcxa346)

target_sources(app PRIVATE src/main.c
                    src/app_shell.c)
```

关键点：
- `set(BOARD ...)` 指定目标板
- `find_package(Zephyr ...)` 加载 Zephyr CMake 样板代码
- 所有用户源文件添加到 `app` target

### prj.conf

```ini
# 基础配置
CONFIG_CONSOLE=y
CONFIG_UART_CONSOLE=y
CONFIG_SERIAL=y
CONFIG_MAIN_STACK_SIZE=8192   # LVGL 等模块需要较大栈空间

# Shell
CONFIG_SHELL=y

# 显示
CONFIG_DISPLAY=y
CONFIG_REQUIRES_FLOAT_PRINTF=y

# 摄像头
CONFIG_DMA_MCUX_SMARTDMA=y
CONFIG_I2C=y
CONFIG_VIDEO=y
CONFIG_DEVICE_SHELL=y
CONFIG_LOG_MODE_DEFERRED=y
CONFIG_VIDEO_LOG_LEVEL_DBG=y
```

### Device Tree 基础

DTS 处理流程：
1. 解析板级和 SoC 级 DTS 文件
2. 应用设备树 overlay
3. 预处理并合并为统一设备树
4. 对每个节点查找匹配的 YAML 绑定文件（binding）
5. 生成 `devicetree_generated.h` 头文件

board 级 DTS 中的 chosen 节点：

```dts
chosen {
    zephyr,sram = &sram0;
    zephyr,flash = &flash;
    zephyr,flash-controller = &fmu;
    zephyr,code-partition = &slot0_partition;
    zephyr,console = &lpuart2;     // 指定 console UART
    zephyr,shell-uart = &lpuart2;  // 指定 shell UART
};
```

生成的头文件 (`devicetree_generated.h`) 中：

```c
#define DT_CHOSEN_zephyr_console        DT_N_S_soc_S_lpuart_400a1000
#define DT_CHOSEN_zephyr_console_EXISTS 1
```

驱动中通过 `DEVICE_DT_GET(DT_CHOSEN(zephyr_console))` 获取设备句柄。

### Device Tree overlay 示例 (ST7735S 屏幕 + LVGL Keypad + OV5640 摄像头)

```dts
#include <zephyr/dt-bindings/mipi_dbi/mipi_dbi.h>
#include <nxp/mcx/MCXA346VLQ-pinctrl.h>
#include <zephyr/dt-bindings/input/input-event-codes.h>
#include <zephyr/dt-bindings/lvgl/lvgl.h>
#include <zephyr/dt-bindings/video/video-interfaces.h>

/ {
    chosen {
        zephyr,display = &st7735r_128x128;
        zephyr,camera = &dvp_20pin_interface;
    };

    /* MIPI DBI-SPI 显示接口 */
    mipi_dbi {
        compatible = "zephyr,mipi-dbi-spi";
        spi-dev = <&lpspi1>;
        dc-gpios = <&gpio3 13 GPIO_ACTIVE_HIGH>;
        reset-gpios = <&gpio2 5 GPIO_ACTIVE_LOW>;
        write-only;

        st7735r_128x128: st7735r@0 {
            compatible = "sitronix,st7735r";
            mipi-max-frequency = <DT_FREQ_M(100)>;
            mipi-mode = "MIPI_DBI_MODE_SPI_4WIRE";
            width = <128>;
            height = <128>;
            x-offset = <2>;
            y-offset = <3>;
            madctl = <0xC8>;
            /* ... 其他寄存器配置 ... */
        };
    };

    /* LVGL Keypad 输入 */
    gpio_keys: gpio_keys {};
    lvgl_keypad_input {
        compatible = "zephyr,lvgl-keypad-input";
        input = <&gpio_keys>;
        input-codes = <INPUT_KEY_0 INPUT_KEY_1>;
        lvgl-codes = <LV_KEY_NEXT LV_KEY_ENTER>;
    };

    /* SmartDMA + OV5640 摄像头 */
    smartdma: smartdma@4000e000 {
        compatible = "nxp,smartdma";
        reg = <0x4000e000 0x1000>;
        interrupt-parent = <&nvic>;
        interrupts = <108 0>;
        program-mem = <0x4000000>;
        status = "okay";

        video_sdma: video-sdma {
            compatible = "nxp,video-smartdma";
            vsync-pin = <TRIG_IN5_P2_26>;
            hsync-pin = <TRIG_IN10_P3_26>;
            pclk-pin = <TRIG_IN4_P4_6>;
            /* ... endpoint 配置 ... */
        };
    };
};

/* SPI1 引脚配置 + DMA */
&lpspi1 {
    status = "okay";
    dmas = <&edma0 0 17>, <&edma0 1 18>;
    dma-names = "rx", "tx";
};

/* I2C2 - OV5640 */
&lpi2c2 {
    status = "okay";
    ov5640: ov5640@3c {
        compatible = "ovti,ov5640";
        reg = <0x3c>;
        reset-gpios = <&gpio1 18 GPIO_ACTIVE_LOW>;
        powerdown-gpios = <&gpio1 16 GPIO_ACTIVE_HIGH>;
    };
};
```

---

## 构建与烧录

### 基本构建命令

```sh
# 查看支持的开发板
west boards

# 构建 hello_world 示例
cd zephyr/
west build -b frdm_mcxa346 samples/hello_world

# 构建自定义应用
cd frdm-mcxa346/
west build                     # 使用 CMakeLists.txt 中指定的 BOARD
west build -p always           # pristine build（清除旧构建）

# QEMU 模拟运行
west build -b qemu_cortex_m3 samples/hello_world
west build -t run
```

### 烧录

使用 MCULink 烧录。注意：如果误用了 P1_29 引脚（该引脚连接 REST SW1 按键），则 MCULink 下载程序时必须按住 ISP SW2 按键上电才能成功。

---

## Zephyr 启动流程

```
reset.S (bl z_prep_c)
  └── pre_c.c :: z_prep_c()
        ├── soc_prep_hook()
        ├── relocate_vector_table()
        ├── arch_bss_zero() / arch_data_copy()
        ├── z_arm_interrupt_init()
        └── z_cstart()
              ├── z_sys_init_run_level(INIT_LEVEL_EARLY)
              ├── arch_kernel_init()
              ├── soc_early_init_hook()
              ├── board_early_init_hook()    // 板级初始化（时钟、外设使能）
              └── z_sys_init_run_level(INIT_LEVEL_PRE_KERNEL_1)
                    // 驱动初始化阶段，如 SYS_INIT 注册的 uart_console_init
```

---

## 代码片段

### Hello World (main.c)

```c
#include <stdio.h>
#include <zephyr/kernel.h>

int main(void)
{
    while (1) {
        printf("Hello World! %s\n", CONFIG_BOARD);
        k_sleep(K_MSEC(1000));
    }
    return 0;
}
```

### Shell 控制 RGB LED (app_shell.c)

```c
#include <zephyr/shell/shell.h>
#include <zephyr/drivers/led.h>
#include <string.h>

const struct led_dt_spec spec_red   = LED_DT_SPEC_GET(DT_NODELABEL(red_led));
const struct led_dt_spec spec_blue  = LED_DT_SPEC_GET(DT_NODELABEL(blue_led));
const struct led_dt_spec spec_green = LED_DT_SPEC_GET(DT_NODELABEL(green_led));

static int cmd_rgb_led_on(const struct shell *shell, size_t argc, char **argv)
{
    if (strcmp(argv[1], "red_led") == 0) {
        led_on(spec_red.dev, spec_red.index);
    } else if (strcmp(argv[1], "blue_led") == 0) {
        led_on(spec_blue.dev, spec_blue.index);
    } else if (strcmp(argv[1], "green_led") == 0) {
        led_on(spec_green.dev, spec_green.index);
    } else {
        shell_print(shell, "Invalid color!");
        return -1;
    }
    shell_print(shell, "Turning on RGB LED: %s", argv[1]);
    return 0;
}

// 注册 shell 子命令
SHELL_STATIC_SUBCMD_SET_CREATE(led_cmds,
    SHELL_CMD_ARG(on, NULL, "Turn on LED", cmd_rgb_led_on, 2, 0),
    SHELL_CMD_ARG(off, NULL, "Turn off LED", cmd_rgb_led_off, 2, 0),
    SHELL_SUBCMD_SET_END
);
SHELL_CMD_REGISTER(rgb_led, &led_cmds, "Control RGB LED", NULL);
```

### LVGL 输入设备注册

```c
// zephyr/modules/lvgl/input/lvgl_common_input.c
int lvgl_input_register_driver(lv_indev_type_t indev_type, const struct device *dev)
{
    struct lvgl_common_input_data *common_data = dev->data;
    common_data->indev = lv_indev_create();
    lv_indev_set_type(common_data->indev, indev_type);
    lv_indev_set_read_cb(common_data->indev, lvgl_input_read_cb);
    lv_indev_set_user_data(common_data->indev, (void *)dev);
    // ...
}
```

### Board 级初始化 (board.c)

```c
// board_early_init_hook 中检查 DTS 节点状态并使能时钟
#if DT_NODE_HAS_STATUS_OKAY(DT_NODELABEL(lpuart0))
    CLOCK_SetClockDiv(kCLOCK_DivLPUART0, 1u);
    CLOCK_AttachClk(kFRO_LF_DIV_to_LPUART0);
    RESET_ReleasePeripheralReset(kLPUART0_RST_SHIFT_RSTn);
#endif

// SmartDMA + 摄像头时钟配置
#if DT_NODE_HAS_STATUS_OKAY(DT_NODELABEL(smartdma))
    CLOCK_EnableClock(kCLOCK_Smartdma);
    RESET_PeripheralReset(kSMART_DMA_RST_SHIFT_RSTn);
    #if DT_NODE_HAS_STATUS_OKAY(DT_NODELABEL(video_sdma))
        CLOCK_AttachClk(kFRO12M_to_CLKOUT);
        CLOCK_SetClockDiv(kCLOCK_DivCLKOUT, 2U);  // 6MHz cam_mclk
    #endif
#endif
```

---

## 踩坑记录

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| BUS FAULT: Precise data bus error, BFAR 0xc | 空指针解引用 | 使用 `addr2line` 定位：`arm-zephyr-eabi-addr2line -e build/zephyr/zephyr.elf <PC地址>` |
| USAGE FAULT: Illegal load of EXC_RETURN into PC | 栈溢出（idle 线程） | `prj.conf` 中增大 `CONFIG_MAIN_STACK_SIZE=8192` |
| MCULink 无法烧录 | P1_29 引脚被 SW1 按键占用 | 避免使用 P1_29；或按住 ISP SW2 上电后烧录 |
| SmartDMA 设备树缺失 | SoC 级 DTS 未定义 smartdma 节点 | 在 app.overlay 根节点手动添加 smartdma 节点并设置 `interrupt-parent` |
| 摄像头固件内存不足 | SmartDMA 程序内存区与应用冲突 | 检查 linker 分配，调整 `program-mem` 地址 |
| Docker 构建超时 | GitHub 下载慢 | Dockerfile 中使用 `edgeone.gh-proxy.org` 代理 |
| Python 版本过低 | Zephyr 要求 Python 3.12+ | 通过 deadsnakes PPA 安装 python3.12，重建 venv |

---

## Debug 技巧

```sh
# 使用 addr2line 定位崩溃地址
/opt/toolchains/zephyr-sdk-0.17.4/arm-zephyr-eabi/bin/arm-zephyr-eabi-addr2line \
  -e build/zephyr/zephyr.elf 0x00032576
```

---

## 相关笔记

- [[NXP-MCXA346-外设驱动]]
- [[Zephyr-Device-Tree-详解]]
- [[LVGL-Zephyr-移植]]
- [[OV5640-摄像头调试]]
- [[ARM-Cortex-M-启动流程]]
- [[Docker-嵌入式开发环境]]
