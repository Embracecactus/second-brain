---
tags:
  - cnc-cam
  - 3d-graphics
  - embedded-gui
  - gcode
  - terminal-rendering
  - c-project
category: cnc-cam
created: 2026-06-09
status: draft
project: 3dgui
path: /home/lijian/project/3dgui
---

# 3dgui - 终端 3D G-Code 可视化渲染器

## 项目概述

一个基于终端(ncurses)的 3D 图形渲染工具，用于在 128x128 位图缓冲区中实现立方体旋转、32x32 图标透视投影等 3D 变换，并通过 ASCII 字符在终端中实时显示。项目名为 `3d_gcode_show_pzx_v1`，属于 CNC/CAM 领域的 G-Code 可视化辅助工具。

## 技术栈

| 类别 | 技术 |
|------|------|
| 语言 | C/C++ (混合编译) |
| 终端库 | ncurses |
| 数学库 | math (libm) |
| 线程库 | pthread |
| 构建 | GCC + bash shell script |
| 目标平台 | Linux (原为 Windows，已移植到 ANSI 终端) |

## 架构与关键设计

### 文件结构

```
3d_gcode_show_pzx_v1/
├── main.cpp          # 主程序：3D 变换引擎 + 渲染循环
├── font.c            # 字体与图标点阵数据 (ASCII 6x8/8x16, 32x32 图标)
├── font.h            # 字体/图标声明
├── shuju.c           # 图片数据(位图数组，含多个 64x64 帧)
├── build.sh          # 构建脚本
└── main              # 编译产物
```

### 核心架构

程序采用 **位图缓冲区 + 3D 变换管线 + 终端字符渲染** 的三层架构：

1. **位图缓冲层** (`tu[][]`)：128x128 位图，使用 bit 操作进行像素级控制
2. **3D 变换管线**：通过 4x4 矩阵实现平移、缩放、旋转变换，支持正射/透视投影
3. **终端渲染层**：将位图转为 ASCII 字符 (`*` 和空格)，通过 ANSI 转义码定位光标刷新显示

## 核心知识点

### 1. 位操作与位图缓冲

使用 `char bits[]` 作为 128x128 位图缓冲区，通过位移和掩码操作实现单像素控制：

```c
#define SHIFT 3
#define MASK 7

void set_bit(char bit_array[], unsigned int bit_number) {
    bit_array[bit_number >> SHIFT] |= 1 << (7 - bit_number & MASK);
}
```

`bit12864()` 函数封装了像素设置/清除/翻转三种操作模式。

### 2. 4x4 变换矩阵

使用齐次坐标的 4x4 矩阵实现 3D 仿射变换：

- **Identity_3D()**：单位矩阵初始化
- **Translate_3D()**：平移变换 (tx, ty, tz)
- **Scale_3D()**：比例变换 (sx, sy, sz)
- **Rotate_3D()**：绕 X/Y/Z 轴旋转变换，内部将角度转为弧度后分别构建旋转矩阵并级联相乘

变换顺序：`Identity -> Translate(居中) -> Scale(放大) -> Rotate(旋转) -> Translate(移到视区中心)`

### 3. 投影方式

- **正射投影 (OrtProject)**：直接丢弃 Z 坐标
- **透视投影 (PerProject)**：基于焦距 `FOCAL_DISTANCE=128` 的中心投影法

```c
Screen.x = (int)(FOCAL_DISTANCE * Space.x / (Space.z + FOCAL_DISTANCE)) + XOrigin;
```

### 4. Bresenham 画线算法

`DrawLine()` 实现了完整的 Bresenham 直线绘制算法，支持任意斜率，并针对水平/垂直线做了快速路径优化。

### 5. G-Code 点阵数据

`gcode_point[500]` 数组预留了 500 个 3D 点用于存储 G-Code 路径坐标，可在 `RotateCube2()` 中通过 `#if 0` 切换为 G-Code 路径渲染模式。

## 重要代码片段

### 主渲染循环

```c
while(1) {
    RotatePic32x32(SETICO[4], turn, turn, turn);  // 旋转 32x32 图标
    genxin(k, tu);       // 位图转字符
    printf(bits);         // 输出到终端
    goto_xy(0,0);        // ANSI 光标归位
    usleep(20000);        // 20ms 延时 (约 50fps)
    memset(tu, 0x00, sizeof(tu));
    turn += 1;
}
```

### 终端光标控制 (从 Windows 移植到 Linux)

```c
void goto_xy(int x, int y) {
    printf("\033[%d;%dH", y + 1, x + 1);  // ANSI 转义码
    fflush(stdout);
}
```

## 构建/运行方法

```bash
# 构建
cd /home/lijian/project/3dgui/3d_gcode_show_pzx_v1/3d_gcode_show_pzx_v1/
bash build.sh

# 运行 (需要终端至少 128 列宽)
./main
```

构建命令：`gcc main.cpp -o main -lm -lpthread -lncurses`

## 相关笔记链接

- [[CNC-GCode]] - G-Code 基础知识
- [[嵌入式GUI-LCD12864]] - 12864 LCD 驱动与位图渲染
- [[3D图形变换矩阵]] - 齐次坐标与仿射变换
- [[Bresenham画线算法]] - 光栅化直线绘制
- [[ncurses终端编程]] - ncurses 库使用
- [[嵌入式点阵字体]] - 位图字体与图标数据格式

---

Note: The `shuju.c` file contains large bitmap data arrays (a 64x64 pixel icon with multiple animation frames encoded as hex arrays). The font files (`font.c`/`font.h`) provide ASCII 6x8 and 8x16 font tables plus 32x32 icon bitmaps (`SETICO[5][128]`). The `SETICO[4]` icon is the one rendered in the main animation loop.

## 相关笔记

- [[pocketcnc]] — PocketCNC Web 端五轴 CNC 仿真模拟器
- [[qt-linuxcnc]] — LinuxCNC CAM 五轴数控软件
- [[pocktcnc-win]] — PocketCNC Windows 版
- [[diaoke]] — CNC G-Code 雕刻查看器
