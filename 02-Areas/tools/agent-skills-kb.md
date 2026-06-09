---
title: Agent Skills 知识库
tags:
  - agent
  - skill
  - iOS
  - computer-vision
  - AVFoundation
  - PyTorch
  - MLOps
date: 2026-06-09
source: /mnt/c/Users/lijian/.agents/skills/
---

# Agent Skills 知识库

## 概述

本笔记提取自 `.agents/skills/` 目录下的两个 Skill 定义文件，涵盖 **iOS 相机诊断** 和 **高级计算机视觉工程** 两大领域。这些 Skill 以 `SKILL.md` 形式定义，供 AI Agent 在执行任务时调用参考。

---

## Skill 1: axiom-camera-capture-diag

> iOS AVFoundation 相机问题的系统化诊断与修复指南

### 基本信息

| 字段 | 值 |
|------|-----|
| 名称 | `axiom-camera-capture-diag` |
| 版本 | 1.0.0 |
| 许可证 | MIT |
| 兼容平台 | iOS 17+, iPadOS 17+, macOS 14+, tvOS 17+ |
| 更新日期 | 2026-01-03 |

### 核心原理

相机问题的根本原因分布：

| 原因类别 | 占比 | 说明 |
|----------|------|------|
| **Threading** | 35% | Session 操作在主线程执行 |
| **Session 生命周期** | 25% | 未启动、中断、未配置 |
| **Rotation** | 20% | 使用已废弃 API、缺少 RotationCoordinator |
| **Permissions** | 15% | 被拒绝、未请求 |
| **Configuration** | 5% | 错误 preset、缺少 input/output |

**关键原则**: 调试 capture 逻辑之前，优先检查线程和 Session 状态。

### 症状速查表

| 症状 | 可能原因 |
|------|----------|
| 预览黑屏 | Session 未启动 / 权限被拒 / 无 camera input |
| 打开相机时 UI 冻结 | `startRunning()` 在主线程调用 |
| 来电后相机冻结 | 未处理 interruption |
| 预览旋转 90 度 | 未使用 RotationCoordinator (iOS 17+) |
| 拍照旋转错误 | 未对 output connection 应用旋转角度 |
| 前置相机照片未镜像 | **这是正确行为**（预览镜像，照片不镜像） |
| "Camera in use by another app" | 其他应用独占访问 |
| 拍照耗时 2 秒以上 | `photoQualityPrioritization` 设为 `.quality` |
| iPad 上 Session 无法启动 | Split View 模式下相机不可用 |
| 旧 iOS 崩溃 | 使用 iOS 17+ API 但未做 availability check |

### 四步诊断流程

1. **检查 Session 状态** -- 确认 `isRunning`、`inputs`、`outputs` 计数
2. **检查线程** -- 确认 session 操作在后台队列而非主线程
3. **检查权限** -- 确认 `AVCaptureDevice.authorizationStatus`
4. **检查中断** -- 注册 `AVCaptureSessionWasInterrupted` 观察者

### 15 个诊断模式 (Diagnostic Patterns)

#### Pattern 1-3: 黑屏/冻结预览

- **Pattern 1 - Session 未启动**: `isRunning = false`，需在 sessionQueue 上调用 `startRunning()`
- **Pattern 2 - 无 Camera Input**: `inputs.count = 0`，检查权限和 `canAddInput()`
- **Pattern 3 - Preview Layer 未连接**: Session 运行正常但预览黑屏，检查 `previewLayer.session`、superlayer、frame

#### Pattern 4: 主线程阻塞

`startRunning()` 是阻塞调用，必须在专用 serial queue 上执行：

```swift
private let sessionQueue = DispatchQueue(label: "camera.session")
sessionQueue.async { session.startRunning() }
```

#### Pattern 5-7: 使用中冻结

- **Pattern 5 - 来电中断**: Session 会自动恢复，只需更新 UI，无需再次调用 `startRunning()`
- **Pattern 6 - Split View**: iPad 多任务模式下相机不可用，需提示用户退出分屏
- **Pattern 7 - 热压力**: 设备过热时系统降低资源，检查 `ProcessInfo.processInfo.thermalState`

#### Pattern 8-10: 旋转问题

- **Pattern 8 - 预览旋转错误**: 使用 `AVCaptureDevice.RotationCoordinator` (iOS 17+) 并 observe 角度变化
- **Pattern 9 - 拍照旋转错误**: 拍照时对 `photoOutput.connection` 应用 `videoRotationAngleForHorizonLevelCapture`
- **Pattern 10 - 前置相机镜像**: 预览镜像但照片不镜像是系统标准行为，如业务需要镜像照片需手动翻转

#### Pattern 11-12: 拍照速度

- **Pattern 11 - 质量优先导致慢**: 将 `photoQualityPrioritization` 从 `.quality` 改为 `.speed` 或 `.balanced`
- **Pattern 12 - 延迟处理**: 启用 `isAutoDeferredPhotoDeliveryEnabled` 实现零快门延迟

#### Pattern 13-15: 权限与兼容

- **Pattern 13 - 未请求权限**: 使用 `AVCaptureDevice.requestAccess(for: .video)` 异步请求
- **Pattern 14 - 权限被拒**: 引导用户前往 Settings
- **Pattern 15 - API 兼容性**: 使用 `@available(iOS 17.0, *)` 做降级处理

### 上线检查清单

- Session 有至少一个 input 和 output
- `isRunning = true`
- Preview layer 连接到 session 且 frame 非零
- 所有 session 操作在 sessionQueue 上
- `startRunning()` 在后台线程
- 权限状态已检查并做相应处理
- RotationCoordinator 已创建并 observe
- 中断观察者已注册并测试来电场景

### 参考资源

- WWDC: 2021-10247, 2023-10105
- 关联 Skill: `axiom-camera-capture`, `axiom-camera-capture-ref`

---

## Skill 2: senior-computer-vision

> 世界级高级计算机视觉工程师 Skill，覆盖生产级 AI/ML 系统

### 基本信息

| 字段 | 值 |
|------|-----|
| 名称 | `senior-computer-vision` |
| 定位 | 生产级计算机视觉系统架构与工程 |

### 核心能力领域

- 图像/视频处理、目标检测、分割、视觉 AI 系统
- 3D 视觉、视频分析、实时处理
- 生产环境部署与优化
- MLOps / DataOps 全流程

### 技术栈

| 类别 | 技术 |
|------|------|
| **Languages** | Python, SQL, R, Scala, Go |
| **ML Frameworks** | PyTorch, TensorFlow, Scikit-learn, XGBoost |
| **Data Tools** | Spark, Airflow, dbt, Kafka, Databricks |
| **LLM Frameworks** | LangChain, LlamaIndex, DSPy |
| **Deployment** | Docker, Kubernetes, AWS/GCP/Azure |
| **Monitoring** | MLflow, Weights & Biases, Prometheus |
| **Databases** | PostgreSQL, BigQuery, Snowflake, Pinecone |

### 三大生产模式 (Production Patterns)

#### Pattern 1: 可扩展数据处理

- 水平扩展架构
- 容错设计
- 实时与批处理
- 数据质量验证
- 性能监控

#### Pattern 2: ML 模型部署

- 低延迟模型服务
- A/B 测试基础设施
- Feature Store 集成
- 模型监控与漂移检测 (drift detection)
- 自动化重训练管线

#### Pattern 3: 实时推理 (Real-Time Inference)

- Batching 与缓存策略
- 负载均衡
- 自动扩缩容 (Auto-scaling)
- 延迟优化
- 成本优化

### 性能目标

| 指标 | 目标值 |
|------|--------|
| P50 延迟 | < 50ms |
| P95 延迟 | < 100ms |
| P99 延迟 | < 200ms |
| 吞吐量 | > 1000 req/s |
| 并发用户 | > 10,000 |
| 可用性 | 99.9% |
| 错误率 | < 0.1% |

### 最佳实践

**开发阶段**: TDD、Code Review、文档即代码、版本控制一切、CI/CD

**生产阶段**: 全面监控、自动化部署、Feature Flags、Canary 发布、结构化日志

**安全合规**: 认证授权、数据加密（静态/传输）、PII 匿名化、GDPR/CCPA 合规、安全审计

### 常用命令

```bash
# 开发
python -m pytest tests/ -v --cov
python -m black src/
python -m pylint src/

# 训练
python scripts/train.py --config prod.yaml
python scripts/evaluate.py --model best.pth

# 部署
docker build -t service:v1 .
kubectl apply -f k8s/
helm upgrade service ./charts/

# 监控
kubectl logs -f deployment/service
python scripts/health_check.py
```

### 参考文档

- `references/computer_vision_architectures.md` -- 架构模式与最佳实践
- `references/object_detection_optimization.md` -- 目标检测优化工作流
- `references/production_vision_systems.md` -- 生产视觉系统技术参考

---

## 两个 Skill 的关联与区别

| 维度 | axiom-camera-capture-diag | senior-computer-vision |
|------|---------------------------|------------------------|
| **领域** | iOS 原生相机开发 | AI/ML 计算机视觉工程 |
| **技术栈** | Swift, AVFoundation | Python, PyTorch, Kubernetes |
| **目标** | 诊断修复相机 bug | 构建生产级视觉 AI 系统 |
| **粒度** | 具体 bug pattern + 修复代码 | 架构模式 + 工程实践 |
| **受众** | iOS 开发者 | ML 工程师 / 数据科学家 |
| **Skill 形式** | 决策树 + 15 个 Pattern | 能力矩阵 + 3 个 Production Pattern |

两者互补：前者解决"相机拍不到"的问题，后者解决"拍到之后怎么用 AI 处理"的问题。
