---
title: WeChatExporter
tags:
  - tools
  - wechat
  - backup
  - nodejs
  - nwjs
  - sqlite
  - angularjs
  - chat-export
category: tools
created: 2026-06-09
status: archived
project_url: https://github.com/tsycnh/WeChatExporter
platform: macOS (primary), Windows/Linux (partial)
license: ISC
---

# WeChatExporter - 微信聊天记录导出工具

## 项目概述

WeChatExporter 是一款基于 Node.js 的微信聊天记录导出工具，无需越狱手机即可导出备份微信聊天记录，支持文字、语音、图片、视频的查看。项目使用 NW.js (原 node-webkit) 作为桌面应用框架，AngularJS 作为前端框架，通过解析 iOS iTunes 备份中的微信 SQLite 数据库来实现数据导出。

## 技术栈

| 层级 | 技术 |
|------|------|
| 桌面框架 | NW.js v0.40.1 (node-webkit) |
| 前端框架 | AngularJS 1.6.1 |
| UI 框架 | Bootstrap 3.3.7 |
| 路由 | angular-ui-router |
| 数据库 | SQLite3 (原生 Node.js 绑定) |
| 音频解码 | silk-v3-decoder (微信语音格式) |
| 构建工具 | Grunt + nw-builder / nwjs-builder-phoenix |
| 数据格式 | plist (iOS 配置文件解析) |
| 包管理 | npm |

## 架构与核心设计

### 应用流程

```
iOS iTunes 备份 → iMazing 导出 Documents → WeChatExporter 解析 → 独立输出目录
```

软件分为两个主要功能模块：

1. **数据解析导出模块** (soft1 → chatList → soft2)
   - 解析 iTunes 备份中的微信数据
   - 导出文字、语音(SILK→MP3)、图片、视频到独立目录

2. **聊天记录查看模块** (soft3 → chatDetail)
   - 加载导出的数据目录
   - 分页显示聊天记录，支持图片放大查看

### 数据存储结构

```
Documents/
├── {user_md5}/              # 微信用户 MD5 标识
│   ├── DB/
│   │   ├── MM.sqlite        # 主聊天数据库
│   │   └── WCDB_Contact.sqlite  # 联系人数据库
│   ├── Audio/{chatter_md5}/ # 语音文件 (.aud)
│   ├── Img/{chatter_md5}/   # 图片文件 (.pic, .pic_thum)
│   ├── Video/{chatter_md5}/ # 视频文件 (.mp4, .video_thum)
│   └── mmsetting.archive    # 用户配置 (plist 格式)
```

### 微信消息类型映射

| Type 值 | 消息类型 | 处理方式 |
|---------|---------|---------|
| 1 | 文字消息 | 直接读取 Message 字段 |
| 3 | 图片消息 | 拷贝 .pic 和 .pic_thum 文件 |
| 34 | 语音消息 | SILK 格式转 MP3 |
| 42 | 名片 | 仅显示类型标识 |
| 43/62 | 视频/小视频 | 拷贝 .mp4 和 .video_thum 文件 |
| 47 | 动画表情 | 显示占位符 |
| 48 | 位置 | 显示占位符 |
| 49 | 分享链接 | 显示占位符 |
| 50 | 语音/视频电话 | 显示占位符 |
| 10000 | 系统消息 | 显示占位符 |

### AngularJS 路由设计

```
/newEntry  → 主入口页面（选择功能）
/chatList  → 聊天列表（选择导出对象）
/soft2     → 数据导出配置与执行
/chatDetail → 聊天记录查看
```

## 核心代码片段

### 联系人信息解码（十六进制 → UTF-8）

微信数据库中联系人信息以十六进制编码存储，需要特殊解码：

```javascript
var decode_user_name_info = function (hex_string) {
    if (hex_string.substr(0, 2) == "x'") {
        hex_string = hex_string.substring(2, hex_string.length - 1)
    }
    var i = 0
    var all_data = {}
    while (i < hex_string.length) {
        var current_mark = hex_string.substr(i, 2)      // 字段标记: 0a=昵称, 12=微信号, 1a=备注
        var data_length = parseInt(hex_string.substr(i + 2, 2), 16) * 2;
        var hex_data = hex_string.substr(i + 4, data_length)
        var utf8_data = hex_to_utf8(hex_data)
        i += 4 + data_length
        all_data[current_mark] = utf8_data
    }
    return {
        "nickname": all_data['0a'],
        "wechatID": all_data['12'],
        "remark": all_data['1a']
    }
}
```

### 语音文件转换流程

```javascript
$scope.processAudio = function (localID, createTime) {
    // 使用 silk-v3-decoder 将微信 SILK 格式转为 MP3
    var command = "sh " + $scope.documentsPath.audioFolder + "/converter.sh " + localID + ".aud mp3";
    var stdOut = require('child_process').execSync(command, { encoding: "utf8" });
    if (stdOut.indexOf("[OK]") > 0) {
        // 转换成功，拷贝到输出目录
        fse.copySync(audioFileOld, audioFileNew);
        result.resourceName = formatTimeStamp(createTime) + ".mp3";
    }
};
```

### SQLite 数据库查询模式

```javascript
// 使用 Promise 链式查询获取聊天表统计信息
getRowName
    .then(getCount)
    .then(function (result) {
        // result[0]: table name, result[1]: count
        if (!(row.count <= $scope.messageLimit)) {
            $scope.dbTables.push(result);
        }
    });
```

## 关键学习与洞察

1. **iOS 数据备份解析**: 微信在 iOS 上的数据存储在 iTunes 备份的 Documents 目录下，用户标识使用 MD5 哈希，聊天记录按联系人分表存储 (Chat_{contact_md5})

2. **SILK 音频格式**: 微信语音使用 SILK v3 编码格式，需要专门的解码器转换为 MP3 才能在浏览器中播放

3. **plist 解析**: iOS 的 mmsetting.archive 文件使用 plist 格式存储用户信息（昵称、微信号、头像 URL），需要先用 `plutil` 转换为 XML 再解析

4. **NW.js 原生能力**: 利用 NW.js 的 Node.js 集成能力，可以直接在前端代码中使用 `fs`、`sqlite3`、`child_process` 等 Node.js 模块，实现桌面应用的文件系统操作

5. **图片 Base64 内嵌**: 为了在导出的 HTML 中正确显示图片，将图片文件转为 Base64 编码内嵌到 `<img>` 标签中

6. **项目状态**: 项目自 2017 年创建，至 2020 年已获得近 600 个 star，但作者声明因时间精力有限基本处于放弃状态，欢迎社区 PR

## 构建与运行

### 环境要求

- macOS (主要支持平台)
- Node.js 8.11.3 或 10.16.3
- NW.js 0.32.1 或 0.40.1
- Xcode (用于编译 sqlite3 原生模块)

### 运行步骤

```bash
# 1. 克隆项目
git clone https://github.com/tsycnh/WeChatExporter

# 2. 安装依赖
cd WeChatExporter/development
npm install

# 3. 编译 sqlite3 (针对 node-webkit)
npm install sqlite3 --build-from-source --runtime=node-webkit \
  --target_arch=x64 --target=0.40.1

# 4. 运行应用
/path/to/nwjs.app/Contents/MacOS/nwjs .
```

### 构建发布版本

```bash
# 使用 Grunt 构建
grunt dist

# 或使用 nwjs-builder-phoenix
nwb nwbuild -v 0.40.1-sdk -p osx64 --production ../build/
```

## 相关概念

- [[NW.js]] - 桌面应用框架
- [[AngularJS]] - 前端 MVC 框架
- [[SQLite]] - 嵌入式数据库
- [[SILK Codec]] - Skype/微信语音编码格式
- [[iTunes Backup]] - iOS 设备备份机制
- [[plist]] - Apple 属性列表格式
