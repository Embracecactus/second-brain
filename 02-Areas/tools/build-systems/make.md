---
tags:
  - make
  - build-system
  - c-language
  - gcc
  - tutorial
category: tools/build-systems
created: 2026-06-09
status: seed
---

# Make 构建系统示例项目

## 项目概述

一个用于学习 GNU Make 构建系统的最小化 C 语言示例项目，展示了如何使用 Makefile 管理源文件编译、自动依赖追踪和构建产物组织。

## 技术栈

- **语言**: C
- **编译器**: GCC (可通过 `CC` 变量覆盖)
- **构建系统**: GNU Make
- **编译选项**: `-Wall` (全部警告)、`-g` (调试信息)、`-MMD` (自动依赖生成)

## 项目结构

```
make/
├── Makefile          # 构建规则定义
├── src/
│   └── main.c        # 源代码
└── build/            # 构建产物目录
    ├── main          # 最终可执行文件
    ├── main.o        # 目标文件
    ├── main.d        # 依赖文件
    └── main.map      # 符号映射文件
```

## 架构与关键设计决策

### 1. 源码与构建产物分离

源文件放在 `src/` 目录，所有中间文件和最终产物输出到 `build/` 目录，保持源码目录整洁。

```makefile
SOURCES_DIR := src
BUILD_DIR := build
```

### 2. 自动源文件发现

使用 `wildcard` 函数自动扫描所有 `.c` 文件，无需手动维护源文件列表：

```makefile
SOURCES := $(wildcard $(SOURCES_DIR)/*.c)
```

### 3. 模式替换生成目标文件路径

使用 `patsubst` 将源文件路径映射为对应的 `.o` 文件路径：

```makefile
OBJECTS := $(patsubst $(SOURCES_DIR)/%.c, $(BUILD_DIR)/%.o, $(SOURCES))
# src/main.c → build/main.o
```

### 4. 自动依赖追踪

利用 GCC 的 `-MMD` 选项和单独的依赖生成规则，实现头文件变更时自动触发重新编译：

```makefile
# 依赖文件存放在 build/.deps/ 子目录
BUILD_DEP_DIR := $(BUILD_DIR)/.deps

# 依赖生成规则
$(BUILD_DEP_DIR)/%.d: $(SOURCES_DIR)/%.c
	@mkdir -p $(dir $@)
	@$(CC) $(CFLAGS) -M $< -MT "$(BUILD_DIR)/$(notdir $(<:.c=.o))" -MT "$@" -MF $@

# 在 Makefile 末尾包含依赖文件
-include $(wildcard $(BUILD_DEP_DIR)/*.d)
```

> [!tip] `-MMD` 与 `-M` 的区别
> `-MMD` 在正常编译时同时生成 `.d` 文件（排除系统头文件），而 `-M` 只生成依赖信息不做编译。本项目两者结合使用：编译规则用 `-MMD`，依赖生成规则用 `-M`。

### 5. Map 文件生成

链接时通过 `-Wl,-Map` 生成符号映射文件，便于分析内存布局和符号地址：

```makefile
$(CC) $(CFLAGS) -Wl,-Map=$(TARGET_MAP) -o $@ $^
```

## 核心 Makefile 模式

### 模式规则 (Pattern Rule)

```makefile
$(BUILD_DIR)/%.o: $(SOURCES_DIR)/%.c
	@mkdir -p $(BUILD_DIR)
	$(CC) $(CFLAGS) -c $< -o $@
```

- `%` 是通配符，匹配任意文件名
- `$<` 表示第一个依赖文件（即 `.c` 源文件）
- `$@` 表示目标文件（即 `.o` 文件）
- `@` 前缀抑制命令回显

### 自动变量速查

| 变量 | 含义 |
|------|------|
| `$@` | 目标文件名 |
| `$<` | 第一个依赖文件 |
| `$^` | 所有依赖文件（去重） |
| `$?` | 比目标新的依赖文件 |

## 构建与运行

```bash
# 编译项目
make

# 运行程序
./build/main

# 清理构建产物
make clean
```

## 关键学习点

1. **`?=` 赋值运算符** — 条件赋值，仅在变量未定义时生效，允许用户通过环境变量覆盖默认值（如 `CC=clang make`）
2. **`-include` 前缀 `-`** — 当被包含的文件不存在时不报错，首次构建时 `.d` 文件尚未生成，需要这个容错处理
3. **`.PHONY` 声明** — 将 `clean` 声明为伪目标，避免与同名文件冲突，确保 `make clean` 始终执行
4. **`@` 抑制回显** — 命令前加 `@` 使 Make 不打印该命令本身，只显示其输出，保持构建日志简洁
5. **自动依赖 vs 手动依赖** — 通过 `-MMD` 自动生成 `.d` 文件并 `-include` 进来，比手动编写头文件依赖更可靠、更易维护

## 相关概念

- [[GNU Make Manual]]
- [[GCC 编译选项]]
- [[C 语言构建工具链]]
- [[CMake]] — 更现代的跨平台构建系统替代方案
- [[Ninja]] — 注重速度的底层构建工具
- [[Bear]] — 生成 `compile_commands.json` 的工具，配合 IDE 使用

## 相关笔记

- [[cmake-apt-setup]] — CMake APT 安装配置
- [[stm32]] — STM32G0B1 Makefile 工程模板
- [[wch]] — WCH CH32V RISC-V MCU 项目（Makefile + Kconfig）
- [[h3-uboot-pack]] — H3 U-Boot 打包工具（Makefile 封装）
