---
tags:
  - ai-ml
  - ocr
  - computer-vision
  - text-detection
  - text-recognition
  - paddleocr
  - deep-learning
category: ai-ml
created: 2026-06-09
status: active
---

# PaddleOCR 文字识别项目

## 项目概述

基于 PaddleOCR 构建的文字识别（OCR）应用，支持中英文及多语言文本检测与识别，集成了图像预处理、多线程批处理和可视化标注等功能。项目使用了 PaddlePaddle 深度学习框架，模型在首次运行时自动从云端下载。

## 技术栈

- **深度学习框架**: PaddlePaddle (PaddlePaddle 2.0+)
- **OCR 引擎**: PaddleOCR (PP-OCRv4, 最新 2.10 版本)
- **编程语言**: Python 3.8+
- **图像处理**: OpenCV (`opencv-python`, `opencv-contrib-python`), Pillow
- **并发处理**: `concurrent.futures.ThreadPoolExecutor`
- **文本检测模型**: DB (Differentiable Binarization)
- **文本识别模型**: CRNN, SVTR_LCNet
- **核心依赖**: numpy, shapely, pyclipper, scikit-image, lmdb, albumentations
- **辅助工具**: cython, pyyaml, tqdm, rapidfuzz

## 架构与设计决策

### OCR 流水线 (Pipeline)

项目采用经典的三阶段 OCR 流水线设计：

1. **文本检测 (Text Detection)**: 使用 DB 算法定位图像中的文字区域，输出边界框 (bounding boxes)
2. **文本方向分类 (Angle Classification)**: 判断文本行是否旋转 180 度，支持自动旋转校正
3. **文本识别 (Text Recognition)**: 使用 CRNN/SVTR 模型将裁剪后的文本图像转换为字符串

### 模型版本管理

PaddleOCR 支持多个模型版本（PP-OCR 到 PP-OCRv4），每个版本对应不同的精度/速度权衡：

- **PP-OCRv4**: 当前默认版本，支持中文、英文、日文、韩文等 20+ 语言
- **PP-StructureV2**: 文档结构分析，支持表格识别、版面分析、公式识别
- 模型首次使用时自动从百度云 (`paddleocr.bj.bcebos.com`) 下载到 `~/.paddleocr/whl/` 目录

### 两个测试脚本的设计差异

| 特性 | `test.py` | `test1.py` |
|------|-----------|------------|
| 语言 | 英文 (`lang='en'`) | 中文 (`lang='ch'`) |
| 角度分类 | 开启 (`use_angle_cls=True`) | 关闭 (`cls=False`) |
| 图像预处理 | 无 | 亮度/对比度增强 (1.5x) |
| 线程数 | 8 | 4 |

## 关键代码片段

### 基础 OCR 调用模式

```python
from paddleocr import PaddleOCR, draw_ocr
from PIL import Image

# 初始化 - 首次运行会自动下载模型
ocr = PaddleOCR(use_angle_cls=True, lang='ch')

# 执行 OCR
result = ocr.ocr(img_path, cls=True)
result = result[0]  # 取第一张图的结果

# 解析结果: [box坐标, (识别文本, 置信度)]
boxes = [line[0] for line in result]
txts = [line[1][0] for line in result]
scores = [line[1][1] for line in result]
```

### 图像预处理增强

```python
from PIL import ImageEnhance

def preprocess_image(image):
    # 提升亮度，适用于暗色背景文档
    enhancer = ImageEnhance.Brightness(image)
    image = enhancer.enhance(1.5)
    # 提升对比度，增强文字与背景的区分度
    enhancer = ImageEnhance.Contrast(image)
    image = enhancer.enhance(1.5)
    return image
```

### 多线程批处理

```python
from concurrent.futures import ThreadPoolExecutor

with ThreadPoolExecutor(max_workers=8) as executor:
    results = list(executor.map(process_image, img_paths))
```

## 关键知识点

1. **PaddleOCR 结果格式**: `ocr.ocr()` 返回嵌套列表，每页为一个列表，每条结果为 `[box, (text, confidence)]`，其中 box 是四个角点的坐标
2. **cls 参数的权衡**: 角度分类器可以识别 180 度旋转的文字，但会增加推理时间；如果确定图像无倒置文字，设为 `cls=False` 可提升性能
3. **图像预处理的重要性**: 对低质量文档图像（如手机拍摄、光照不均），亮度和对比度增强能显著提升识别准确率
4. **rec_image_shape**: PP-OCRv3/v4 使用 `3,48,320` 输入尺寸，旧版本使用 `3,32,320`，直接影响识别精度
5. **多语言支持**: 通过 `lang` 参数切换语言，内部自动映射到对应的检测和识别模型

## 构建与运行

### 安装依赖

```bash
pip install paddlepaddle  # 或 paddlepaddle-gpu (GPU 版本)
pip install paddleocr
```

### 项目依赖安装

```bash
cd /home/lijian/project/OCR/PaddleOCR
pip install -r requirements.txt
```

### 运行 OCR

```bash
# Python API 方式
python test.py    # 英文 OCR
python test1.py   # 中文 OCR (带图像增强)

# 命令行方式
paddleocr --image_dir ./your_image.png --lang ch
paddleocr --image_dir ./your_image.png --type structure  # 文档结构分析
```

### 模型训练 (分布式)

```bash
python3 -m paddle.distributed.launch \
    --log_dir=./debug/ \
    --gpus '0,1,2,3,4,5,6,7' \
    tools/train.py \
    -c configs/rec/rec_mv3_none_bilstm_ctc.yml
```

## 项目文件结构

```
/home/lijian/project/OCR/
├── test.py                  # 英文 OCR 测试脚本
├── test1.py                 # 中文 OCR 测试脚本 (带预处理)
├── 20250417212136.png       # 测试输入图像
├── result.jpg               # 输出结果图像
├── Aaargh.ttf               # 字体文件
└── PaddleOCR/               # PaddleOCR 引擎源码
    ├── paddleocr.py         # 核心 API (PaddleOCR/PPStructure 类)
    ├── __init__.py          # 模块入口
    ├── train.sh             # 训练启动脚本
    ├── setup.py             # 包安装脚本
    ├── pyproject.toml       # 项目配置
    └── requirements.txt     # Python 依赖
```

## 相关概念链接

- [[PaddlePaddle]] - 百度深度学习框架
- [[Computer Vision]] - 计算机视觉
- [[Text Detection]] - 文本检测 (DB, EAST)
- [[Text Recognition]] - 文本识别 (CRNN, SVTR)
- [[Document Understanding]] - 文档理解
- [[Image Preprocessing]] - 图像预处理技术
- [[Deep Learning Model Deployment]] - 模型部署
- [[OCR Pipeline Design]] - OCR 流水线设计模式

## 相关笔记

- [[AX650N]] — AX650N YOLOv5 目标检测模型训练与部署
- [[resistor-ocr]] — 电阻 OCR 数据集项目
- [[camera-diag-skills]] — Camera 诊断技能（计算机视觉）
