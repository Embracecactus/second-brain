---
tags: [camera, diagnostic, agent, computer-vision, avfoundation, ios, yolo, pytorch]
category: camera-isp
created: 2026-06-09
updated: 2026-06-09
status: active
source: /mnt/c/Users/lijian/.agents/skills/
---

# Camera Diagnostic & Computer Vision Agent Skills

## 项目/工具概述

本文档汇总了 `/mnt/c/Users/lijian/.agents/skills/` 目录下两个核心 Agent Skill 的完整技术细节。第一个 Skill `axiom-camera-capture-diag` 是一套 iOS AVFoundation 相机系统化诊断指南，覆盖 15 种常见故障模式（黑屏、主线程阻塞、旋转错误、权限问题、来电中断等），内置决策树、症状速查表和上线检查清单，可直接用于生产环境的相机问题排查。第二个 Skill `senior-computer-vision` 是世界级计算机视觉工程能力包，涵盖 PyTorch/OpenCV/YOLO/SAM 技术栈、三大生产模式（可扩展数据处理、ML 模型部署、实时推理）、性能目标（P50 < 50ms, 99.9% 可用性）以及 MLOps 全流程，配有三份参考文档分别聚焦架构设计、目标检测优化和生产视觉系统。

## 技术栈 / 关键特性

### axiom-camera-capture-diag

| 维度 | 内容 |
|------|------|
| 平台 | iOS 17+, iPadOS 17+, macOS 14+, tvOS 17+ |
| 框架 | AVFoundation (AVCaptureSession, AVCaptureDeviceInput, AVCapturePhotoOutput, AVCaptureVideoPreviewLayer) |
| 语言 | Swift |
| 核心 API | RotationCoordinator (iOS 17+), photoQualityPrioritization, deferred photo delivery |
| 诊断覆盖 | 15 种 Pattern, 涵盖 Session 生命周期、线程、旋转、权限、中断、热压力 |
| 参考 WWDC | 2021-10247, 2023-10105 |

### senior-computer-vision

| 维度 | 内容 |
|------|------|
| 语言 | Python, SQL, R, Scala, Go |
| ML 框架 | PyTorch, TensorFlow, Scikit-learn, XGBoost |
| 数据工具 | Spark, Airflow, dbt, Kafka, Databricks |
| LLM 框架 | LangChain, LlamaIndex, DSPy |
| 部署 | Docker, Kubernetes, AWS/GCP/Azure |
| 监控 | MLflow, Weights & Biases, Prometheus |
| 数据库 | PostgreSQL, BigQuery, Snowflake, Pinecone |
| 视觉模型 | YOLO, SAM, Diffusion Models, Vision Transformers |

## 架构与设计

### AVFoundation 相机诊断架构 (axiom-camera-capture-diag)

```
App Layer (SwiftUI/UIKit)
    |
    v
AVCaptureSession (sessionQueue: background serial queue)
    |-- AVCaptureDeviceInput (.video)
    |-- AVCapturePhotoOutput
    |-- AVCaptureVideoPreviewLayer
    |
    v
NotificationCenter (interruption / runtime error observers)
    |
    v
RotationCoordinator (iOS 17+, observe angle changes)
```

核心设计原则：
- **Session 所有操作必须在后台 serial queue 执行**，`startRunning()` 是阻塞调用，主线程执行会导致 UI 冻结 1-3 秒
- **故障概率分布**：Threading 35% > Session lifecycle 25% > Rotation 20% > Permissions 15% > Configuration 5%
- **诊断优先级**：先检查线程和 Session 状态，再排查 Capture 逻辑

### 计算机视觉生产系统架构 (senior-computer-vision)

```
数据采集 --> 数据处理管道 (Spark/Airflow)
    |
    v
特征工程 --> Feature Store (Pinecone/BigQuery)
    |
    v
模型训练 (PyTorch/TF) --> MLflow/W&B 跟踪
    |
    v
模型服务 (Docker/K8s) --> A/B Testing
    |
    v
监控 (Prometheus/MLflow) --> 自动重训练管道
```

## 核心知识点

### 一、AVFoundation 相机诊断 15 种 Pattern

#### Pattern 1: Session 未启动
- **症状**：黑屏预览，`isRunning = false`
- **根因**：`startRunning()` 未调用 / Session 无 Input / Session 停止后未重启
- **修复**：在 sessionQueue 异步执行，先检查 inputs 是否为空再启动
- **耗时**：10 min

#### Pattern 2: 无相机 Input
- **症状**：`session.inputs.count = 0`
- **根因**：权限被拒 / `AVCaptureDeviceInput` 创建失败 / `canAddInput()` 返回 false
- **修复**：确保权限在创建 input 前已授予，使用 `beginConfiguration()`/`commitConfiguration()` 包裹
- **耗时**：15 min

#### Pattern 3: Preview Layer 未连接
- **症状**：`isRunning = true`，Input 已配置，但预览黑屏
- **根因**：previewLayer.session 未设置 / 不在 view hierarchy / frame 为零
- **修复**：设置 `previewLayer.session = session`，确保 frame 非零（SwiftUI 常见问题）
- **耗时**：10 min

#### Pattern 4: 主线程阻塞
- **症状**：打开相机时 UI 冻结 1-3 秒
- **根因**：`startRunning()` 在主线程调用（阻塞调用）
- **修复**：创建专用 serial queue `DispatchQueue(label: "camera.session")`，所有 session 操作在该 queue 异步执行
- **耗时**：15 min

#### Pattern 5: 来电中断
- **症状**：来电后相机冻结
- **根因**：未处理 `AVCaptureSessionWasInterrupted` 通知
- **关键点**：Session 在中断结束后**自动恢复**，无需再次调用 `startRunning()`，只需更新 UI
- **耗时**：30 min

#### Pattern 6: iPad Split View 相机不可用
- **症状**：进入分屏后相机停止工作
- **根因**：`InterruptionReason.videoDeviceNotAvailableWithMultipleForegroundApps`
- **修复**：显示提示信息，退出分屏后自动恢复
- **耗时**：15 min

#### Pattern 7: 热压力导致降级
- **症状**：长时间使用后相机随机停止
- **根因**：设备过热，系统减少资源分配
- **修复**：监听 `ProcessInfo.processInfo.thermalState`，在 `.serious`/`.critical` 时降低 sessionPreset
- **耗时**：20 min

#### Pattern 8: 预览旋转错误
- **症状**：预览画面旋转 90 度
- **根因**：未使用 RotationCoordinator (iOS 17+)
- **修复**：创建 `RotationCoordinator(device:previewLayer:)`，observe `videoRotationAngleForHorizonLevelPreview` 变化
- **耗时**：30 min

#### Pattern 9: 拍照旋转错误
- **症状**：预览正常，但拍出的照片旋转
- **根因**：拍照时未将旋转角度应用到 photo output connection
- **修复**：在 `capturePhoto()` 前设置 `connection.videoRotationAngle = coordinator.videoRotationAngleForHorizonLevelCapture`
- **耗时**：15 min

#### Pattern 10: 前置相机镜像
- **症状**：设计师反馈"前置相机照片与预览不一致"
- **真相**：这是**正确行为**，不是 Bug。预览镜像（符合用户预期），照片不镜像（文字可读）。若业务需要镜像照片，使用 `.upMirrored` orientation
- **耗时**：5 min (解释) 或 15 min (实现镜像)

#### Pattern 11: 拍照延迟（质量优先）
- **症状**：拍照耗时 2+ 秒
- **根因**：`photoQualityPrioritization = .quality`
- **修复**：社交分享场景用 `.speed`，通用场景用 `.balanced`
- **耗时**：5 min

#### Pattern 12: 延迟处理（零快门延迟）
- **症状**：需要最大响应速度
- **方案**：启用 `isAutoDeferredPhotoDeliveryEnabled = true` (iOS 17+)，delegate 先返回 proxy 供即时显示，稍后返回最终图像
- **耗时**：30 min

#### Pattern 13-15: 权限与兼容性
- Pattern 13: 权限未请求 --> 使用 `AVCaptureDevice.requestAccess(for: .video)` async API
- Pattern 14: 权限被拒 --> 引导用户前往 Settings
- Pattern 15: API 兼容性 --> 使用 `#available(iOS 17.0, *)` guard，回退到 deprecated `videoOrientation`

### 二、计算机视觉生产系统核心知识

#### 三大生产模式

1. **Scalable Data Processing (可扩展数据处理)**
   - 水平扩展架构，容错设计
   - 实时 + 批处理双模
   - 数据质量校验 + 性能监控

2. **ML Model Deployment (ML 模型部署)**
   - 低延迟模型服务
   - A/B Testing 基础设施
   - Feature Store 集成
   - 模型监控与漂移检测
   - 自动重训练管道

3. **Real-Time Inference (实时推理)**
   - Batching 与 Caching 策略
   - 负载均衡 + Auto-scaling
   - 延迟优化 + 成本优化

#### 性能目标

| 指标 | 目标值 |
|------|--------|
| P50 延迟 | < 50ms |
| P95 延迟 | < 100ms |
| P99 延迟 | < 200ms |
| 吞吐量 | > 1000 req/s |
| 并发用户 | > 10,000 |
| 可用性 | 99.9% |
| 错误率 | < 0.1% |

#### 参考文档覆盖范围

- `references/computer_vision_architectures.md` -- 架构设计原则：Production-First Design, Performance by Design, Security & Privacy；高级模式：Distributed Processing, Real-Time Systems, ML at Scale
- `references/object_detection_optimization.md` -- 目标检测优化工作流：算法选择、模型压缩、NMS 优化、TensorRT/ONNX 推理加速
- `references/production_vision_systems.md` -- 生产视觉系统：系统设计原则、部署策略、监控与可观测性

## 关键代码/配置片段

### AVFoundation Session 初始化模板

```swift
private let session = AVCaptureSession()
private let sessionQueue = DispatchQueue(label: "camera.session")

func setupSession() {
    sessionQueue.async { [self] in
        session.beginConfiguration()
        session.sessionPreset = .photo

        // 添加输入
        guard let camera = AVCaptureDevice.default(.builtInWideAngleCamera, for: .video, position: .back),
              let input = try? AVCaptureDeviceInput(device: camera),
              session.canAddInput(input) else { return }
        session.addInput(input)

        // 添加输出
        let output = AVCapturePhotoOutput()
        guard session.canAddOutput(output) else { return }
        session.addOutput(output)

        session.commitConfiguration()
        session.startRunning()
    }
}
```

### RotationCoordinator 配置 (iOS 17+)

```swift
let coordinator = AVCaptureDevice.RotationCoordinator(
    device: camera, previewLayer: previewLayer
)
previewLayer.connection?.videoRotationAngle =
    coordinator.videoRotationAngleForHorizonLevelPreview

observation = coordinator.observe(\.videoRotationAngleForHorizonLevelPreview) {
    [weak previewLayer] coord, _ in
    DispatchQueue.main.async {
        previewLayer?.connection?.videoRotationAngle =
            coord.videoRotationAngleForHorizonLevelPreview
    }
}
```

### 相机诊断 Session 状态打印

```swift
func diagnoseSession() {
    print("Session isRunning: \(session.isRunning)")
    print("Inputs: \(session.inputs.count)")
    print("Outputs: \(session.outputs.count)")
    for input in session.inputs {
        if let deviceInput = input as? AVCaptureDeviceInput {
            print("  Input: \(deviceInput.device.localizedName)")
        }
    }
    let status = AVCaptureDevice.authorizationStatus(for: .video)
    print("Permission: \(status.rawValue)")
    print("Thermal: \(ProcessInfo.processInfo.thermalState.rawValue)")
}
```

### CV 推理服务部署命令

```bash
# 训练
python scripts/train.py --config prod.yaml
python scripts/evaluate.py --model best.pth

# 容器化部署
docker build -t vision-service:v1 .
kubectl apply -f k8s/
helm upgrade vision-service ./charts/

# 监控
kubectl logs -f deployment/vision-service
python scripts/health_check.py

# 测试
python -m pytest tests/ -v --cov
```

## 使用方法 / 构建步骤

### axiom-camera-capture-diag 使用流程

1. **症状识别**：对照 Quick Reference Table 快速定位可能的 Pattern
2. **必做诊断**：按顺序执行 Step 1-4（Session State -> Threading -> Permissions -> Interruptions）
3. **决策树导航**：根据诊断结果进入对应分支，定位到具体 Pattern
4. **按 Pattern 修复**：每个 Pattern 包含诊断代码、修复代码和预估耗时
5. **上线前检查**：执行 Checklist 中 Basics / Threading / Permissions / Rotation / Interruptions 五项全部打勾

### senior-computer-vision 使用流程

1. **数据准备**：使用 Spark/Airflow 构建数据处理管道
2. **模型训练**：PyTorch 训练，MLflow/W&B 跟踪实验
3. **评估优化**：Profile 性能瓶颈，模型压缩（量化/剪枝/蒸馏）
4. **容器化部署**：Docker 打包，K8s 编排，Helm 管理
5. **生产监控**：Prometheus 指标采集，漂移检测，自动重训练触发

## 两个 Skill 对比

| 维度 | axiom-camera-capture-diag | senior-computer-vision |
|------|--------------------------|----------------------|
| 领域 | iOS 相机诊断 | 计算机视觉工程 |
| 平台 | iOS/iPadOS/macOS/tvOS | 跨平台 (Linux/Docker/K8s) |
| 语言 | Swift | Python, Go, Scala |
| 核心框架 | AVFoundation | PyTorch, OpenCV, YOLO |
| 故障模式 | 15 种明确 Pattern + 决策树 | 3 大生产模式 + 最佳实践 |
| 性能目标 | 拍照 < 1s, 零快门延迟 | P50 < 50ms, 99.9% 可用性 |
| 参考资源 | WWDC 2021/2023 | 3 份架构参考文档 |
| 输出产物 | 诊断代码片段 + Checklist | 训练/部署/监控全流程 |

## 相关笔记

- [[myskills]] -- CamSkills 工业相机固件开发技能包
- [[skill]] -- GenICam 属性管理与专家系统
- [[work]] -- SC-Sensor-Driver 专家系统
- [[rv1126b]] -- RV1126B 运动相机项目（Camera ISP 应用）
- [[ffmpeg-learning]] -- FFmpeg 学习（V4L2 视频采集）
