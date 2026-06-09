---
tags: [wechat, miniprogram]
category: app/wechat
created: 2026-06-09
---
Now I have a comprehensive understanding of the project. Let me generate the Obsidian note.

```markdown
---
title: weixinxiaoapp - 微信小程序项目合集
created: 2026-06-09
category: app/wechat
tags:
  - wechat
  - miniprogram
  - 小程序
  - weui
  - nfc
  - cloud-development
status: active
---

# weixinxiaoapp - 微信小程序项目合集

## 项目概述

这是一个包含 5 个微信小程序的项目集合，涵盖了从基础模板到 NFC 硬件交互、云开发等多种开发模式，展示了微信小程序开发的不同场景和技术路线。

## 项目结构

```
weixinxiaoapp/
├── miniprogram-1/    # NFC 交互 + Skyline 渲染引擎
├── miniprogram-2/    # TypeScript + Sass 模板
├── miniprogram-3/    # 云开发 QuickStart
├── miniprogram-4/    # WeUI 组件库 + NFC
└── miniprogram-5/    # TypeScript QuickStart
```

## 技术栈

| 子项目 | 语言 | 渲染引擎 | 特性 |
|--------|------|----------|------|
| miniprogram-1 | JavaScript | Skyline | NFC, glass-easel |
| miniprogram-2 | TypeScript | Skyline | Sass, glass-easel |
| miniprogram-3 | JavaScript | WebView | 云开发 |
| miniprogram-4 | JavaScript | WebView | WeUI, NFC |
| miniprogram-5 | TypeScript | WebView | glass-easel |

- **语言**: JavaScript / TypeScript
- **样式**: CSS / Sass (SCSS)
- **UI 框架**: WeUI (miniprogram-4)
- **渲染引擎**: Skyline / WebView
- **组件框架**: glass-easel
- **云服务**: 微信云开发 (CloudBase)

## 架构与设计决策

### 1. Skyline 渲染引擎

miniprogram-1 和 miniprogram-2 采用了微信新一代 Skyline 渲染引擎，相比传统 WebView 渲染有更好的性能表现：

```json
// project.config.json
{
  "renderer": "skyline",
  "rendererOptions": {
    "skyline": {
      "defaultDisplayBlock": true,
      "defaultContentBox": true,
      "tagNameStyleIsolation": "legacy",
      "sdkVersionBegin": "3.0.0"
    }
  },
  "componentFramework": "glass-easel"
}
```

### 2. NFC 硬件交互

miniprogram-1 和 miniprogram-4 均支持 NFC 功能，用于与 ST25DV 等 NFC 标签进行通信：

```javascript
// miniprogram-1/app.js - NFC FTM 模式交互示例
wx.onNFCDiscovery(res => {
  if (res.tag && res.tag.techs.includes('iso15693')) {
    // 启用 FTM 模式（需 ST25DV 支持）
    wx.nfcA.sendCommand({
      command: [0x02, 0xA2, 0x00, 0x00, 0x03],
      success: () => console.log("FTM activated")
    });
    // 写入 Mailbox 数据
    wx.nfcA.transceive({
      data: [0x20, 0x00, ...yourData],
      success: (res) => console.log("Data written:", res.data)
    });
  }
});
```

### 3. 云开发集成

miniprogram-3 演示了微信云开发的三大基础能力：
- **云数据库**: JSON 文档型数据库，前端和云函数均可读写
- **文件存储**: 小程序前端直接上传/下载云端文件
- **云函数**: 云端运行代码，微信私有协议天然鉴权

```javascript
// miniprogram-3/miniprogram/app.js
wx.cloud.init({
  env: "", // 环境 ID
  traceUser: true,
});
```

### 4. TypeScript 支持

miniprogram-2 和 miniprogram-5 使用 TypeScript 开发，通过 `miniprogram-api-typings` 提供类型支持：

```typescript
// miniprogram-2/miniprogram/app.ts
App<IAppOption>({
  globalData: {},
  onLaunch() {
    wx.login({
      success: res => {
        console.log(res.code)
      },
    })
  },
})
```

## 关键工具函数

miniprogram-4 提供了实用的数据转换工具：

```javascript
// miniprogram-4/utils/util.js
// 字符串转 ArrayBuffer（NFC 通信常用）
const str2ab = (str) => {
  var buf = new ArrayBuffer(str.length);
  var bufView = new Uint8Array(buf);
  for (var i = 0, strLen = str.length; i < strLen; i++) {
    bufView[i] = str.charCodeAt(i);
  }
  return buf;
}

// ArrayBuffer 转字符串
const ab2str = (buf) => {
  return String.fromCharCode.apply(null, new Uint8Array(buf));
}
```

## AppID 汇总

| 子项目 | AppID |
|--------|-------|
| miniprogram-1 | touristappid (游客模式) |
| miniprogram-2 | wx516102e83f0f8f1d |
| miniprogram-3 | - |
| miniprogram-4 | wx8c870cf6267e5f6b |
| miniprogram-5 | wx8c870cf6267e5f6b |

## 构建与运行

1. 使用 **微信开发者工具** 打开对应子项目目录
2. 确保基础库版本满足要求（Skyline 需 3.0.0+）
3. 对于云开发项目(miniprogram-3)，需先在云控制台创建环境并配置 env ID
4. NFC 功能需在真机上调试，模拟器不支持

### 云函数部署

```bash
# miniprogram-3 提供的云函数部署脚本
${installPath} cloud functions deploy --e ${envId} --n quickstartFunctions --r --project ${projectPath}
```

## 关键学习点

1. **Skyline vs WebView**: Skyline 是微信新一代渲染引擎，性能更优但需要较高基础库版本，部分 API 行为有差异
2. **NFC 开发限制**: 小程序 NFC 功能仅支持 ISO 14443/15693 标准，需真机调试，且部分高级功能(如 FTM)依赖特定硬件
3. **云开发优势**: 免服务器、免域名备案、天然鉴权，适合快速原型开发
4. **TypeScript 价值**: 小程序 API 类型提示可显著提升开发效率，减少运行时错误
5. **glass-easel**: 微信自研组件框架，相比 Exparser 有更好的性能和兼容性

## 相关资源

- [[微信小程序开发]]
- [[NFC 技术]]
- [[TypeScript 基础]]
- [[WeUI 组件库]]
- [微信云开发文档](https://developers.weixin.qq.com/miniprogram/dev/wxcloud/basis/getting-started.html)
- [Skyline 渲染引擎](https://developers.weixin.qq.com/miniprogram/dev/framework/runtime/skyline/introduction.html)
```

## 相关笔记

- [[feishuxiaoapp]] — 飞书小程序项目
- [[HBuilder]] — UniApp 智能硬件应用
- [[esp32s3-nfc]] — ESP32-S3 + ST25DV NFC（NFC 相关）
- [[brithday]] — ESP32 生日项目（NFC 相关）
- [[vuepress]] — VuePress 知识库（前端技术）
