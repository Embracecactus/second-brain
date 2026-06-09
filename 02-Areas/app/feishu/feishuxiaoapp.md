---

```yaml
---
title: "飞书小程序 Blank Demo 项目"
tags:
  - feishu
  - miniprogram
  - tt-miniapp
  - javascript
  - sample-project
category: app/feishu
created: 2026-06-09
status: reference
project_path: "/mnt/c/Users/lijian/workspace/feishuxiaoapp"
---

# 飞书小程序 Blank Demo 项目

## 项目概述

这是飞书（字节跳动）小程序官方 Blank Demo 项目，提供了一套完整的组件与 API 能力示例，涵盖 70+ 个页面，涵盖视图容器、表单、媒体、设备、开放接口、网络等全部小程序能力。项目可作为飞书小程序开发的参考模板与学习资料。

## 技术栈

- **平台**: 飞书小程序 (Feishu MiniApp / TT MiniApp)
- **语言**: JavaScript (ES6+)
- **框架**: 飞书小程序原生框架（类微信小程序架构）
- **样式**: TTSS (类似 WXSS 的字节跳动小程序样式语言，使用 rpx 响应式单位)
- **模板**: TTML (类似 WXML 的字节跳动小程序模板语言)
- **国际化**: 自定义 i18n 模块，支持中文/英文
- **后端**: 腾讯云 (`14592619.qcloud.la`) 作为示例后端服务
- **SDK 版本**: libVersion `1.9.1`
- **构建工具**: 飞书开发者工具 (IDE 编译，minified + postcss + ES6 转译)

## 项目结构

```
feishuxiaoapp/
  blank/                          # 空白模板项目根目录
    app.js                        # 应用入口，生命周期管理 + 热更新
    app.json                      # 应用配置（页面路由、TabBar、窗口样式）
    app.ttss                      # 全局样式
    config.js                     # 后端服务地址配置
    project.config.json           # 项目编译配置
    image/                        # 静态图片资源（TabBar 图标等）
    util/
      util.js                     # 工具函数（时间格式化、经纬度格式化）
    page/
      i18n/                       # 国际化
        index.js                  # i18n 入口（根据系统语言自动切换）
        zh.js                     # 中文语言包
        en.js                     # 英文语言包
      component/                  # 组件示例
        index.js                  # 组件示例首页
        pages/                    # 24 个组件示例页
          button/ canvas/ checkbox/ editor/ form/ icon/
          image/ input/ label/ navigator/ picker/ picker-view/
          progress/ radio/ richtext/ scroll-view/ slider/
          swiper/ switch/ text/ textarea/ video/ view/ web-view/
      API/                        # API 示例
        index.js                  # API 示例首页
        pages/                    # 49 个 API 示例页
          login/ request/ share/ storage/ get-location/
          scan-code/ image/ video/ voice/ file/ canvas/ ...
```

## 架构与关键设计决策

### 1. 应用生命周期管理

入口文件 `app.js` 采用飞书小程序标准生命周期模型，通过 `tt.getUpdateManager()` 实现应用热更新检测：

```javascript
App({
  onLaunch: function (args) {
    console.log('App Launch')
  },
  onShow: function (args) {
    let updateManager = tt.getUpdateManager();
    updateManager.onCheckForUpdate((result) => {
      console.log('is there any update?:' + result.hasUpdate);
    });
    updateManager.onUpdateReady((result) => {
      tt.showModal({
        title: 'Update information',
        content: 'new version is ready, do you want to restart app?',
        success: res => {
          if (res.confirm) {
            updateManager.applyUpdate();
          }
        }
      })
    });
  },
  globalData: {
    hasLogin: false,
    openid: null
  }
})
```

### 2. 路由与页面注册

所有页面在 `app.json` 的 `pages` 数组中声明式注册，共 70+ 个页面路由。页面分为两大类：
- **Component 类**: 24 个基础 UI 组件示例（view, text, image, button, form, input, picker 等）
- **API 类**: 49 个平台能力示例（login, request, storage, location, scan-code 等）

### 3. 双 Tab 导航架构

采用 `tabBar` 底部双 Tab 设计：Component 与 API，颜色方案使用飞书品牌色 `#3370FF` 作为选中色。

### 4. 国际化方案

自实现轻量级 i18n 模块，通过 `tt.getSystemInfoSync()` 获取系统语言，自动切换中/英文：

```javascript
// page/i18n/index.js
import zh from './zh'
import en from './en'
let i18n = en
try {
    var res = tt.getSystemInfoSync();
    if (res.language && res.language.indexOf('zh') != -1) {
        i18n = zh;
    }
} catch (error) {
    console.log(error);
}
export default i18n;
```

### 5. 全局样式体系

`app.ttss` 定义了一套完整的响应式样式系统：
- 使用 `rpx` 单位实现屏幕适配
- 统一的页面容器布局（`.container` flexbox 布局）
- 通用的 section/card 样式组件
- 原生字体栈 `-apple-system-font, Helvetica Neue, Helvetica, sans-serif`

## 关键 API 能力覆盖

项目示例覆盖了飞书小程序的全部核心能力模块：

| 分类 | 能力 |
|------|------|
| **开放接口** | 登录、获取用户信息、分享、联系人选择、生物认证、密码验证、活体检测 |
| **网络** | HTTP Request、WebSocket、文件上传/下载 |
| **媒体** | 图片、录音、文件、音频、视频、摄像头 |
| **数据缓存** | Storage 增删改查（同步/异步） |
| **位置** | 获取位置、查看位置、选择位置 |
| **设备** | 系统信息、网络状态、重力感应、罗盘、扫码、剪贴板、振动、截屏监听 |
| **界面** | Toast、Modal、ActionSheet、导航栏、动画、下拉刷新 |
| **特殊能力** | 水印检测、邮件发送、设备认证、schema 跳转 |

## 构建与运行

### 环境要求
- 安装飞书开发者工具（或字节跳动小程序开发者工具）

### 运行步骤
1. 打开飞书开发者工具
2. 导入 `blank/` 目录作为项目
3. 在 `project.config.json` 中填入 `appid`（在飞书开放平台申请）
4. 点击编译运行

### 注意事项
- `project.config.json` 中 `appid` 字段为空，需要申请后填入才能正常调试
- 后端服务地址 (`config.js`) 指向腾讯云示例服务，生产环境需替换
- `urlCheck` 设置为 `false`，允许请求非 HTTPS 域名（仅开发阶段）

## 关键学习点

1. **飞书小程序与微信小程序的高度相似性**: TTML/TTSS 对应 WXML/WXSS，`tt.*` API 对应 `wx.*` API，迁移成本极低
2. **应用内热更新机制**: 使用 `UpdateManager` 实现无需用户手动更新的静默更新流程
3. **响应式布局**: `rpx` 单位在 750rpx 设计稿下自动适配不同屏幕
4. **文件选择能力**: 飞书小程序特有的 `chooseMessageFile` 等 API 支持从飞书会话中选择附件
5. **企业级特性**: 生物认证、密码二次验证、活体检测等面向企业安全场景的 API

## 相关资源

- [[飞书小程序开发文档]]
- [[字节跳动小程序平台]]
- [[小程序原生框架]]
- [[rpx 响应式布局]]
- [[小程序生命周期]]
- [[腾讯云小程序后端]]
```
## 相关笔记

- [[weixinxiaoapp]] — 微信小程序项目合集
- [[HBuilder]] — UniApp 智能硬件应用
- [[harmonyos-dev]] — HarmonyOS DevEco Studio 项目
- [[vuepress]] — VuePress 知识库（同为前端技术）
