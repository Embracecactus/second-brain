---
tags:
  - harmonyos
  - deveco
  - arkts
category: app/harmonyos
created: 2026-06-09
---

# HarmonyOS DevEco Studio 项目结构与开发知识

## 概述

本地 DevEco Studio 工作空间包含三个 HarmonyOS 应用项目：`MyApplication`、`MyApplication2`、`MyApplication3`，均采用 **Stage 模型**（stageMode）开发，使用 **ArkTS** 语言，基于 **Hvigor** 构建系统。三个项目均为 `com.example.myapplication` 的初始模板项目，targeting HarmonyOS 5.0.x SDK。其中 MyApplication3 是最新的迭代，已实现多页面路由跳转功能。

**项目时间线：**
- `MyApplication` — 最早构建记录 2025-02-04，最后 2025-02-09
- `MyApplication2` — 构建记录 2025-02-09
- `MyApplication3` — 构建记录 2025-08-18（最新，含多页面功能）

## 关键知识点

### 1. HarmonyOS 应用工程结构

标准的 DevEco Studio 工程采用以下目录结构：

```
ProjectRoot/
├── AppScope/                    # 应用级配置
│   └── app.json5                # 应用全局信息（bundleName, version 等）
├── entry/                       # 主模块（HAP Module）
│   ├── src/
│   │   ├── main/
│   │   │   ├── ets/             # ArkTS 源码
│   │   │   │   ├── entryability/         # UIAbility 入口
│   │   │   │   ├── entrybackupability/   # Backup 扩展能力
│   │   │   │   └── pages/                # 页面组件
│   │   │   ├── resources/       # 资源文件
│   │   │   └── module.json5     # 模块配置
│   │   ├── ohosTest/            # 设备端测试
│   │   └── test/                # 本地单元测试
│   ├── build-profile.json5      # 模块构建配置
│   ├── hvigorfile.ts            # 模块级 Hvigor 构建脚本
│   └── oh-package.json5         # 模块级包描述
├── build-profile.json5          # 项目级构建配置
├── hvigorfile.ts                # 项目级 Hvigor 构建脚本
├── oh-package.json5             # 项目级包描述
├── oh-package-lock.json5        # 依赖锁文件
├── code-linter.json5            # 代码检查配置
└── hvigor/                      # Hvigor 构建系统配置
    └── hvigor-config.json5
```

### 2. Stage 模型核心概念

- **UIAbility** — 应用的入口组件，管理生命周期和窗口
- **ExtensionAbility** — 扩展能力（如 Backup、Service 等）
- **WindowStage** — 窗口舞台，承载页面内容
- **Want** — 用于组件间通信的消息对象

### 3. ArkTS 声明式 UI 开发

ArkTS 基于 TypeScript 扩展，采用声明式 UI 范式：
- `@Entry` — 标记页面入口组件
- `@Component` — 标记自定义组件
- `@State` — 状态装饰器，驱动 UI 更新
- 布局容器：`Row()`、`Column()`、`RelativeContainer()`

### 4. 页面路由

通过 `UIContext.getRouter()` 获取 Router 实例，使用 `router.pushUrl()` 进行页面跳转，`router.back()` 返回上一页。页面路由配置在 `main_pages.json` 中声明。

### 5. 测试框架

- **本地单元测试** — 使用 `@ohos/hypium` 框架，文件位于 `src/test/`
- **设备端测试** — 使用 `@ohos/hypium` + `@ohos/hamock`，文件位于 `src/ohosTest/`
- 测试结构：`describe()` -> `it()` -> `expect().assertXxx()`

### 6. 安全与代码质量

`code-linter.json5` 配置了严格的安全规则，包括：
- 加密算法安全检查（AES、RSA、DSA、DH、ECDSA、3DES）
- 性能推荐规则 `@performance/recommended`
- TypeScript ESLint 推荐规则 `@typescript-eslint/recommended`

## 技术细节

### SDK 版本差异

| 项目 | compatibleSdkVersion | 模式 |
|------|---------------------|------|
| MyApplication | 5.0.2(14) | stageMode |
| MyApplication2 | 5.0.0(12) | stageMode |
| MyApplication3 | 5.0.2(14) | stageMode |

### 构建模式

每个项目都配置了两种构建模式：
- **debug** — 开发调试模式
- **release** — 发布模式，支持代码混淆（obfuscation，默认关闭）

### 目标设备类型

所有项目均支持三种设备类型：`phone`、`tablet`、`2in1`（二合一设备）

### Hvigor 构建系统配置

```json5
// hvigor-config.json5 关键配置项
{
  "execution": {
    // "daemon": true,        // 守护进程编译
    // "incremental": true,   // 增量编译
    // "parallel": true,      // 并行编译
    // "typeCheck": false,    // 类型检查
  },
  "nodeOptions": {
    // "maxOldSpaceSize": 8192  // Node.js 堆内存上限（MB）
  }
}
```

### 严格模式配置

```json5
// build-profile.json5
"buildOption": {
  "strictMode": {
    "caseSensitiveCheck": true,     // 大小写敏感检查
    "useNormalizedOHMUrl": true     // 规范化 OHM URL
  }
}
```

## 代码/配置片段

### UIAbility 生命周期（EntryAbility.ets）

```typescript
import { AbilityConstant, ConfigurationConstant, UIAbility, Want } from '@kit.AbilityKit';
import { hilog } from '@kit.PerformanceAnalysisKit';
import { window } from '@kit.ArkUI';

export default class EntryAbility extends UIAbility {
  onCreate(want: Want, launchParam: AbilityConstant.LaunchParam): void {
    this.context.getApplicationContext().setColorMode(
      ConfigurationConstant.ColorMode.COLOR_MODE_NOT_SET
    );
  }

  onWindowStageCreate(windowStage: window.WindowStage): void {
    windowStage.loadContent('pages/Index', (err) => {
      if (err.code) {
        hilog.error(0x0000, 'testTag', 'Failed: %{public}s', JSON.stringify(err));
        return;
      }
    });
  }

  onForeground(): void { /* 前台回调 */ }
  onBackground(): void { /* 后台回调 */ }
  onDestroy(): void { /* 销毁回调 */ }
  onWindowStageDestroy(): void { /* 窗口销毁回调 */ }
}
```

### 页面路由跳转（MyApplication3 - Index.ets）

```typescript
import { BusinessError } from '@kit.BasicServicesKit';

@Entry
@Component
struct Index {
  @State message: string = 'Hello World';

  build() {
    Row() {
      Column() {
        Text(this.message).fontSize(50).fontWeight(FontWeight.Bold)
        Button() {
          Text('Next').fontSize(30).fontWeight(FontWeight.Bold)
        }
        .type(ButtonType.Capsule)
        .onClick(() => {
          let uiContext: UIContext = this.getUIContext();
          let router = uiContext.getRouter();
          router.pushUrl({ url: 'pages/Second' })
            .then(() => { console.info('Jump to second page.') })
            .catch((err: BusinessError) => {
              console.error(`Failed. Code: ${err.code}, msg: ${err.message}`)
            })
        })
      }.width('100%')
    }.height('100%')
  }
}
```

### 页面路由配置（main_pages.json）

```json
// 单页面项目（MyApplication/MyApplication2）
{ "src": ["pages/Index"] }

// 多页面项目（MyApplication3）
{ "src": ["pages/Index", "pages/Second"] }
```

### 应用全局配置（app.json5）

```json5
{
  "app": {
    "bundleName": "com.example.myapplication",
    "vendor": "example",
    "versionCode": 1000000,
    "versionName": "1.0.0",
    "icon": "$media:app_icon",
    "label": "$string:app_name"
  }
}
```

### 模块配置（module.json5）

```json5
{
  "module": {
    "name": "entry",
    "type": "entry",
    "mainElement": "EntryAbility",
    "deviceTypes": ["phone", "tablet", "2in1"],
    "deliveryWithInstall": true,
    "pages": "$profile:main_pages",
    "abilities": [{
      "name": "EntryAbility",
      "srcEntry": "./ets/entryability/EntryAbility.ets",
      "exported": true,
      "skills": [{
        "entities": ["entity.system.home"],
        "actions": ["action.system.home"]
      }]
    }],
    "extensionAbilities": [{
      "name": "EntryBackupAbility",
      "srcEntry": "./ets/entrybackupability/EntryBackupAbility.ets",
      "type": "backup"
    }]
  }
}
```

### BackupExtensionAbility（备份扩展）

```typescript
import { BackupExtensionAbility, BundleVersion } from '@kit.CoreFileKit';

export default class EntryBackupAbility extends BackupExtensionAbility {
  async onBackup() {
    await Promise.resolve();
  }
  async onRestore(bundleVersion: BundleVersion) {
    await Promise.resolve();
  }
}
```

### 本地单元测试模板

```typescript
import { describe, it, expect, beforeAll, beforeEach, afterEach, afterAll } from '@ohos/hypium';

export default function localUnitTest() {
  describe('localUnitTest', () => {
    beforeAll(() => { /* 全局前置 */ });
    beforeEach(() => { /* 用例前置 */ });
    afterEach(() => { /* 用例后置 */ });
    afterAll(() => { /* 全局后置 */ });

    it('assertContain', 0, () => {
      let a = 'abc';
      let b = 'b';
      expect(a).assertContain(b);
      expect(a).assertEqual(a);
    });
  });
}
```

### Hvigor 构建脚本

```typescript
// 项目级 hvigorfile.ts
import { appTasks } from '@ohos/hvigor-ohos-plugin';
export default { system: appTasks, plugins: [] }

// 模块级 entry/hvigorfile.ts
import { hapTasks } from '@ohos/hvigor-ohos-plugin';
export default { system: hapTasks, plugins: [] }
```

## 相关链接

- [HarmonyOS 应用开发文档](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/)
- [ArkTS 语言指南](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/arkts-get-started/)
- [Stage 模型应用组件](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/application-model-description/)
- [DevEco Studio 使用指南](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/ide-overview/)
- [Hvigor 构建系统](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/ide-hvigor/)
- [@ohos/hypium 测试框架](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/ide-unit-test/)
- [module.json5 配置参考](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/module-configuration-file/)
