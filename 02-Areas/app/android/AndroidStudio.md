---
tags:
  - android
  - app
  - iot
  - smart-home
  - e-paper
  - java
category: app/android
created: 2026-06-09
status: active
projects:
  - cherry-app
  - cherrydevice
  - smartlight
  - lianxicloud
  - lianxi
  - MyApplication
  - sample1
---

# AndroidStudioProjects

## 项目概述

AndroidStudioProjects 是一个 Android 项目集合，包含多个基于 Java 的 Android 应用，涵盖物联网设备管理、智能灯光控制、墨水屏图片传输、NFC 读写等功能。这些项目共享相似的技术栈（Material Design + Bottom Navigation + Fragment），面向 IoT/智能家居场景进行开发和练习。

## 技术栈

| 类别 | 技术 |
|------|------|
| 语言 | Java 11 |
| 构建工具 | Gradle (Kotlin DSL / Groovy) |
| AGP 版本 | 8.10.1 |
| compileSdk | 35 / 36 |
| minSdk | 33 (Android 13) |
| UI 框架 | Material Components, ViewBinding |
| 架构组件 | Navigation, Lifecycle (ViewModel + LiveData) |
| 网络通信 | HttpURLConnection (原生) |
| 测试 | JUnit, Espresso |

## 项目清单与架构

### 1. cherry-app（com.example.cherry）

核心功能：墨水屏 (E-Paper) 图片传输工具，连接 ESP32 WiFi AP 向墨水屏推送图片。

- `WifiTransferActivity` -- 核心页面，实现图片选择、Floyd-Steinberg 抖动算法三色量化、二值化处理、HTTP POST 上传至 ESP32 (`192.168.4.1`)
- `MemoActivity` -- 备忘录功能
- `NfcActivity` -- NFC 读写功能
- 墨水屏分辨率：250x128，支持黑白/黑白红三色模式

### 2. cherrydevice（com.example.cherrydevice）

核心功能：IoT 设备管理应用，带底部导航的多 Fragment 架构。

- **架构**：MVVM 模式（ViewModel + LiveData + ViewBinding）
- **导航**：Navigation Component + BottomNavigationView
- **页面**：HomeFragment（设备列表）、SceneFragment（场景）、MeFragment（个人中心）
- **数据模型**：`Device` -- 包含 id, name, type, online, iconResName
- **适配器**：`DeviceAdapter` -- RecyclerView 展示设备卡片，动态加载图标资源

### 3. smartlight（com.example.smartlight）

核心功能：智能灯光控制应用，实现了沉浸式状态栏。

- **架构**：Fragment + BottomNavigationView 手动切换
- **页面**：HomeFragment、DevicesFragment、ProfileFragment
- **特色**：完整的沉浸式状态栏实现（透明状态栏 + 动态文字颜色适配 + 日夜模式）

### 4. lianxicloud / lianxi / sample1 / MyApplication

练习项目，功能包括设备管理、用户注册等基础功能开发。

## 核心知识点

### 沉浸式状态栏实现

```java
// 关键 API 调用
getWindow().getDecorView().setSystemUiVisibility(
    View.SYSTEM_UI_FLAG_LAYOUT_STABLE |
    View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN
);
getWindow().setStatusBarColor(Color.TRANSPARENT);
WindowInsetsControllerCompat controller =
    WindowCompat.getInsetsController(getWindow(), getWindow().getDecorView());
controller.setAppearanceLightStatusBars(true);
```

- 主题需继承 `Theme.MaterialComponents.DayNight.NoActionBar`
- 每个 Fragment 顶部添加占位 View（高度 `?attr/actionBarSize`）防止内容被遮挡
- `fitsSystemWindows="false"` 让内容自行处理状态栏区域
- 兼容性：API 21+ 基本效果，API 23+ 文字颜色控制，API 28+ 刘海屏适配

### Floyd-Steinberg 抖动算法（墨水屏图像处理）

将 RGB 图像量化为三色（白/黑/红）调色板，核心流程：
1. 遍历每个像素，累加误差扩散值
2. 计算与白/黑/红三色的欧几里得距离，选择最近色
3. 误差按 7/16, 3/16, 5/16, 1/16 比例扩散到右、左下、下、右下四个相邻像素

### IoT 设备管理架构

- `Device` 数据模型封装设备属性（id, name, type, online, iconResName）
- `DeviceAdapter` 使用 RecyclerView 展示设备列表，通过资源名动态加载图标
- 设备状态用颜色标签区分：在线绿色 (#4CAF50)，离线红色 (#F44336)

## 重要代码片段

### ESP32 墨水屏通信协议

```java
// 上传地址
private static final String UPLOAD_URL = "http://192.168.4.1/upload/epaper.bin";
private static final String DELETE_URL = "http://192.168.4.1/delete/epaper.bin";

// bin 文件格式：blackFrame + redFrame，每帧 = bytesPerRow * EPD_WIDTH 字节
// bytesPerRow = (EPD_HEIGHT + 7) / 8 = 16 字节
// 每帧大小 = 16 * 250 = 4000 字节
// 总文件大小 = 8000 字节
```

### Navigation Component 设置（cherrydevice）

```java
AppBarConfiguration appBarConfiguration = new AppBarConfiguration.Builder(
    R.id.navigation_home, R.id.navigation_scene, R.id.navigation_me).build();
NavController navController = Navigation.findNavController(this, R.id.nav_host_fragment_activity_main);
NavigationUI.setupActionBarWithNavController(this, navController, appBarConfiguration);
NavigationUI.setupWithNavController(binding.navView, navController);
```

## 构建/运行方法

```bash
# 在 Android Studio 中打开对应项目目录
# 或使用命令行构建
cd <project-name>
./gradlew assembleDebug

# 安装到设备
./gradlew installDebug

# cherry-app 需连接 ESP32 热点 (192.168.4.1) 才能使用墨水屏上传功能
```

注意事项：
- 所有项目 minSdk = 33，需要 Android 13+ 设备
- cherry-app 的 WiFi 传输功能需要设备连接到 ESP32 的 AP 热点
- cherrydevice 使用 Navigation Component，需确保 nav_graph.xml 正确配置

## 相关笔记链接

- [[Android-沉浸式状态栏]]
- [[ESP32-墨水屏开发]]
- [[Android-Navigation-Component]]
- [[Android-MVVM架构]]
- [[Floyd-Steinberg抖动算法]]
- [[RecyclerView最佳实践]]
- [[Android-ViewBinding]]

## 相关笔记

- [[arduino]] — ESP32 墨水屏驱动项目（cherry-app 墨水屏 App）
- [[esp32-box-lite]] — ESP32-S3-BOX-Lite（同为 IoT 设备）
- [[brithday]] — ESP32 生日项目（智能手表 App 生态）
- [[HBuilder]] — UniApp 智能硬件应用
- [[weixinxiaoapp]] — 微信小程序项目合集
