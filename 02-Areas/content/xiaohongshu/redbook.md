---
tags: [xiaohongshu, tutorial, iot]
category: content/xiaohongshu
created: 2026-06-09
---
Here is the Obsidian markdown note:

---
```markdown
---
title: "Redbook - 小红书技术内容创作素材库"
category: content/xiaohongshu
tags:
  - IoT
  - ESP32
  - 嵌入式
  - MQTT
  - OTA
  - Docker
  - 全志H618
  - 联犀平台
  - C语言
  - 静态库
  - 动态库
created: 2026-06-09
status: active
---

# Redbook - 小红书技术内容创作素材库

## 项目概述

这是一个面向小红书（Redbook）平台的技术内容创作素材仓库，包含多个嵌入式开发、IoT 物联网接入、Docker 容器化开发等主题的图文教程草稿。每个子目录对应一篇独立的技术内容，涵盖从硬件移植到云平台对接的完整链路。

## 技术栈

- **ESP-IDF** (Espressif IoT Development Framework) - ESP32 开发框架
- **MQTT 5.0** - 设备与云平台的通信协议
- **Docker** - 使用 `espressif/idf:release-v5.1` 镜像进行 ESP32 交叉编译
- **联犀物联网平台 (UnitedRhino)** - 开源 IoT 云平台，支持设备管理与 OTA 升级
- **全志 H618 SoC** - ARM Cortex-A53 四核处理器，用于电视盒子刷机
- **C 语言 / JSON (cJSON)** - 嵌入式固件开发与数据解析
- **Ubuntu 18.04** - Docker 基础镜像

## 内容架构

### 1. `lianximqtt/` - 联犀物联网平台接入教程

核心主题：ESP32 设备接入联犀（UnitedRhino）开源云平台，实现数据上报与 OTA 远程升级。

**关键流程：**
1. 在联犀云平台创建产品与设备
2. 配置物模型（Thing Model）
3. 生成 MQTT 三元组（Client ID / Username / Password）
4. 设备端使用 MQTT 5.0 协议连接云平台
5. 通过 OTA 升级包管理实现远程固件更新

**核心代码模式 - MQTT 客户端配置：**
```c
esp_mqtt_client_config_t mqtt5_cfg = {
    .broker.address.uri = CONFIG_BROKER_URL,
    .session.protocol_ver = MQTT_PROTOCOL_V_5,
    .network.disable_auto_reconnect = false,
    .credentials.client_id = client_id,
    .credentials.username = username,
    .credentials.authentication.password = password,
    .session.last_will.topic = "/topic/will",
    .session.last_will.msg = "device offline",
    .session.last_will.qos = 1,
    .session.last_will.retain = true,
};
```

**OTA 升级指令解析模式：**
```c
cJSON *root = cJSON_ParseWithLength(data, len);
cJSON *method = cJSON_GetObjectItem(root, "method");
if (method && strcmp(method->valuestring, "upgrade") == 0) {
    handle_property_ota(data, len);
}
```

### 2. `h618aitv/` - H618 电视盒子刷机移植记录

核心主题：基于 LubanCat-A1 SDK 为全志 H618 电视盒子移植 Ubuntu 系统。

**关键技术点：**
- 通过 `binwalk` 解包官方固件，确认使用 `lubancat_a1_defconfig`
- 使用 `strings` 命令提取 DRAM 配置参数（`[dram_select_para]` 段）
- DDR3 时序参数迁移：将原厂 DRAM/PHY 参数替换到 SDK 的 DTS 与初始化表中
- 发现部分驱动为闭源实现（千兆网卡、蓝牙/WiFi、lbc-firmware），主线 Linux/U-Boot 中不可用

**反向分析固件的关键命令：**
```bash
binwalk 全志H618-Linux千兆版.img
strings 全志H618-Linux千兆版.img | grep -A 2000 "[dram_select_para]"
```

### 3. `docker/` - ESP-IDF Docker 开发环境

快速搭建 ESP-IDF 编译环境的容器化方案：

```bash
docker pull docker.1ms.run/espressif/idf:release-v5.1
docker run -it -v $PWD:/project -u $UID -e HOME=/tmp docker.1ms.run/espressif/idf:release-v5.1
cd project && idf.py build
```

### 4. `提词器/` - C 语言代码保护方案

讲解静态库与动态库的概念，用于保护 C 语言核心源代码不被泄露：
- **静态库 (.a / .lib)**：编译时链接，源码完全隐藏，类似"料理包"
- **动态库 (.so / .dll)**：运行时加载，多程序共享，便于升级，类似"中央厨房"
- 构建流程：`.c` -> `.o` -> `.a/.so`，仅分发头文件与库文件

## 设计决策与关键洞察

1. **MQTT 5.0 vs 3.1.1**：项目选择 MQTT 5.0 协议，支持遗嘱消息（Last Will）和更丰富的会话管理
2. **Docker 化交叉编译**：避免本地安装 ESP-IDF 工具链的复杂性，通过 `-v` 挂载实现宿主机与容器的源码共享
3. **OTA 安全设计**：云平台下发升级指令 -> 设备解析 JSON -> 提取固件 URL -> `esp_https_ota()` 下载并校验 -> 重启生效
4. **H618 刷机经验**：DRAM 时序参数是刷机成功的关键，必须从原厂固件中提取并精确迁移
5. **代码保护策略**：通过编译为库文件的方式，在分享功能的同时保护核心实现

## 运行指南

### ESP32 + 联犀平台接入

1. 安装 Docker 并拉取 ESP-IDF 镜像
2. 在联犀平台 (demo.unitedrhino.com) 注册账号、创建产品/设备
3. 生成 MQTT 三元组并配置到固件中
4. `idf.py build && idf.py flash` 编译烧录

### H618 盒子刷机

1. 使用 `binwalk` 解包原厂固件
2. 提取 DRAM 配置参数
3. 在 LubanCat-A1 SDK 中替换 DDR 参数
4. 构建并烧录自定义镜像

## 相关链接

- [[嵌入式开发]]
- [[MQTT 协议]]
- [[Docker 容器化]]
- [[全志 SoC 平台]]
- [[IoT 物联网平台]]
- [[C 语言编程]]
- [[ESP32 开发]]

## 资源

- [联犀云平台文档](https://doc.unitedrhino.com/)
- [联犀体验账户](https://demo.unitedrhino.com/app/core/#/user/user-login?back_url=/app/core/)
- [ESP-IDF 编程指南](https://docs.espressif.com/projects/esp-idf/zh_CN/latest/esp32/)
- [MQTT 5.0 协议规范](https://docs.oasis-open.org/mqtt/mqtt/v5.0/os/mqtt-v5.0-os.html)
```

## 相关笔记

- [[selfMedia]] — 猪猪猪序员自媒体内容制作项目
- [[douyin-creator-skills]] — 抖音创作者技能集合
- [[zhuzhuxu-style]] — 猪猪猪序员口播文案生成器
- [[embedded-blog]] — 嵌入式技术博客
- [[esp32c3]] — ESP32-C3 智能尾灯项目（MQTT 相关）
- [[h618-buildroot]] — H618 完整开发笔记（H618 刷机内容）
- [[esp-idf-v5-guide]] — ESP-IDF v5 开发指南
