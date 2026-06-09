---
tags: [academic, thesis, sar, rf, electromagnetic, comsol, hfss, matlab, alzheimer, autophagy, bioeffect]
category: career
created: 2026-06-09
updated: 2026-06-09
status: active
---

# 学术论文与研究课题索引

## 项目/工具概述

本笔记是研究生阶段（山东建筑大学，学号 212040286-李健）全部学术研究内容的综合索引与知识沉淀。研究核心方向为 **射频电磁辐射（RF EMR）对人体组织的 Specific Absorption Rate (SAR) 评估**，以及射频暴露对神经退行性疾病（阿尔茨海默病）生物效应机制的探索。研究链路覆盖电磁仿真建模（COMSOL Multiphysics、Ansys HFSS）、数值计算（MATLAB）、体外/体内实验设计（混响室暴露系统、细胞/动物模型）、射频电路设计（RFIC、LNA）以及嵌入式软件开发（Qt/C++、Linux Socket 编程）。

源数据位于 OneDrive 云同步目录 `/mnt/c/Users/lijian/OneDrive/lijian/`，涵盖 46+ 篇分类文献、多个仿真工程文件、周报/组会记录及课程资料。

## 技术栈 / 关键特性

| 类别 | 工具/技术 | 说明 |
|------|-----------|------|
| 电磁仿真 | COMSOL Multiphysics (RF Module) | 人体头部 SAR 仿真，多物理场耦合 |
| 电磁仿真 | Ansys HFSS | 矩形谐振腔 540x500x580mm 全波仿真 |
| 数值计算 | MATLAB | 矩阵运算、微分方程、信号处理 |
| 电路仿真 | Multisim | 射频电路原理验证 |
| GUI 开发 | Qt 5.9 (MinGW 32bit) | 远程桌面控制软件 QTdesk |
| 网络编程 | C / Linux Socket | TCP client-server 通信模型 |
| 科学绘图 | Origin 2018 | 数据可视化与分析 |
| 3D 建模 | Fusion 360 | 谐振腔/暴露装置 CAD 建模 |
| 文档 | LaTeX / Word | 论文撰写与周报 |

## 架构与设计

### 研究整体架构

```
电磁仿真层 (COMSOL + HFSS)
    |
    v
SAR 值计算与剂量学评估
    |
    +---> 动物实验 (混响室/Ferris-Wheel 暴露系统)
    |         |
    |         +---> 氧化应激指标 (MDA, SOD, GSH)
    |         +---> DNA 损伤 (8-oxoG, 彗星实验)
    |         +---> 认知功能 (Morris 水迷宫)
    |
    +---> 细胞实验 (体外径向传输线暴露)
    |         |
    |         +---> Neuro-2a / 原代神经元
    |         +---> 自噬通路 (mTOR, Beclin-1, LC3-II/LC3-I)
    |         +---> 炎症反应 (STAT3, IL-6, TNF-α)
    |
    +---> 生物效应机制
              |
              +---> 阿尔茨海默病 (Aβ 聚集, tau 蛋白)
              +---> 氧化应激假说 (ROS, 8-oxoG)
              +---> 自噬功能障碍
              +---> 深脑刺激 (DBS, 帕金森病模型)
```

### COMSOL SAR 仿真数据流

COMSOL 输出的 SAR 插值数据文件 `sar_in_human_head_interp.zh_CN.txt` 包含三维空间网格（55 x 50 x 100 点），每层 55 个采样值，覆盖 x 方向 [-80mm, +80mm]，脑组织区域典型 SAR 值约 1028-1100 W/kg（归一化），热点区域可达 2000-2500+ W/kg。

## 核心知识点

### SAR 基础与评估标准

- **定义**：SAR (Specific Absorption Rate) = σ|E|²/ρ，单位 W/kg
- **FCC 限值**：局部 1g 组织 SAR ≤ 1.6 W/kg
- **ICNIRP 限值**：局部 10g 组织 SAR ≤ 2.0 W/kg
- **研究频率**：900 MHz、1800 MHz、2.4 GHz、2.45 GHz、890 MHz、8.2 GHz、9.9 GHz

### 阿尔茨海默病与射频辐射的关联

- **氧化应激假说**：RF-EMR 可能通过诱导 ROS 加速 Aβ (Amyloid-Beta) 蛋白聚集
- **DNA 损伤**：900/1800 MHz RF-EMF 导致神经细胞 8-oxoG 碱基氧化损伤
- **tau 蛋白**：tau seed 在脑内的积累速率决定 AD 发展进程（参考 "In vivo rate-determining steps of tau seed accumulation"）
- **关键基因**：APP、PS-1 突变与氧化应激关联
- **红外光谱分析**：ATR-FTIR 用于 AD 患者外周血单核细胞和血浆的振动光谱检测
- **近红外荧光探针**：用于 Aβ 斑块活体成像（NIR fluorescent probes for amyloid plaques）
- **NIRS 脑氧监测**：fNIRS 检测 AD 患者顶叶脑氧合变化

### 自噬通路 (Autophagy)

- mTOR 通路、Beclin-1、LC3-II/LC3-I 比值是自噬活性的关键标志物
- 自噬功能障碍是 AD 的核心病理机制之一
- 研究射频暴露是否通过影响自噬通路参与神经退行性病变

### 深脑刺激 (DBS) 与帕金森病

- 延迟反馈控制抑制病理性振荡（Neural Mass Model）
- 噪声诱导改善帕金森状态（计算神经科学方法）
- 实时神经形态系统（Conductance-Based Spiking Neural Networks）

### 射频电路设计

- CMOS RFIC 多级放大器级间耦合方法
- 射频 LNA 优化设计与 ESD 防护
- 高功率微波宽带波导定向耦合器设计

### 体外暴露系统设计

- **径向传输线**：大规模培养瓶微波辐照（900 MHz / 1800 MHz / 2.45 GHz）
- **混响室**：全身体小动物暴露，SAR 分布均匀性优于传统波导
- **Ferris-Wheel**：标准化小鼠 RF 暴露剂量学

## 关键代码/配置片段

### MATLAB -- 矩阵范数与特征值分析

```matlab
format rat;
a = [1,0,0; -1,2,1; 0,-1,-1];
c = inv(a);              % 逆矩阵
b = [-1,1,0; -6,5,2; 3,-3,-2];
A = c * b;               % 所求 A 矩阵
A_T = A';
TEMP = A_T * A;
A_1 = norm(A, 1);        % 一范数
lanbo = eig(TEMP);        % 特征值
A__2 = norm(A);           % 二范数 (谱范数)
```

### MATLAB -- 矩阵指数与微分方程求解

```matlab
syms x y t;
A = [-1,1,0; -4,3,0; 1,0,2];
X_0 = [0;1;2];
A_F_1 = expm(t * A);                          % 矩阵指数 e^(At)
C = expm(-t * A);                              % e^(-At)
y = dsolve('Dy+2*x*y=x*exp(-x^2)', 'x');      % 符号微分方程
```

### C -- Linux Socket TCP Client

```c
#define MYPORT 8887
#define BUFFER_SIZE 1024

int sock_cli = socket(AF_INET, SOCK_STREAM, 0);
struct sockaddr_in servaddr;
servaddr.sin_family = AF_INET;
servaddr.sin_port = htons(MYPORT);
servaddr.sin_addr.s_addr = inet_addr("127.0.0.1");
connect(sock_cli, (struct sockaddr *)&servaddr, sizeof(servaddr));

while (fgets(sendbuf, sizeof(sendbuf), stdin) != NULL) {
    send(sock_cli, sendbuf, strlen(sendbuf), 0);
    if (strcmp(sendbuf, "q\n") == 0) break;
    recv(sock_cli, recvbuf, sizeof(recvbuf), 0);
    fputs(recvbuf, stdout);
}
```

### Qt/C++ -- QTdesk 远程桌面多线程网络架构

```cpp
// 被控端 PassiveHandler - 子线程创建 TCP Socket
void QnyWindow::startPassiveConnect(DeviceInfo *info) {
    QThread *thread = new QThread;
    m_passive_netWorkHandler = new PassiveHandler();
    m_passive_netWorkHandler->initInfo(info, NetWorkHandler::PASSIVE);
    connect(thread, &QThread::started,
            m_passive_netWorkHandler, &PassiveHandler::createSocket);
    m_passive_netWorkHandler->moveToThread(thread);
    thread->start();
}

// NetWorkHandler::createSocket() - TCP 连接建立
void NetWorkHandler::createSocket() {
    m_tcpSocket = new QTcpSocket(this);
    connect(m_tcpSocket, &QTcpSocket::connected,
            this, &NetWorkHandler::handlerConnected);
    m_tcpSocket->connectToHost(remoteHost, remotePort);
}
```

### COMSOL SAR 数据格式示例

```
% Grid (x-axis, 55 points, -80mm to +80mm)
-8.00000e-002  -7.70370e-002  ...  7.70370e-002  8.00000e-002
% Data (brain tissue SAR, W/kg)
1.07769e+003   1.09150e+003   1.08963e+003   ...  1.09610e+003
% 热点区域示例 (靠近天线侧)
2.45489e+003   2.50053e+003   2.20580e+003   2.32166e+003
```

## 使用方法 / 构建步骤

### 环境准备

1. **COMSOL Multiphysics**：安装 RF Module，打开 `.mph` 模型文件
2. **Ansys HFSS**：打开 `.aedt` 工程文件，配置求解频率和网格
3. **MATLAB**：运行 `.m` 脚本进行矩阵计算和数据处理
4. **Qt 5.9**：使用 Qt Creator 打开 `.pro` 文件，选择 MinGW 32bit 编译器
5. **GCC (Linux)**：编译 Socket 程序

### 编译 Socket 示例

```shell
gcc -o server server.c
gcc -o client client.c
# 终端 1: ./server
# 终端 2: ./client
# 输入: +,100,200 → 返回计算结果
# 输入: q → 断开连接
```

### Qt 项目构建

```shell
# 使用 Qt Creator 打开 qtdesk.pro
# 选择 Kit: Desktop Qt 5.9.0 MinGW 32bit
# 构建目录: build-qtdesk-Desktop_Qt_5_9_0_MinGW_32bit-Debug
# 配置文件: config.ini (REMOTE_DESKTOP_SERVER.remoteHost / remotePort)
```

### COMSOL SAR 仿真工作流

1. 导入人体头部几何模型（皮肤/颅骨/脑脊液/灰质/白质）
2. 赋值各组织层介电常数 (εr) 和电导率 (σ)
3. 设置平面波或天线近场激励
4. 自适应网格剖分（组织交界面加密）
5. 频域求解 → 后处理计算 1g/10g 局部平均 SAR

### HFSS 谐振腔仿真工作流

1. 建模矩形腔体 540x500x580mm
2. 波端口或探针激励
3. 本征模式分析 → S 参数 → E/H 场可视化

## 关键文献清单（46+ 篇，按主题分类）

### SAR 评估与暴露系统 (6 篇)
- High peak SAR exposure unit with tight exposure and environmental control for in vitro experiments at 1800 MHz
- In vitro exposure systems for RF exposures at 900 MHz
- models.rf.sar_in_human_head.zh_CN.pdf (COMSOL 官方文档)
- Numerical Techniques for SAR Assessment of Small Animals in Reverberation Chamber
- Development and Validation of Reverberation-Chamber Type Whole-Body Exposure System
- Radiofrequency Dosimetry for the Ferris-Wheel Mouse Exposure System

### 动物实验文献 (6 篇)
- A reverberation chamber to investigate the possible effects of in vivo exposure of rats to 1.8GHz EMF
- Whole-body new-born and young rats' exposure assessment in a reverberating chamber at 2.4 GHz
- Biochemical Changes in Rat Brain Exposed to Low Intensity 9.9 GHz Microwave Radiation
- Impact of Cerebral Radiofrequency Exposures on Oxidative Stress and Corticosterone in a Rat Model of AD
- Effects of far infrared light on Alzheimer's disease-transgenic mice
- In vivo rate-determining steps of tau seed accumulation in Alzheimer's disease

### 细胞实验与 DNA 损伤 (5 篇)
- 8-oxoG DNA Glycosylase-1 Inhibition Sensitizes Neuro-2a Cells to Oxidative DNA Base Damage Induced by 900 MHz RF-EMR
- Cell Type-Dependent Induction of DNA Damage by 1800 MHz RF-EMF
- Differential Pro-Inflammatory Responses of Astrocytes and Microglia Involve STAT3 Activation in Response to 1800 MHz RF Fields
- A mechanistic link between oxidative stress and membrane mediated amyloidogenesis
- In vivo imaging of reactive oxygen species specifically associated with thioflavine S-positive amyloid plaques

### 阿尔茨海默病诊断与检测 (15+ 篇)
- A light therapy for treating Alzheimer's disease / Near infrared light therapy for treating AD
- Development of Near-Infrared Fluorescent Probes for Use in AD Diagnosis
- In Vivo Brain Imaging of Aβ Aggregates in AD with a Near-Infrared Fluorescent Probe
- Progress of Near-Infrared Fluorescent Probes for β-Amyloid Plaques in the Brain
- Smart optical probes for near-infrared fluorescence imaging of AD pathology
- Automated Thresholding Method for fNIRS-Based Functional Connectivity Analysis
- Dynamic cortical connectivity alterations associated with AD: An EEG and fNIRS integration study
- Early rapid diagnosis of AD based on fusion of near- and mid-infrared spectral features
- Vibrational spectroscopic analysis of peripheral blood plasma of patients with AD
- Infrared spectroscopic analysis of mononuclear leukocytes in peripheral blood from AD patients
- Synchrotron-Based FTIR Microspectroscopy Study on the Effect of AD Aβ Aggregates on PC12 Cells
- Raman Imaging Reveals Accumulation of Hemoproteins in Plaques from AD Tissues
- Search for biomarkers of AD: Recent insights, current challenges and future prospects
- The Symptom Classification of AD Based on Machine Learning: A fNIRS Study

### 自噬与氧化应激 (4 篇)
- Oxidative Damage Is the Earliest Event in Alzheimer Disease
- Autophagy and Alzheimer's Disease: From Molecular Mechanisms to Therapeutic Implications
- Targeting Autophagy for the Treatment of AD: Challenges and Opportunities
- High ability of apolipoprotein E4 to stabilize amyloid-β peptide oligomers

### 深脑刺激 (3 篇)
- Delayed Feedback-Based Suppression of Pathological Oscillations in a Neural Mass Model
- Mathematical Modeling for Description of Oscillation Suppression Induced by DBS
- Noise-Induced Improvement of the Parkinsonian State: A Computational Study

### 射频电路设计 (5 篇)
- CMOS 射频集成电路现状与进展分析
- 射频低噪声放大器的优化设计方法研究
- 射频集成电路的模拟技术
- CMOS RFIC 中多级放大器级间耦合方法
- 高功率微波宽带波导定向耦合器设计

## 课程记录

| 课程 | 方向 | 资料位置 |
|------|------|----------|
| 矩阵论 | 数学基础 | YANJIUSHENG/课程/GU_矩阵论重点简述.docx |
| 数字信号处理 | 信号 | YANJIUSHENG/课程/数字信号处理期末.docx |
| 天线理论 | 电磁场 | YANJIUSHENG/课程/天线.doc |
| 嵌入式系统 | 硬件 | YANJIUSHENG/课程/嵌入式.docx + 嵌入式大作业/ |
| 微波技术 | 射频 | YANJIUSHENG/课程/微波大作业.docx |
| 电子材料与器件 | 材料 | YANJIUSHENG/课程/电子材料与器件.doc |

## 文件路径索引

```
OneDrive/lijian/
├── HFSS/                              # 矩形谐振腔仿真 (540x500x580mm)
│   ├── 矩形谐振腔-540-500-580Project1.aedt
│   └── *.aedtresults/                 # S 参数、E/H 场结果
├── MATLAB/LIANXITI_2/                 # 矩阵运算练习代码
├── comsol/                            # COMSOL SAR 仿真工程
│   ├── SAR_1.mph
│   └── 21-2.mph
├── YANJIUSHENG/
│   ├── COMSOL/COMSOL/SAR/             # 人体头部 SAR 仿真核心
│   │   ├── sar_in_human_head.zh_CN.mph
│   │   ├── sar_in_human_head_interp.zh_CN.txt (2MB, 网格数据)
│   │   └── models.rf.sar_in_human_head.zh_CN.pdf
│   ├── 射频文献/                       # 按子主题分类
│   │   ├── 仿真/ 动物/ 细胞/ 略读/
│   ├── 射频与自噬/                     # 射频-自噬交叉专题
│   ├── 老师_要求下载/                  # AD 诊断/检测文献 (35+ 篇)
│   ├── 46篇/ 50篇文献/ 文献/          # 综合文献合集
│   ├── 天大刘晨/                       # DBS/帕金森文献
│   ├── 元器件/                         # 电子器件数据手册
│   ├── 课程/                           # 研究生课程全部资料
│   ├── 李健-*周报.docx                 # 12 次周报 (2024.9-12)
│   └── 仿真结果.docx                   # COMSOL 仿真结果
├── QT/
│   ├── QTdesk/                         # 远程桌面控制项目 (Qt 5.9)
│   │   └── qtdesk/                     # 源码: main.cpp, qnywindow.*, common/
│   ├── learn/ learn_1/ QW/            # Qt 学习项目
│   └── netcat-win32-1.12/             # 网络调试工具
├── MUltisim/                          # 电路仿真 (6 份仿真报告)
├── Fusion 360/                        # 3D CAD 模型
└── *.pdf (根目录)                      # 关键文献 PDF
```

## 相关笔记

- [[SAR-研究生课题-射频电磁辐射研究]] -- 本课题的详细技术笔记（含完整文献引用与实验设计）
- [[comsol]] -- COMSOL 压电仿真项目笔记
- [[comsol-batch]] -- COMSOL 批量仿真自动化笔记
- [[hfss]] -- HFSS 天线仿真项目笔记
- [[qt-projects]] -- Qt 项目汇总（含 QTdesk 远程桌面、LinuxCNC CAM）
- [[qt-linuxcnc]] -- Qt + LinuxCNC 数控加工相关
- [[lijianResume]] -- 李健简历（嵌入式软件工程师）
