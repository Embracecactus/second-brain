---
tags:
  - lvgl
  - gui
  - embedded
category: mcu/common
created: 2026-06-09
tool: SquareLine Studio 1.5.1
lvgl_version: 8.3.11
project: SquareLine_Project
---

# LVGL SquareLine Studio UI 项目笔记

## 概述

本项目是一个由 **SquareLine Studio 1.5.1** 自动生成的 LVGL UI 工程，基于 **LVGL 8.3.11** 版本。项目名为 `SquareLine_Project`，采用标准的 SquareLine Studio 导出结构，包含单屏幕 (Screen1)，其中放置了一个 Image Button 和一个 Switch 控件。整个工程通过 CMake 构建系统组织，可直接集成到嵌入式项目中。

## 关键知识点

### 1. SquareLine Studio 工程结构

SquareLine Studio 导出的代码遵循固定目录结构：

```
LVGL/
├── CMakeLists.txt          # CMake 构建配置
├── filelist.txt            # 源文件清单
├── ui.c                    # UI 初始化入口
├── ui.h                    # UI 头文件，声明所有屏幕和控件
├── ui_events.h             # 事件回调声明（本项目为空）
├── ui_helpers.c            # SquareLine 提供的辅助函数实现
├── ui_helpers.h            # 辅助函数头文件
├── components/
│   └── ui_comp_hook.c      # 自定义组件钩子（本项目为空）
├── screens/
│   └── ui_Screen1.c        # Screen1 的布局定义
├── fonts/                  # 字体文件目录（当前为空）
└── UI/                     # UI 子目录副本
```

### 2. LVGL 版本兼容性要求

项目对 LVGL 编译配置有严格要求：

- `LV_COLOR_DEPTH` 必须为 **32bit**，否则编译报错
- `LV_COLOR_16_SWAP` 必须为 **0**

这是 SquareLine Studio 的默认渲染配置，与目标硬件的 framebuffer 格式需保持一致。

### 3. UI 初始化流程

`ui_init()` 是整个 UI 系统的入口函数，执行顺序为：

1. 获取默认 display 对象 (`lv_disp_get_default`)
2. 初始化默认主题 (Blue/Red 配色，非暗色模式)
3. 初始化 Screen1 (`ui_Screen1_screen_init`)
4. 创建 `initial_actions` 对象用于事件绑定
5. 加载 Screen1 为当前活动屏幕

### 4. 屏幕管理机制

SquareLine 使用延迟初始化 (lazy initialization) 模式管理屏幕：

```c
void _ui_screen_change(lv_obj_t ** target, lv_scr_load_anim_t fademode, int spd, int delay, void (*target_init)(void))
{
    if(*target == NULL)
        target_init();                          // 首次访问时才初始化
    lv_scr_load_anim(*target, fademode, spd, delay, false);
}
```

屏幕对象指针通过二级指针 (`lv_obj_t **`) 传递，支持运行时创建/销毁。

### 5. 动画系统

`ui_helpers` 提供了完整的动画回调框架，核心数据结构：

```c
typedef struct _ui_anim_user_data_t {
    lv_obj_t * target;          // 动画目标对象
    lv_img_dsc_t ** imgset;     // 图片帧集合（用于帧动画）
    int32_t imgset_size;        // 帧数量
    int32_t val;                // 当前帧索引
} ui_anim_user_data_t;
```

支持的动画属性包括：X/Y 位置、宽高、透明度 (opacity)、图片缩放 (zoom)、图片旋转 (angle)、帧动画 (image frame)。

## 技术细节

### Screen1 布局定义

```c
void ui_Screen1_screen_init(void)
{
    // 创建主屏幕，禁用滚动
    ui_Screen1 = lv_obj_create(NULL);
    lv_obj_clear_flag(ui_Screen1, LV_OBJ_FLAG_SCROLLABLE);

    // 创建图片按钮，居中对齐，尺寸自适应内容
    ui_ImgButton1 = lv_imgbtn_create(ui_Screen1);
    lv_obj_set_width(ui_ImgButton1, LV_SIZE_CONTENT);
    lv_obj_set_height(ui_ImgButton1, LV_SIZE_CONTENT);
    lv_obj_set_align(ui_ImgButton1, LV_ALIGN_CENTER);

    // 创建开关控件，50x25 像素，偏移到左上区域
    ui_Switch1 = lv_switch_create(ui_Screen1);
    lv_obj_set_width(ui_Switch1, 50);
    lv_obj_set_height(ui_Switch1, 25);
    lv_obj_set_x(ui_Switch1, -293);
    lv_obj_set_y(ui_Switch1, -176);
    lv_obj_set_align(ui_Switch1, LV_ALIGN_CENTER);
    lv_obj_set_flex_flow(ui_Switch1, LV_FLEX_FLOW_ROW);
}
```

### 辅助函数分类

| 类别 | 函数 | 用途 |
|------|------|------|
| **属性设置** | `_ui_bar_set_property` | Bar 控件值设置（支持动画） |
| | `_ui_basic_set_property` | 基础属性 (x/y/width/height) |
| | `_ui_dropdown_set_property` | Dropdown 选中项 |
| | `_ui_image_set_property` | Image 图片源 |
| | `_ui_label_set_property` | Label 文本 |
| | `_ui_roller_set_property` | Roller 选中项 |
| | `_ui_slider_set_property` | Slider 值 |
| **屏幕管理** | `_ui_screen_change` | 屏幕切换（带懒加载） |
| | `_ui_screen_delete` | 屏幕销毁 |
| **控件操作** | `_ui_arc_increment` | Arc 值增量 |
| | `_ui_bar_increment` | Bar 值增量 |
| | `_ui_slider_increment` | Slider 值增量 |
| | `_ui_keyboard_set_target` | 键盘绑定 Textarea |
| **状态管理** | `_ui_flag_modify` | 对象 Flag 增/删/切换 |
| | `_ui_state_modify` | 对象 State 增/删/切换 |
| **文本联动** | `_ui_arc_set_text_value` | Arc 值同步到 Label |
| | `_ui_slider_set_text_value` | Slider 值同步到 Label |
| | `_ui_checked_set_text_value` | Checked 状态同步文本 |

### Flex 布局

Switch 控件使用了 LVGL 的 Flex 布局系统：

```c
lv_obj_set_flex_flow(ui_Switch1, LV_FLEX_FLOW_ROW);           // 水平排列
lv_obj_set_flex_align(ui_Switch1, LV_FLEX_ALIGN_START,        // 主轴对齐
                               LV_FLEX_ALIGN_START,            // 交叉轴对齐
                               LV_FLEX_ALIGN_START);           // track 对齐
```

## 代码/配置片段

### CMakeLists.txt

```cmake
SET(SOURCES screens/ui_Screen1.c
    ui.c
    components/ui_comp_hook.c
    ui_helpers.c)

add_library(ui ${SOURCES})
```

所有 UI 源文件编译为静态库 `ui`，外部项目通过 `target_link_libraries` 引入即可。

### LVGL 颜色配置编译检查

```c
#if LV_COLOR_DEPTH != 32
    #error "LV_COLOR_DEPTH should be 32bit to match SquareLine Studio's settings"
#endif
#if LV_COLOR_16_SWAP != 0
    #error "LV_COLOR_16_SWAP should be 0 to match SquareLine Studio's settings"
#endif
```

### 屏幕懒加载模式

```c
void _ui_screen_change(lv_obj_t ** target, lv_scr_load_anim_t fademode, int spd, int delay, void (*target_init)(void))
{
    if(*target == NULL)
        target_init();
    lv_scr_load_anim(*target, fademode, spd, delay, false);
}
```

### 状态/Flag 通用修改器

```c
void _ui_flag_modify(lv_obj_t * target, int32_t flag, int value)
{
    if(value == _UI_MODIFY_FLAG_TOGGLE) {
        if(lv_obj_has_flag(target, flag)) lv_obj_clear_flag(target, flag);
        else lv_obj_add_flag(target, flag);
    }
    else if(value == _UI_MODIFY_FLAG_ADD) lv_obj_add_flag(target, flag);
    else lv_obj_clear_flag(target, flag);
}
```

## 相关链接

- [LVGL 官方文档](https://docs.lvgl.io/8.3/)
- [SquareLine Studio 官网](https://squareline.io/)
- [LVGL GitHub 仓库 (v8.3)](https://github.com/lvgl/lvgl/tree/release/v8.3)
- [LVGL Flex 布局文档](https://docs.lvgl.io/8.3/layouts/flex.html)
- [LVGL 动画系统文档](https://docs.lvgl.io/8.3/overview/animation.html)

## 相关笔记

- [[lvgl]] — LVGL T5AI 项目
- [[zephyr]] — Zephyr RTOS 项目笔记（LVGL 集成）
- [[ok1126b-sdk]] — OK1126B SDK（含 LVGL SquareLine Studio）
- [[brithday]] — ESP32 生日项目（LVGL 手表界面）
- [[TuyaOpen]] — 涂鸦框架（LVGL 集成）
