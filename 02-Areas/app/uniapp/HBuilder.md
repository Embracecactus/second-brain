---
tags:
  - uniapp
  - vue3
  - bluetooth
  - nfc
  - iot
  - ble
  - mesh
  - hbuilder
  - 蓝牙
  - 智能硬件
category: app/uniapp
created: 2026-06-09
status: active
project_count: 2
---

# HBuilder 项目集 — UniApp 智能硬件应用

## 项目概述

HBuilderProjects 目录下包含两个基于 uniapp + Vue 3 的智能硬件控制应用：**smartlight**（蓝牙车灯控制系统）和 **NFC标签读取写入示例**。两者均面向 IoT 场景，通过 uni-app 跨平台框架实现一套代码多端运行（H5 / 小程序 / Android / iOS），分别演示了 BLE 蓝牙通信和 NFC 近场通信的完整实现。

## 技术栈

| 类别 | 技术 |
|------|------|
| 框架 | uniapp + Vue 3 (Composition API) |
| 构建 | Vite 4.0 + @dcloudio/vite-plugin-uni |
| 样式 | CSS3 + Flex + Grid + rpx 响应式单位 |
| 蓝牙 | uni-app BLE API (openBluetoothAdapter / createBLEConnection / writeBLECharacteristicValue) |
| NFC | plus.android.importClass 调用 Android NfcAdapter 原生 API |
| 存储 | uni.setStorageSync / uni.getStorageSync 本地持久化 |
| 状态管理 | globalData 全局状态 + 页面级 data |
| 平台适配 | mp-weixin / mp-alipay / mp-baidu / mp-toutiao / app-plus / H5 |

## 项目一：smartlight — 蓝牙车灯控制系统

### 架构设计

```
smartlight/
├── pages/
│   ├── index/index.vue      # 首页 — 蓝牙状态与快捷控制
│   ├── device/device.vue     # 设备管理 — 扫描、连接、管理
│   ├── control/control.vue   # 车灯控制 — 模式选择与参数调节
│   ├── mesh/mesh.vue         # Mesh组网 — 网络管理与批量控制
│   ├── debug/debug.vue       # 调试盘 — RGB调色与参数实时调节
│   └── profile/profile.vue   # 我的 — 用户信息与设置
├── utils/
│   ├── bluetooth.js          # BluetoothManager 单例类
│   └── mesh.js               # MeshManager 单例类
├── common/
│   ├── constants.js          # 模式/颜色/命令/配置常量
│   └── common.css            # 全局通用样式
├── App.vue                   # 应用入口 + 蓝牙初始化
├── main.js                   # createSSRApp 入口
├── pages.json                # 4 Tab 底部导航配置
├── manifest.json             # 平台权限声明
└── package.json              # 依赖与脚本
```

### 核心设计决策

**1. 单例模式管理蓝牙和Mesh**

BluetoothManager 和 MeshManager 均以 ES6 class 实现并导出单例实例，确保全局唯一的连接状态管理。

```javascript
// utils/bluetooth.js — 单例导出
export default new BluetoothManager()
```

**2. 命令协议结构化**

所有蓝牙指令统一为 `{ type, data, timestamp }` JSON 格式，通过 `stringToArrayBuffer` 转为 BLE 可写入的 ArrayBuffer。

```javascript
// 命令类型定义 (common/constants.js)
export const BLE_COMMANDS = {
  MODE: 'mode',
  BRIGHTNESS: 'brightness',
  SPEED: 'speed',
  COLOR: 'color',
  APPLY_COLOR: 'applyColor',
  EMERGENCY: 'emergency',
  SYNC: 'sync',
  TEST: 'test'
}
```

**3. 模式配置驱动 UI**

6种灯光模式通过配置对象声明式定义，每个模式携带 icon、渐变色、是否需要颜色选择器等属性，控制页面通过 `v-for` 渲染模式网格，通过 `needsColor` 动态显示/隐藏调试盘。

```javascript
// 典型模式配置
{
  id: 'breathing',
  name: '呼吸灯',
  desc: '渐亮渐暗效果',
  icon: '💨',
  color: 'linear-gradient(45deg, #2196F3, #64B5F6)',
  needsColor: true,
  supportsBrightness: true,
  supportsSpeed: true
}
```

**4. 设备过滤策略**

扫描时通过设备名称关键词过滤（`car`, `light`, `led`, `lamp`, `车灯`, `CarLight`），避免展示无关蓝牙设备。

**5. 设置持久化**

灯光参数（模式、亮度、速度、颜色）通过 `uni.setStorageSync` 自动保存，下次进入自动恢复。

### 权限配置要点

manifest.json 中声明了完整的蓝牙权限链：
- Android: `BLUETOOTH` + `BLUETOOTH_ADMIN` + `ACCESS_COARSE_LOCATION` + `ACCESS_FINE_LOCATION`
- iOS: `com.apple.developer.bluetooth-central` entitlement
- 微信小程序: `scope.bluetooth` + `requiredBackgroundModes: ["bluetooth-central"]`

## 项目二：NFC标签读取写入示例

### 架构设计

```
NFC标签读取ID 写入内容读取内容-sample/
├── pages/index/index.vue                  # 主页 — 三种NFC操作模式切换
├── uni_modules/yang-nfc-tag/js_sdk/nfc.js # NFC核心SDK封装
├── App.vue                                # 最简入口
├── manifest.json                          # OAuth + 微信配置
└── pages.json                             # 单页面配置
```

### 核心设计决策

**1. 三种 NFC 操作模式**

通过 radio 切换 `cardNo`（读取卡号）、`write`（写入内容）、`read`（读取内容），同一 NFC 感应事件根据当前模式分发到不同处理函数。

```javascript
// nfc.js — 模式分发
function readCardNo() {
  if (nfcType === 'cardNo') {
    __read_no(intent)
  } else if (nfcType === 'write') {
    __write(intent)
  } else {
    __read(intent)
  }
}
```

**2. Android 原生 API 桥接**

通过 `plus.android.importClass` 直接调用 Java 类（NfcAdapter, NdefRecord, NdefMessage, PendingIntent），利用 `plus.globalEvent.addEventListener` 监听 `newintent` / `pause` / `resume` 生命周期事件管理前台调度。

**3. NDEF 写入流程**

写入时先尝试 `Ndef.get(tag)` 获取已格式化的 NDEF 标签；若为 null 则回退到 `NdefFormatable.get(tag)` 进行格式化后写入。同时校验 `isWritable()` 和 `getMaxSize()`。

**4. 卡号读取算法**

读取卡号后进行字节翻转（`reverseTwo`），将原始字节数组转为 16 进制字符串，每 2 位翻转后拼接，最终转为 10 进制显示。

## 关键学习与洞察

1. **uni-app 蓝牙 API 封装模式**: 将 Promise 包装的 uni API 封装为 class 单例，是 uni-app IoT 项目的标准实践。BluetoothManager 展示了完整的 scan -> connect -> discover services -> discover characteristics -> write 流程。

2. **BLE 数据传输限制**: `stringToArrayBuffer` 使用单字节编码（`charCodeAt`），仅支持 ASCII 范围。中文等多字节字符需要使用 TextEncoder 或自定义编码方案。

3. **NFC 前台调度**: Android NFC 需要在 `resume` 时启用 `enableForegroundDispatch`，`pause` 时禁用，确保应用在前台时优先处理 NFC 事件而非系统默认行为。

4. **Mesh 网络模拟**: smartlight 的 MeshManager 当前为模拟实现（mock），`scanAvailableNetworks` 返回硬编码数据。实际生产环境需对接真实 Mesh 协议栈（如 nRF Mesh SDK 或 BLE Mesh 规范）。

5. **跨平台蓝牙差异**: 微信小程序需要声明 `scope.bluetooth` 权限且需要用户手动授权；App 端需要在 manifest.json 中声明 Android permissions 和 iOS capabilities；H5 端依赖 Web Bluetooth API，浏览器兼容性有限。

6. **rpx 响应式设计**: 所有尺寸使用 rpx 单位（750rpx = 屏幕宽度），配合 Flex + Grid 布局实现多端自适应，是 uni-app 项目的标准做法。

## 构建与运行

### 环境要求
- HBuilderX 3.0+
- Node.js 14+
- 支持蓝牙 4.0+ 的设备（smartlight）
- 支持 NFC 的 Android 设备（NFC 示例）

### 运行命令

```bash
# smartlight 项目
cd smartlight
npm install
npm run dev:h5          # H5 开发
npm run dev:mp-weixin   # 微信小程序
npm run dev:app-plus    # App

# NFC 项目（仅支持 Android App）
# 通过 HBuilderX 运行到 Android 真机
```

### 测试建议
1. 先在 H5 环境测试 UI 布局与交互
2. 微信小程序环境测试蓝牙扫描与连接
3. Android 真机测试完整 BLE 通信流程
4. NFC 示例必须在支持 NFC 的 Android 真机上运行

## 相关概念链接

- [[uni-app]] — 跨平台应用框架
- [[BLE蓝牙通信]] — 低功耗蓝牙协议与 GATT 服务
- [[NFC近场通信]] — NDEF 标签读写与前台调度
- [[Mesh组网]] — BLE Mesh 网络拓扑与批量控制
- [[Vue3 Composition API]] — 响应式状态管理
- [[HBuilderX]] — DCloud 官方 IDE
- [[IoT物联网应用开发]] — 智能硬件控制应用模式

## 相关笔记

- [[weixinxiaoapp]] — 微信小程序项目合集（同为小程序开发）
- [[feishuxiaoapp]] — 飞书小程序项目
- [[AndroidStudio]] — AndroidStudioProjects（IoT 设备管理）
- [[esp32s3-nfc]] — ESP32-S3 + ST25DV NFC（NFC 相关）
- [[esp32c3]] — ESP32-C3 智能尾灯（BLE 蓝牙相关）
