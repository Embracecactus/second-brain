---
tags: [python, learning]
category: learning
created: 2026-06-09
---
```yaml
---
title: "Python 学习项目 - 错误处理与调试"
tags:
  - python
  - learning
  - debugging
  - faulthandler
  - error-handling
category: learning
created: 2026-06-09
status: active
---
```

```markdown
# Python 学习项目 - 错误处理与调试

## 项目概述

这是一个 Python 学习项目，专注于探索 Python 的错误处理机制和底层调试技术。项目通过实际代码演示了如何使用 `faulthandler` 模块捕获底层崩溃（如段错误），以及如何自定义异常钩子来记录 Python 异常。

## 技术栈

- **语言**: Python 3.10
- **核心模块**:
  - `faulthandler` - 底层崩溃调试
  - `ctypes` - C 语言接口，用于触发底层错误
  - `sys` - 系统级异常钩子
  - `traceback` - 异常堆栈追踪

## 架构与设计决策

### 错误分层处理

项目展示了 Python 错误的两个层次：

1. **底层崩溃（Fatal Errors）**: 由 `faulthandler` 模块捕获，记录到 `crash.log`
2. **Python 异常**: 通过自定义 `sys.excepthook` 捕获，记录到 `error.log`

### 关键设计

```python
# 启用 faulthandler，将崩溃日志写入 crash.log
faulthandler.enable(file=open('crash.log', 'w'))

# 自定义异常钩子，记录 Python 异常到 error.log
def exception_hook(exc_type, exc_value, exc_traceback):
    with open('error.log', 'w') as f:
        traceback.print_exception(exc_type, exc_value, exc_traceback, file=f)
    sys.__excepthook__(exc_type, exc_value, exc_traceback)
sys.excepthook = exception_hook
```

## 关键学习点

### 1. Faulthandler 的作用

`faulthandler` 模块专门用于调试 Python 解释器本身的崩溃（如段错误），这些错误通常无法被 Python 的异常处理机制捕获。

### 2. 触发段错误的方法

```python
import ctypes
ctypes.string_at(0)  # 访问空指针地址，触发段错误
```

通过 `ctypes` 访问内存地址 0（空指针），强制触发操作系统级别的段错误。

### 3. 异常钩子机制

`sys.excepthook` 允许自定义未捕获异常的处理方式，非常适合用于全局错误日志记录。

### 4. 日志输出示例

**crash.log** (底层崩溃):
```
Fatal Python error: Segmentation fault

Current thread 0x00007f8c62769000 (most recent call first):
  File "/usr/lib/python3.10/ctypes/__init__.py", line 517 in string_at
  File "test.py", line 19 in trigger_segfault
```

**error.log** (Python 异常):
```
Traceback (most recent call last):
  File "test.py", line 23, in trigger_div_zero
    1 / 0
ZeroDivisionError: division by zero
```

## 运行方法

```bash
cd /home/lijian/project/learn-python
python test.py
```

程序会在 5 秒后触发段错误，生成 `crash.log` 和 `error.log` 两个日志文件。

## 相关概念

- [[Python 异常处理]]
- [[Python 调试技术]]
- [[C 语言内存管理]]
- [[操作系统信号处理]]
- [[Python ctypes 模块]]
```

## 相关笔记

- [[wenan]] — 嵌入式学习笔记合集
- [[xmind-notes]] — XMind 知识库概览
