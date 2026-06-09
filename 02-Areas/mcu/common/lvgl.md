---
tags:
  - lvgl
  - mcu
  - embedded-ui
  - t5ai
  - gui
  - xml-driven
category: mcu/common
created: 2026-06-09
status: draft
platform: T5 AI
display: 320x480
tool: LVGL Editor
---

# LVGL T5AI 项目

## 项目概述

基于 LVGL Editor 生成的 XML 驱动 GUI 项目，目标平台为 T5 AI MCU，分辨率 320x480。项目采用代码生成架构，Editor 输出 XML 布局描述和对应的 C 代码骨架，用户在生成代码的预留区域中添加自定义逻辑。

## 技术栈

- **GUI 框架**: LVGL（Light and Versatile Graphics Library）
- **UI 设计工具**: LVGL Editor（XML 可视化设计器）
- **构建系统**: CMake >= 3.10
- **编程语言**: C / C++（支持 extern "C"）
- **UI 描述语言**: XML（screen、component、widget、globals）
- **目标硬件**: T5 AI 系列 MCU
- **显示分辨率**: 320 x 480（project_local.xml）/ 320 x 240（project.xml）

## 项目结构

```
lvlg-t5ai/
├── CMakeLists.txt                  # 顶层构建文件，编译为 lib-ui 静态库
├── file_list_gen.cmake             # Editor 自动生成的源文件列表
├── project.xml                     # 项目配置（target 定义）
├── project_local.xml               # 本地项目配置（覆盖分辨率）
├── globals.xml                     # 全局资源定义（styles/fonts/images/subjects）
├── lvlg_t5ai.h / .c               # 用户层初始化入口（调用 _gen 版本）
├── lvlg_t5ai_gen.h / .c           # Editor 自动生成的初始化代码
├── screens/                        # 屏幕定义
│   ├── screen_hello_world1.xml     # XML 布局描述
│   └── screen_hello_world1_gen.c/h # 生成的 C 代码
├── components/                     # 可复用组件（XML <component>）
├── widgets/                        # 自定义控件（XML <widget>）
├── fonts/                          # TTF/WOFF 字体文件
└── images/                         # PNG 图片资源
```

## 架构与设计决策

### 1. 代码生成 + 手动扩展分离

核心模式是 `_gen` 后缀文件由 Editor 自动生成，用户不应直接修改；无 `_gen` 后缀的同名文件是用户的扩展入口。

```c
// lvlg_t5ai.c — 用户扩展层
void lvlg_t5ai_init(const char * asset_path)
{
    lvlg_t5ai_init_gen(asset_path);  // 先调用自动生成的初始化
    /* Add your own custom code here if needed */
}
```

这种设计保证 Editor 重新生成代码时不会覆盖用户的自定义逻辑。

### 2. XML 驱动 UI 定义

屏幕布局用 XML 声明式描述，Editor 将其转换为等价的 C API 调用：

```xml
<screen>
    <styles>
        <style name="style_main" bg_color="0x00655a" />
    </styles>
    <view>
        <style name="style_main" />
        <lv_button align="center" style_bg_color="0x111">
            <lv_label text="hello world" style_text_font="lv_font_montserrat_14" />
        </lv_button>
    </view>
</screen>
```

生成的 C 代码：

```c
lv_obj_t * screen_hello_world1_create(void)
{
    static lv_style_t style_main;
    static bool style_inited = false;
    if (!style_inited) {
        lv_style_init(&style_main);
        lv_style_set_bg_color(&style_main, lv_color_hex(0x00655a));
        style_inited = true;
    }

    lv_obj_t * lv_obj_0 = lv_obj_create(NULL);
    lv_obj_add_style(lv_obj_0, &style_main, 0);
    lv_obj_t * lv_button_0 = lv_button_create(lv_obj_0);
    lv_obj_set_align(lv_button_0, LV_ALIGN_CENTER);
    lv_obj_set_style_bg_color(lv_button_0, lv_color_hex3(0x111), 0);
    lv_obj_t * lv_label_0 = lv_label_create(lv_button_0);
    lv_label_set_text(lv_label_0, "hello world");
    lv_obj_set_style_text_font(lv_label_0, lv_font_montserrat_14, 0);
    lv_obj_set_name(lv_obj_0, "screen_hello_world1");
    return lv_obj_0;
}
```

### 3. 样式单例初始化

生成代码中使用 `static bool style_inited` 标志确保样式只初始化一次，这是一个值得关注的模式——LVGL 的 style 对象是全局共享的，不能重复 init。

### 4. 双重 XML 模式支持

代码中大量使用 `#if LV_USE_XML` 条件编译：
- `LV_USE_XML = 1`：运行时通过 `lv_xml_create()` 解析 XML 创建 UI（适合开发/预览）
- `LV_USE_XML = 0`：直接调用生成的 C 函数创建 UI（适合生产环境）

### 5. CMake 库化构建

项目将所有 UI 源码编译为 `lib-ui` 静态库，可作为子模块集成到更大的工程中。CMakeLists.txt 检测 `IS_TOP_LEVEL` 和 `LV_EDITOR_PREVIEW` 来区分编辑器预览和实际目标构建。

## 关键学习与洞察

- **LVGL Editor 工作流**: 设计师在 Editor 中拖拽控件 -> 导出 XML + C 代码 -> 开发者在非 _gen 文件中添加业务逻辑 -> 构建刷机
- **globals.xml 是全局资源注册中心**: 样式、字体、图片、Subject 等全局资源在此统一声明，Editor 生成代码时会自动引用
- **project.xml vs project_local.xml**: `project.xml` 是共享配置，`project_local.xml` 是本地覆盖（如不同开发者的显示器分辨率不同），后者应加入 .gitignore
- **Subject 机制**: LVGL 的 Subject 是数据绑定原语，`UI_SUBJECT_STRING_LENGTH` 默认 256 字节，用于 UI 与数据的响应式连接
- **资源路径**: `lvlg_t5ai_init(asset_path)` 接受资源路径参数，字体和图片资源通过该路径加载

## 构建说明

```bash
# 作为独立项目构建
mkdir build && cd build
cmake ..
make

# 作为子模块集成到目标工程
# 在上层 CMakeLists.txt 中：
add_subdirectory(lvlg-t5ai)
target_link_libraries(your_target PRIVATE lib-ui)
```

> 需要确保 LVGL 库已正确引入（头文件路径 `lvgl/lvgl.h` 或定义 `LV_LVGL_H_INCLUDE_SIMPLE`）。

## 相关概念

- [[LVGL]] — 轻量级嵌入式 GUI 框架
- [[LVGL Editor]] — 可视化 UI 设计工具
- [[MCU GUI 架构模式]] — 嵌入式 GUI 的代码生成与运行时分离
- [[CMake 嵌入式构建]] — 跨平台构建系统在 MCU 项目中的应用
- [[XML 驱动 UI]] — 声明式 UI 描述与代码生成
- [[T5 AI 平台]] — 目标硬件平台

## 相关笔记

- [[lvgl-project]] — LVGL SquareLine Studio UI 项目
- [[zephyr]] — Zephyr RTOS 项目笔记（LVGL 集成）
- [[brithday]] — ESP32 生日项目（LVGL 手表界面）
- [[TuyaOpen]] — 涂鸦框架（LVGL 集成）
