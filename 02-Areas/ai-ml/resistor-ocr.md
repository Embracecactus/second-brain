---
tags:
  - ocr
  - resistor
  - ml
  - dataset
  - aoI
  - detection
  - recognition
category: ai-ml
created: 2026-06-09
project: resistorOcr
version: v2/v3
framework: PaddleOCR
---

# 电阻 OCR 数据集项目笔记

## 项目概述

本项目是一个基于 **OCR (Optical Character Recognition)** 技术的 **电阻丝印识别** 数据集，用于训练和评估自动识别电阻表面丝印编码的深度学习模型。项目包含两个版本的数据集（resistorOcr2 和 resistorOcr3），数据格式兼容 **PaddleOCR** 框架的标注规范。

**项目目标：** 通过机器学习模型自动识别电阻表面的丝印数字/字母编码（如 `103`、`472`、`2R00`、`10R0` 等），将电阻丝印转换为可读的阻值信息。

**数据来源路径：**
- resistorOcr2: `/mnt/c/Users/lijian/Downloads/resistorOcr2/resistorOcr2/`
- resistorOcr3: `/mnt/c/Users/lijian/Downloads/resistorOcr3/`

**关联项目路径：**
- `D:\CodeBase\AI_DataSet\resistors\` (原始数据集)
- `D:\CodeBase\AOI\training\assets\images\crop\resistors\` (AOI 训练数据)
- `D:\CodeBase\AOIOLD\training\assets\images\crop\resistors\` (旧版 AOI 数据)

---

## 关键知识点

### 1. 数据集规模

| 数据集 | 训练集 (train) | 验证集 (val) | 图片总数 |
|--------|---------------|-------------|---------|
| resistorOcr2 | 1,168 条 | 535 条 | 1,704 张 |
| resistorOcr3 | 1,187 条 | 515 条 | 1,702 张 |

### 2. 标注格式说明

采用 **PaddleOCR** 标准标注格式，每行包含：

```
图片路径\t[JSON标注数组]
```

JSON 标注结构：
```json
{
  "transcription": "103",           // 丝印文本（识别目标）
  "points": [[x1,y1],[x2,y2],[x3,y3],[x4,y4]],  // 四点定位框（左上、右上、右下、左下）
  "difficult": false                // 是否为困难样本
}
```

### 3. 电阻丝印编码规则

数据集中出现的丝印编码遵循 **EIA-96 标准** 和传统三位/四位数标识法：

**三位数标识法（3-digit marking）：**
- 前两位为有效数字，第三位为 10 的幂次
- 示例：`103` = 10 * 10^3 = 10kΩ，`472` = 47 * 10^2 = 4.7kΩ，`101` = 10 * 10^1 = 100Ω

**四位数标识法（4-digit marking）：**
- 前三位为有效数字，第四位为 10 的幂次
- 示例：`1001` = 100 * 10^1 = 1kΩ，`4701` = 470 * 10^1 = 4.7kΩ

**带 R 的标识法（小数点表示）：**
- `R` 表示小数点位置
- 示例：`2R00` = 2.00Ω，`10R0` = 10.0Ω，`4R70` = 4.70Ω，`1R20` = 1.20Ω

**EIA-96 标识法：**
- 三位编码：两位数字 + 一个字母
- 示例：`01A` = 100Ω，`01B` = 102Ω，`01C` = 105Ω，`30B` = 200Ω，`30C` = 205Ω
- 字母对应乘数：A=10^0, B=10^1, C=10^2, D=10^3, E=10^4, F=10^5, X=10^-1, Y=10^-2, Z=10^-3

**特殊值：**
- `0` = 0Ω（跳线电阻）

### 4. 数据增强策略

从 `Label电阻紧贴.txt` 与 `Label.txt` 的对比可以看出，项目采用了 **"紧贴"标注** 策略——bounding box 更紧密地贴合文字区域，减少背景干扰。这是 OCR 任务中常见的数据增强/标注优化手段。

---

## 技术细节

### 1. 文件结构

```
resistorOcr2/resistorOcr2/
├── train.txt                    # PaddleOCR 格式训练集标注
├── val.txt                      # PaddleOCR 格式验证集标注
└── resistors/
    ├── *.jpg / *.png            # 电阻图片（1,704 张）
    ├── Label.txt                # 完整标注文件（1,704 条）
    ├── Label_backup.txt         # 标注备份（含噪声数据）
    ├── Label电阻紧贴.txt         # 紧贴标注版本（1,540 条）
    ├── fileState.txt            # 文件状态标记（1,908 条）
    ├── fileState_backup.txt     # 文件状态备份（含旧路径）
    ├── fileState_紧贴backup.txt  # 紧贴版文件状态备份
    ├── rec_gt.txt               # Recognition Ground Truth（1,708 条）
    ├── rec_gt_eval.txt          # 识别评估集（362 条）
    └── rec_gt_train.txt         # 识别训练集（2,799 条）

resistorOcr3/
├── train.txt                    # PaddleOCR 格式训练集标注（1,187 条）
├── val.txt                      # PaddleOCR 格式验证集标注（515 条）
└── resistors/
    └── *.jpg / *.png            # 电阻图片
```

### 2. 标注格式对比

| 文件 | 格式 | 用途 |
|------|------|------|
| `train.txt` / `val.txt` | `图片路径\t[JSON]` | PaddleOCR 检测+识别训练 |
| `Label.txt` | `图片路径\t[JSON]` | 完整标注（含四点坐标） |
| `rec_gt.txt` | `crop_img/xxx.jpg\t文本` | 纯识别任务 Ground Truth |
| `fileState.txt` | `绝对路径\t状态码` | 文件管理/状态追踪 |

### 3. 图片来源分类

数据集中的图片来自多个来源：
- **哈希命名图片**（如 `0a0a601c2e869a9cfbf0a6bc502024fd.jpg`）：来自网络爬取或批量采集
- **时间戳命名图片**（如 `1638782063214.jpg`）：来自相机拍摄，时间戳约为 2021年12月
- **编号命名图片**（如 `1.JPG`, `1002.JPG`）：手工拍摄/整理
- **XWD 系列图片**（如 `image9_XWD_T.png`, `image103_XWD_B.png`）：来自 XWD 数据源，`_T` 和 `_B` 可能表示 Top/Bottom 视角

### 4. 标注质量问题

从 `Label_backup.txt` 中发现的噪声数据：
- 非电阻标注混入：如 `皖D`、`皖A`、`皖HC`（车牌识别数据混入）
- 短文本噪声：如 `23`、`UIA`、`E82`
- 旋转标注：部分标注点坐标非轴对齐（如 `[[44.0, 0.0], [177.0, 7.0], [171.0, 76.0], [38.0, 67.0]]`）

这些数据在后续版本（`Label.txt`）中已被清理。

---

## 代码/配置片段

### PaddleOCR 训练数据格式示例

```
# train.txt 每行格式
resistors/1638782063214.jpg	[{"transcription": "1001", "points": [[49, 19], [78, 19], [78, 42], [49, 42]], "difficult": false}]
```

### Recognition GT 格式示例

```
# rec_gt.txt 每行格式（裁剪后的图片 + 文本标签）
crop_img/0a0a601c2e869a9cfbf0a6bc502024fd_crop_0.jpg	102
crop_img/0a01d1d07dde9117be45f99ca71483de_crop_0.jpg	2R00
crop_img/6_crop_0.jpg	4R70
```

### 电阻阻值解码参考

```python
def decode_resistor_marking(marking: str) -> float:
    """解码电阻丝印为阻值（单位：Ω）"""
    if 'R' in marking:
        # 带 R 的标识：R 表示小数点
        return float(marking.replace('R', '.'))

    if len(marking) == 3 and marking[2].isdigit():
        # 三位数标识：前两位 * 10^第三位
        return int(marking[:2]) * (10 ** int(marking[2]))

    if len(marking) == 4 and marking[3].isdigit():
        # 四位数标识：前三位 * 10^第四位
        return int(marking[:3]) * (10 ** int(marking[3]))

    if len(marking) == 3 and marking[2].isalpha():
        # EIA-96 标识：两位数字 + 字母乘数
        code = int(marking[:2])
        multiplier_map = {
            'Z': 0.001, 'Y': 0.01, 'X': 0.1,
            'A': 1, 'B': 10, 'C': 100,
            'D': 1000, 'E': 10000, 'F': 100000
        }
        base_values = [
            100, 102, 105, 107, 110, 113, 115, 118, 121, 124,
            127, 130, 133, 137, 140, 143, 147, 150, 154, 158,
            162, 165, 169, 174, 178, 182, 187, 191, 196, 200,
            205, 210, 215, 221, 226, 232, 237, 243, 249, 255,
            261, 267, 274, 280, 287, 294, 301, 309, 316, 324,
            332, 340, 348, 357, 365, 374, 383, 392, 402, 412,
            422, 432, 442, 453, 464, 475, 487, 499, 511, 523,
            536, 549, 562, 576, 590, 604, 619, 634, 649, 665,
            681, 698, 715, 732, 750, 768, 787, 806, 825, 845,
            866, 887, 909, 931, 953, 976
        ]
        return base_values[code - 1] * multiplier_map[marking[2]]

    if marking == '0':
        return 0.0

    return float(marking)  # fallback
```

---

## 数据集丝印值分布统计

从标注数据中提取的主要丝印值类别：

| 类型 | 示例 | 含义 |
|------|------|------|
| 三位数 | `100`, `101`, `102`, `103`, `104`, `105` | 标准 3-digit 标识 |
| 四位数 | `1001`, `1003`, `1202`, `1302`, `4701` | 标准 4-digit 标识 |
| R 型 | `2R00`, `10R0`, `4R70`, `1R20`, `39R0` | 小数点标识 |
| EIA-96 | `01A`, `01B`, `01C`, `30B`, `30C`, `01X`, `01E` | EIA-96 标准 |
| 零值 | `0` | 跳线/零欧姆电阻 |
| 其他 | `220`, `470`, `681`, `334`, `242` | 常见阻值 |

---

## 相关链接

- **PaddleOCR 官方文档**: https://github.com/PaddlePaddle/PaddleOCR
- **PaddleOCR 数据标注格式**: https://github.com/PaddlePaddle/PaddleOCR/blob/main/doc/doc_en/dataset/datasets_en.md
- **EIA-96 电阻标识标准**: https://en.wikipedia.org/wiki/E_series_of_preferred_numbers
- **AOI (Automated Optical Inspection)**: 自动光学检测，本项目的工业应用场景

---

## 备注

1. 数据集经历了多次迭代：从 `fileState_backup.txt`（2,665 条）到 `fileState.txt`（1,908 条），说明进行了数据清洗
2. `Label_backup.txt` 中混入了车牌识别数据（`皖D`、`皖A`），已在正式版中移除
3. resistorOcr3 相比 resistorOcr2 增加了更多 EIA-96 编码的样本（如 `01C`, `30B`, `30C` 等）
4. 图片时间戳显示数据采集时间为 2021年11月至2022年5月
5. 数据集同时包含检测（detection）和识别（recognition）两种任务的标注
