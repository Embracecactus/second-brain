---
title: PocketCNC - 五轴CNC仿真模拟器
tags:
  - cnc
  - 5-axis
  - simulation
  - web-app
  - babylonjs
  - gcode
  - kinematics
  - 3d-visualization
category: cnc-cam
created: 2025-06-09
status: in-progress
project-url: https://github.com/pocketnc
related:
  - "[[LinuxCNC]]"
  - "[[G代码编程]]"
  - "[[五轴加工]]"
  - "[[3D可视化]]"
---

# PocketCNC - 五轴CNC仿真模拟器

## 项目概述

**PocketCNC** 是一个基于Web的五轴XYZAC轴CNC仿真模拟器，支持导入机器模型并基于G代码进行逐步仿真。项目包含两个主要部分：基于原始Pocket NC模拟器的逆向工程分析，以及使用现代技术栈重构的新版本。

**项目路径**: `/mnt/c/Users/lijian/workspace/pocktcnc`

---

## 项目结构

```
pocktcnc/
├── pocketcnc/              # 新版重构项目 (React + Babylon.js)
│   ├── src/
│   │   ├── components/     # React组件
│   │   ├── core/           # 核心功能模块
│   │   ├── types/          # TypeScript类型定义
│   │   └── utils/          # 工具函数
│   └── package.json
├── reversed/               # 逆向工程分析
│   ├── kinematics/         # 运动学算法还原
│   ├── constants/          # 常量定义
│   ├── reducers/           # Redux状态管理
│   └── utils/              # G代码解析器
└── 14pocketsim-master/     # 原始Pocket NC模拟器
```

---

## 技术栈

### 新版 (pocketcnc)
- **前端框架**: React 18 + TypeScript
- **3D引擎**: Babylon.js 7.x
- **构建工具**: Vite 5.x
- **状态管理**: Zustand
- **样式方案**: Tailwind CSS + Radix UI
- **数学库**: gl-matrix
- **G代码解析**: gcode-parser
- **包管理**: pnpm

### 原版 (14pocketsim-master)
- **前端框架**: React 16.x
- **状态管理**: Redux 4.x
- **3D引擎**: Three.js
- **UI组件**: Material-UI 4.x
- **构建工具**: Webpack
- **PWA支持**: Workbox

---

## 核心架构

### 1. 五轴运动学引擎

基于LinuxCNC的运动学理论，实现XYZAC配置的正逆运动学计算：

```typescript
// 正运动学：关节空间 → 工作空间
const kinematics = new FiveAxisKinematics(machineConfig);
const workPosition = kinematics.forwardKinematics(jointPosition);

// 逆运动学：工作空间 → 关节空间
const jointPosition = kinematics.inverseKinematics(workPosition);
```

**关键算法**:
- 齐次变换矩阵计算
- 旋转矩阵构建
- 工具方向向量计算
- 角度归一化处理
- 短路径插值

### 2. G代码处理系统

完整的G代码解析、验证和优化流程：

```typescript
const processor = new GCodeProcessor();
const program = processor.parseGCode(gcodeContent);
const validation = processor.validateGCode(program);
const optimized = processor.optimizeGCode(program);
```

**支持的G代码命令**:
- G0/G1: 快速/线性移动
- G2/G3: 圆弧插补
- G90/G91: 绝对/增量坐标
- M3/M4/M5: 主轴控制
- M7/M8/M9: 冷却控制

### 3. 仿真执行引擎

支持多种执行模式的仿真控制：

```typescript
const engine = new SimulationEngine(kinematics);
await engine.loadProgram(program);
await engine.start({
  mode: 'continuous',  // 'continuous' | 'step' | 'debug'
  speed: 1.0,
  enableCollision: true
});
```

**执行模式**:
- **continuous**: 连续执行
- **step**: 单步执行
- **debug**: 调试模式

### 4. 3D可视化系统

基于Babylon.js的实时渲染：

```typescript
const visualizer = new Visualizer3D(canvas);
await visualizer.loadMachineModel(models, machineConfig);
visualizer.updateMachinePosition(jointPosition);
visualizer.createToolpath(points, options);
```

**场景结构**:
```
Scene
├── Camera (Perspective/Orthographic)
├── Lights (Ambient + Directional)
├── Machine Group
│   ├── Base (底座)
│   ├── Y-Axis Group
│   │   └── X-Axis Group
│   │       └── B-Axis Group
│   │           └── A-Axis Group
│   │               └── Z-Axis Group
│   │                   └── Tool
└── ToolPath Group
```

---

## 关键设计决策

### 1. 运动学配置

采用**XYZAC**配置（非XYZBC），这是Pocket NC机床的实际配置：
- X/Y/Z: 线性轴
- A轴: 绕X轴旋转
- C轴: 绕Z轴旋转

### 2. 状态管理策略

**新版**: 使用Zustand进行轻量级状态管理
**原版**: 使用Redux进行复杂状态管理

```typescript
// Zustand store示例
const useSimulationStore = create((set) => ({
  position: { x: 0, y: 0, z: 0, a: 0, c: 0 },
  isRunning: false,
  setPosition: (pos) => set({ position: pos }),
  setRunning: (running) => set({ isRunning: running }),
}));
```

### 3. 3D引擎选择

**新版选择Babylon.js**而非Three.js的原因：
- 更好的TypeScript支持
- 内置物理引擎
- 更活跃的社区支持
- 更好的性能优化工具

### 4. 构建优化

Vite配置中的代码分割策略：

```typescript
// vite.config.ts
manualChunks: {
  'babylonjs': ['@babylonjs/core', '@babylonjs/gui', '@babylonjs/loaders'],
  'react': ['react', 'react-dom'],
}
```

---

## 核心算法实现

### 五轴运动学正解

```typescript
// 关节空间 -> 笛卡尔空间
forwardKinematics(jointPos: JointPosition): WorkPosition {
  const { x, y, z, a, c } = jointPos;
  
  // 计算旋转矩阵
  const SA = Math.sin(a), CA = Math.cos(a);
  const SC = Math.sin(c), CC = Math.cos(c);
  
  // XYZAC配置的旋转矩阵
  const R00 = CA * CC, R01 = -CA * SC, R02 = SA;
  const R10 = SC,      R11 = CC,       R12 = 0;
  const R20 = -SA * CC, R21 = SA * SC,  R22 = CA;
  
  // 工具方向向量
  return {
    position: { x, y, z },
    orientation: { x: SA, y: -CA * SC, z: CA * CC },
    matrix: rotationMatrix
  };
}
```

### 五轴运动学逆解

```typescript
// 笛卡尔空间 -> 关节空间
inverseKinematics(workPos: WorkPosition): JointPosition {
  const { position, orientation } = workPos;
  
  // 从方向向量提取旋转角度
  const a = Math.asin(orientation.z);
  const c = Math.atan2(orientation.y, orientation.x);
  
  // 归一化角度
  const normalizedA = this.normalizeAngle(a);
  const normalizedC = this.normalizeAngle(c);
  
  // 求解关节位置
  const jointPos = this.solveJointPosition(position, normalizedA, normalizedC);
  
  return { x: jointPos.x, y: jointPos.y, z: jointPos.z, a: normalizedA, c: normalizedC };
}
```

### 角度归一化

```typescript
// 归一化角度到 [-π, π]
normalizeAngle(angle: number): number {
  while (angle > Math.PI) angle -= 2 * Math.PI;
  while (angle < -Math.PI) angle += 2 * Math.PI;
  return angle;
}
```

---

## 类型系统设计

### 核心类型定义

```typescript
// 关节空间位置
interface JointPosition {
  x: number; y: number; z: number;
  a: number; c: number;
}

// 工作空间位置
interface WorkPosition {
  position: Vector3;
  orientation: Vector3;
  matrix: mat4;
}

// 机床配置
interface MachineConfig {
  type: MachineType;
  axisOffsets: Vector3;
  limits: Record<Axis, AxisLimits>;
  rapidRate: number;
  feedRate: number;
}
```

### G代码命令类型

```typescript
type CommandType =
  | 'RAPID'          // G0
  | 'LINEAR'         // G1
  | 'ARC_CW'         // G2
  | 'ARC_CCW'        // G3
  | 'DWELL'          // G4
  | 'COORDINATE_SYS' // G54-G59
  | 'PLANE_SELECT'   // G17-G19
  | 'UNIT_MODE'      // G20/G21
  | 'TOOL_CHANGE'    // M6
  | 'SPINDLE'        // M3/M4/M5
  | 'COOLANT'        // M7/M8/M9
  | 'UNKNOWN';
```

---

## 逆向工程分析

### 原始项目结构

原始Pocket NC模拟器（14pocketsim-master）的逆向分析：

**可行性评估**:
| 方面 | 可行性 | 说明 |
|------|--------|------|
| 核心算法 | ✅ 高 | 运动学、G代码解析逻辑完整 |
| 组件结构 | ✅ 中 | 可还原组件层级和Props |
| Redux状态 | ✅ 高 | Actions和Reducers可完整还原 |
| 样式 | ⚠️ 中 | Material-UI结构可识别，但细节丢失 |
| 变量名 | ❌ 低 | 被压缩混淆，需推断 |

### 已还原内容

- ✅ 五轴运动学算法（正解/逆解）
- ✅ G代码解释器（词法分析、语法解析）
- ✅ Redux状态结构
- ✅ React组件层级
- ✅ Three.js渲染管线
- ✅ 自定义Shader（路径渲染）

### 测试结果

**逆向代码测试通过率**: 81.5%

| 模块 | 通过率 | 状态 |
|------|--------|------|
| 常量定义 | 100% | ✅ |
| 运动学算法 | 92% | ✅ |
| G代码解析 | 67% | ⚠️ |
| 集成测试 | 80% | ⚠️ |

---

## 开发环境设置

### 前置要求

- Node.js >= 18.0.0
- pnpm >= 8.0.0

### 安装和运行

```bash
# 进入项目目录
cd /mnt/c/Users/lijian/workspace/pocktcnc/pocketcnc

# 安装依赖
pnpm install

# 开发模式
pnpm dev

# 构建生产版本
pnpm build

# 预览生产版本
pnpm preview
```

### Windows部署

使用提供的PowerShell脚本：

```powershell
# 运行部署脚本
.\deploy-to-windows.ps1
```

脚本会自动：
1. 检查Node.js和pnpm安装
2. 从WSL2复制项目文件
3. 安装依赖
4. 构建项目
5. 启动开发服务器

---

## 项目配置

### Vite配置

```typescript
// vite.config.ts
export default defineConfig({
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
      '@components': path.resolve(__dirname, './src/components'),
      '@core': path.resolve(__dirname, './src/core'),
      '@utils': path.resolve(__dirname, './src/utils'),
    },
  },
  server: {
    host: '0.0.0.0',
    port: 3000,
  },
  build: {
    outDir: 'dist',
    sourcemap: true,
  },
})
```

### TypeScript配置

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "jsx": "react-jsx",
    "strict": true,
    "paths": {
      "@/*": ["./src/*"],
      "@core/*": ["./src/core/*"]
    }
  }
}
```

---

## 开发路线图

### 已完成 ✅

- [x] 项目基础架构
- [x] 核心类型定义
- [x] 五轴运动学引擎
- [x] G代码处理器
- [x] 仿真执行引擎
- [x] 3D可视化系统
- [x] 基础UI框架
- [x] 逆向工程分析

### 进行中 🚧

- [ ] 高级碰撞检测
- [ ] 材料去除模拟
- [ ] 机器模型导入

### 计划中 📋

- [ ] 项目管理系统
- [ ] 性能优化
- [ ] 用户文档
- [ ] 单元测试完善

---

## 关键学习和洞察

### 1. 五轴运动学复杂性

五轴机床的运动学计算比三轴复杂得多，需要处理：
- 旋转轴的耦合效应
- 奇异点处理
- 角度归一化
- 短路径选择

### 2. G代码解析挑战

G代码的状态管理是主要难点：
- 增量/绝对坐标模式切换
- 模态命令保持
- 多行命令分组
- 错误恢复机制

### 3. Web 3D性能优化

在浏览器中实现实时3D渲染需要：
- 合理的代码分割
- 模型LOD管理
- 渲染批处理
- 内存管理

### 4. 逆向工程价值

通过逆向工程可以：
- 理解现有算法实现
- 学习最佳实践
- 验证新实现的正确性
- 加速开发进程

---

## 相关资源

### 参考项目

- [LinuxCNC](https://www.linuxcnc.org/) - 五轴运动学理论基础
- [Babylon.js](https://www.babylonjs.com/) - 强大的3D渲染引擎
- [CNCjs](https://github.com/cncjs/cncjs) - G代码解析器
- [gl-matrix](https://github.com/toji/gl-matrix) - 高性能数学库

### 学习资料

- 五轴机床运动学原理
- G代码编程规范
- Web 3D渲染技术
- React状态管理最佳实践

---

## 代码质量指标

### 测试覆盖率

- 常量定义: 100%
- 运动学算法: 92%
- G代码解析: 80%
- 集成测试: 75%

### 性能指标

- 常量加载: <1ms
- 运动学计算: <1ms/次
- G代码解析: 需优化大文件
- 3D渲染: 60fps目标

---

## 未来改进方向

### 短期（1-2个月）

1. 完善G代码解释器状态管理
2. 实现完整的圆弧插补
3. 添加更多单元测试
4. 优化大文件处理性能

### 中期（3-6个月）

1. 实现高级碰撞检测
2. 添加材料去除模拟
3. 支持更多机床配置
4. 完善用户文档

### 长期（6个月+）

1. 支持多机床协同仿真
2. 添加云端项目管理
3. 实现实时协作功能
4. 移动端适配

---

## 贡献指南

欢迎贡献！请随时提交问题或拉取请求。

### 开发流程

1. Fork项目
2. 创建特性分支
3. 提交更改
4. 推送到分支
5. 创建Pull Request

### 代码规范

- 使用TypeScript严格模式
- 遵循ESLint规则
- 编写清晰的注释
- 保持测试覆盖率

---

## 许可证

MIT License

---

*最后更新: 2025-06-09*
