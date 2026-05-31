# SPIRV-Tools Code Wiki

> 版本: v2026.2 | 许可证: Apache 2.0 | 维护方: Khronos Group Inc.

---

## 目录

1. [项目概述](#1-项目概述)
2. [项目架构](#2-项目架构)
3. [构建系统与依赖](#3-构建系统与依赖)
4. [核心模块 (source/)](#4-核心模块-source)
   - 4.1 [SPIR-V 核心 (source/ 根目录)](#41-spir-v-核心-source-根目录)
   - 4.2 [优化器模块 (source/opt/)](#42-优化器模块-sourceopt)
   - 4.3 [验证器模块 (source/val/)](#43-验证器模块-sourceval)
   - 4.4 [链接器模块 (source/link/)](#44-链接器模块-sourcelink)
   - 4.5 [模糊测试模块 (source/fuzz/)](#45-模糊测试模块-sourcefuzz)
   - 4.6 [缩减器模块 (source/reduce/)](#46-缩减器模块-sourcereduce)
   - 4.7 [差异比较模块 (source/diff/)](#47-差异比较模块-sourcediff)
   - 4.8 [工具库 (source/util/)](#48-工具库-sourceutil)
5. [公共 API 接口](#5-公共-api-接口)
6. [命令行工具](#6-命令行工具)
7. [模块间依赖关系](#7-模块间依赖关系)
8. [预编译工具下载](#8-预编译工具下载)
9. [输入文件获取方式](#9-输入文件获取方式)
10. [详细测试使用说明](#10-详细测试使用说明)
11. [macOS 编译指南](#11-macos-编译指南)

---

## 1. 项目概述

**SPIRV-Tools** 是由 Khronos Group 维护的开源项目，提供用于处理 SPIR-V 模块的 API 和命令行工具。SPIR-V 是 Vulkan、OpenCL 等 Khronos API 使用的中间着色器语言二进制格式。

### 核心功能

| 功能 | 描述 | 状态 |
|------|------|------|
| **汇编器 (Assembler)** | 将 SPIR-V 汇编文本转换为二进制 | 稳定 |
| **反汇编器 (Disassembler)** | 将 SPIR-V 二进制转换为汇编文本 | 稳定 |
| **二进制解析器** | 解析 SPIR-V 二进制模块并发出回调 | 稳定 |
| **验证器 (Validator)** | 检查 SPIR-V 模块是否符合规范 | 开发中 |
| **优化器 (Optimizer)** | 对 SPIR-V 模块应用代码变换/优化 | 稳定 |
| **链接器 (Linker)** | 合并多个 SPIR-V 二进制模块 | 开发中 |
| **缩减器 (Reducer)** | 简化/缩小 SPIR-V 模块 | 开发中 |
| **模糊测试器 (Fuzzer)** | 应用语义保持的变换进行变异测试 | 开发中 |
| **差异比较 (Diff)** | 比较两个 SPIR-V 模块的差异 | 开发中 |
| **代码检查 (Linter)** | 对 SPIR-V 模块执行代码检查 | 开发中 |

### 支持的 SPIR-V 版本

- SPIR-V 1.0 至 1.6
- 目标环境: Vulkan 1.0~1.4, OpenCL 1.2~2.2, OpenGL 4.0~4.5

---

## 2. 项目架构

### 目录结构

```
spirv-tools/
├── include/spirv-tools/     # 公共 API 头文件
│   ├── libspirv.h           # C 语言 API
│   ├── libspirv.hpp         # C++ 语言 API
│   ├── optimizer.hpp        # 优化器 C++ API
│   ├── linker.hpp           # 链接器 C++ API
│   └── linter.hpp           # 代码检查 C++ API
├── source/                  # 核心实现源码
│   ├── *.cpp/h              # 核心 SPIR-V 处理逻辑
│   ├── opt/                 # 优化器模块 (~70+ 优化 pass)
│   ├── val/                 # 验证器模块
│   ├── link/                # 链接器模块
│   ├── fuzz/                # 模糊测试模块
│   ├── reduce/              # 缩减器模块
│   ├── diff/                # 差异比较模块
│   └── util/                # 通用工具库
├── tools/                   # 命令行工具入口
├── test/                    # 测试代码
├── examples/                # 示例代码
├── external/                # 外部依赖
├── cmake/                   # CMake 配置文件
├── docs/                    # 文档
├── utils/                   # 构建和开发脚本
├── build_overrides/         # GN 构建覆盖配置
└── android_test/            # Android 测试配置
```

### 架构层次图

```
┌─────────────────────────────────────────────────────┐
│                  命令行工具层 (tools/)                │
│  spirv-as | spirv-dis | spirv-val | spirv-opt | ... │
├─────────────────────────────────────────────────────┤
│                   公共 API 层                        │
│  libspirv.h(C) | libspirv.hpp(C++) | optimizer.hpp  │
│  linker.hpp | linter.hpp                            │
├─────────────────────────────────────────────────────┤
│                   核心库层                           │
│  ┌─────────┐ ┌──────────┐ ┌──────┐ ┌──────┐       │
│  │ 汇编/反汇编│ │ 二进制解析 │ │ 验证器│ │ 诊断  │       │
│  └─────────┘ └──────────┘ └──────┘ └──────┘       │
├─────────────────────────────────────────────────────┤
│                  优化器库层 (opt/)                    │
│  IRContext | Pass | PassManager | Module | CFG      │
│  70+ 优化 Pass (DCE, 内联, 常量折叠, 循环优化...)    │
├─────────────────────────────────────────────────────┤
│               扩展功能层                             │
│  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐    │
│  │ 链接器 │ │ 模糊器 │ │ 缩减器 │ │ 差异  │ │ 检查  │    │
│  └──────┘ └──────┘ └──────┘ └──────┘ └──────┘    │
├─────────────────────────────────────────────────────┤
│                  工具库层 (util/)                     │
│  BitVector | SmallVector | IlList | Timer | ...     │
└─────────────────────────────────────────────────────┘
```

---

## 3. 构建系统与依赖

### 构建系统

| 构建系统 | 支持程度 | 说明 |
|----------|----------|------|
| **CMake** | 主要 | 推荐方式，最小版本 3.22.1 |
| **Bazel** | 支持 | 版本 7.4.0 |
| **GN** | 支持 | Chromium 构建系统 |
| **Android NDK** | 支持 | ndk-build 方式 |
| **Emscripten** | 支持 | 编译为 WebAssembly |

### 编译要求

- **C++ 标准**: C++17 (必须)
- **编译器**: GCC 15 / Clang 18 / AppleClang 15.0 / VS 2022
- **Python 3**: 用于脚本和测试

### 外部依赖

| 依赖 | 用途 | 必需 |
|------|------|------|
| **SPIRV-Headers** | SPIR-V 语法定义和 JSON 语法文件 | 是 |
| **googletest** | 测试框架 | 否(仅测试) |
| **Effcee** | 状态匹配测试库 | 否(仅测试) |
| **RE2** | 正则表达式库 (Effcee 依赖) | 否(仅测试) |
| **Abseil** | C++ 基础库 (RE2 依赖) | 否(仅测试) |
| **mimalloc** | 高性能内存分配器 | 否(可选) |
| **protobuf** | Fuzzer 序列化 (仅 fuzzer) | 否(仅 fuzzer) |

### CMake 选项

| 选项 | 默认值 | 说明 |
|------|--------|------|
| `SPIRV_BUILD_FUZZER` | OFF | 构建 spirv-fuzz 工具 |
| `SPIRV_COLOR_TERMINAL` | ON | 启用彩色控制台输出 |
| `SPIRV_SKIP_TESTS` | OFF | 跳过测试构建 |
| `SPIRV_SKIP_EXECUTABLES` | OFF | 仅构建库 |
| `SPIRV_WERROR` | ON | 警告视为错误 |
| `SPIRV_WARN_EVERYTHING` | OFF | 启用所有警告 |
| `SPIRV_USE_SANITIZER` | - | 启用 Clang sanitizer |
| `SPIRV_TOOLS_USE_MIMALLOC` | 平台相关 | 使用 mimalloc |
| `ENABLE_RTTI` | OFF | 启用 RTTI |

### 生成库文件

| 目标 | 输出 | 说明 |
|------|------|------|
| `SPIRV-Tools` | `libSPIRV-Tools.a` / `.lib` | 核心静态库 |
| `SPIRV-Tools-shared` | `libSPIRV-Tools-shared.so` / `.dll` | 核心动态库 |
| `SPIRV-Tools-opt` | `libSPIRV-Tools-opt.a` / `.lib` | 优化器静态库 |
| `SPIRV-Tools-link` | `libSPIRV-Tools-link.a` / `.lib` | 链接器静态库 |

---

## 4. 核心模块 (source/)

### 4.1 SPIR-V 核心 (source/ 根目录)

核心模块实现了 SPIR-V 二进制的汇编、反汇编、解析等基础功能，是所有其他模块的基础。

#### 关键文件

| 文件 | 职责 |
|------|------|
| `libspirv.cpp` | C API 的主要实现，封装汇编/反汇编/验证功能 |
| `binary.cpp/h` | SPIR-V 二进制编解码 |
| `text.cpp/h` | SPIR-V 汇编文本处理 |
| `text_handler.cpp/h` | 汇编文本词法分析 |
| `assembly_grammar.cpp/h` | SPIR-V 汇编语法规则 |
| `opcode.cpp/h` | SPIR-V 操作码定义和查询 |
| `operand.cpp/h` | 操作数类型定义和解析 |
| `parsed_operand.cpp/h` | 已解析操作数结构 |
| `disassemble.cpp/h` | 反汇编器实现 |
| `diagnostic.cpp/h` | 诊断信息（错误/警告）处理 |
| `ext_inst.cpp/h` | 扩展指令集处理 |
| `extensions.cpp/h` | SPIR-V 扩展支持 |
| `spirv_endian.cpp/h` | 字节序处理 |
| `spirv_target_env.cpp/h` | 目标环境（Vulkan/OpenCL 等）支持 |
| `table.cpp/h` | SPIR-V 指令语法查找表 |
| `name_mapper.cpp/h` | ID 到名称的映射 |
| `cfa.h` | 控制流分析算法 |
| `enum_set.h` | 枚举集合模板 |

#### 关键函数 (C API)

```c
spv_result_t spvTextToBinary(spv_const_context, const char* text,
                              size_t length, spv_binary* binary,
                              spv_diagnostic* diagnostic);

spv_result_t spvBinaryToText(spv_const_context, const uint32_t* binary,
                              size_t word_count, uint32_t options,
                              spv_text* text, spv_diagnostic* diagnostic);

spv_result_t spvBinaryParse(spv_const_context, void* user_data,
                             const uint32_t* words, size_t num_words,
                             spv_parsed_header_fn_t,
                             spv_parsed_instruction_fn_t,
                             spv_diagnostic* diagnostic);

spv_result_t spvValidate(spv_const_context, spv_const_binary,
                          spv_diagnostic* diagnostic);
```

---

### 4.2 优化器模块 (source/opt/)

优化器是 SPIRV-Tools 中最大的模块，包含 70+ 个优化 Pass。

#### 核心类

| 类 | 描述 |
|----|------|
| `IRContext` | IR 上下文，持有模块和分析结果 |
| `Module` | SPIR-V 模块结构 |
| `Function` | SPIR-V 函数 |
| `BasicBlock` | 基本块 |
| `Instruction` | 指令 |
| `Pass` | 优化 Pass 基类 |
| `PassManager` | Pass 管理器 |
| `CFG` | 控制流图 |
| `DefUseManager` | 定义-使用管理器 |
| `DecorationManager` | 装饰管理器 |
| `TypeManager` | 类型管理器 |
| `FeatureManager` | 特性管理器 |

#### 优化 Pass 分类

**简化类**: StripDebugInfoPass, StripNonSemanticInfoPass, FlattenDecorationPass, CompactIdsPass, RemoveDuplicatesPass, CFGCleanupPass

**代码缩减类**: InlineExhaustivePass, LocalAccessChainConvertPass, LocalSingleBlockElimPass, LocalSingleStoreElimPass, LocalMultiStoreElimPass, AggressiveDCEPass, DeadBranchElimPass, BlockMergePass, EliminateDeadFunctionsPass, MergeReturnPass, SSARewritePass

**代码改进类**: CCPass, IfConversionPass, LICMPass, LoopFissionPass, LoopFusionPass, LoopUnrollPass, SimplificationPass, StrengthReductionPass, ScalarReplacementPass, RedundancyEliminationPass, CopyPropagateArraysPass

**预设配方**: `-O` (性能), `-Os` (大小), `--legalize-hlsl` (合法化)

---

### 4.3 验证器模块 (source/val/)

验证器检查 SPIR-V 模块是否符合规范。包含 30+ 个验证文件，覆盖：控制流、类型系统、内存操作、图像/采样、算术/位运算、函数调用、装饰/注解、光线追踪、扩展指令、内建变量、接口/布局、网格着色等。

### 4.4 链接器模块 (source/link/)

链接器将多个 SPIR-V 二进制模块合并。支持选项：CreateLibrary, VerifyIds, AllowPartialLinkage, UseHighestVersion 等。

### 4.5 模糊测试模块 (source/fuzz/)

模糊测试器对 SPIR-V 模块应用语义保持的变换。核心类：Fuzzer, FuzzerPass, Transformation。

### 4.6 缩减器模块 (source/reduce/)

缩减器简化 SPIR-V 模块。14 种内置缩减策略。

### 4.7 差异比较模块 (source/diff/)

差异比较工具。使用最长公共子序列 (LCS) 算法匹配指令。

### 4.8 工具库 (source/util/)

通用工具：BitVector, SmallVector, IlList, Timer, Span, HexFloat, StringUtils 等。

---

## 5. 公共 API 接口

### C API (libspirv.h)

| 函数 | 描述 |
|------|------|
| `spvContextCreate(env)` | 创建上下文 |
| `spvTextToBinary(...)` | 汇编 |
| `spvBinaryToText(...)` | 反汇编 |
| `spvBinaryParse(...)` | 二进制解析 |
| `spvValidate(...)` | 验证 |
| `spvOptimizerCreate(env)` | 创建优化器 |
| `spvOptimizerRun(...)` | 运行优化 |

### C++ API

| 类 | 头文件 | 描述 |
|----|--------|------|
| `SpirvTools` | libspirv.hpp | 汇编/反汇编/验证 |
| `Optimizer` | optimizer.hpp | 优化器 |
| `LinkerOptions` | linker.hpp | 链接器选项 |
| `Linter` | linter.hpp | 代码检查 |
| `Context` | libspirv.hpp | RAII 上下文 |

---

## 6. 命令行工具

| 工具 | 路径 | 描述 |
|------|------|------|
| `spirv-as` | tools/as/ | 汇编器: 文本 → 二进制 |
| `spirv-dis` | tools/dis/ | 反汇编器: 二进制 → 文本 |
| `spirv-val` | tools/val/ | 验证器 |
| `spirv-opt` | tools/opt/ | 优化器 |
| `spirv-link` | tools/link/ | 链接器 |
| `spirv-cfg` | tools/cfg/ | 控制流图导出 (GraphViz) |
| `spirv-fuzz` | tools/fuzz/ | 模糊测试器 |
| `spirv-reduce` | tools/reduce/ | 缩减器 |
| `spirv-diff` | tools/diff/ | 差异比较 |
| `spirv-lint` | tools/lint/ | 代码检查 |
| `spirv-objdump` | tools/objdump/ | 对象转储 |

---

## 7. 模块间依赖关系

```
命令行工具 → libSPIRV-Tools-opt → libSPIRV-Tools → SPIRV-Headers
           ↘ libSPIRV-Tools-link ↗
Fuzzer/Reducer/Diff → libSPIRV-Tools-opt → libSPIRV-Tools
```

---

## 8. 预编译工具下载

本项目的 dist 目录中提供了已编译好的工具文件，可直接使用：

### dist 目录结构

```
dist/
├── linux-x86_64/              # Linux x86_64 预编译工具
│   ├── spirv-as               # 汇编器
│   ├── spirv-dis              # 反汇编器
│   ├── spirv-val              # 验证器
│   ├── spirv-opt              # 优化器
│   ├── spirv-link             # 链接器
│   ├── spirv-cfg              # 控制流图
│   ├── spirv-diff             # 差异比较
│   ├── spirv-lint             # 代码检查
│   ├── spirv-objdump          # 对象转储
│   └── spirv-reduce           # 缩减器
├── windows-x86_64/            # Windows x86_64 预编译工具 (MinGW 交叉编译)
│   ├── spirv-as.exe
│   ├── spirv-dis.exe
│   ├── spirv-val.exe
│   ├── spirv-opt.exe
│   ├── spirv-link.exe
│   ├── spirv-cfg.exe
│   ├── spirv-diff.exe
│   ├── spirv-lint.exe
│   ├── spirv-objdump.exe
│   └── spirv-reduce.exe
├── samples/                   # 测试用样本文件
│   ├── simple_vertex.spvasm   # 顶点着色器汇编源码
│   ├── simple_fragment.spvasm # 片段着色器汇编源码
│   ├── simple_vertex.spv      # 已编译顶点着色器二进制
│   ├── simple_fragment.spv    # 已编译片段着色器二进制
│   ├── simple_vertex_opt.spv  # 优化后的顶点着色器
│   ├── simple_fragment_opt.spv# 优化后的片段着色器
│   ├── simple_vertex_stripped.spv # 去除调试信息后的着色器
│   └── linked.spv             # 链接后的着色器
├── SPIRV-Tools-Code-Wiki.md   # 本文档 (Markdown)
└── SPIRV-Tools-Code-Wiki.html # 本文档 (HTML)
```

### 编译信息

| 平台 | 编译器 | 编译模式 | 大小 |
|------|--------|----------|------|
| Linux x86_64 | GCC 13.3.0 | Release | ~28MB (全部工具) |
| Windows x86_64 | MinGW-w64 GCC 13.2.0 (交叉编译) | Release | ~55MB (全部工具) |
| macOS | 需在 macOS 上原生编译 | Release | 见下方编译指南 |

### Linux 使用方法

```bash
# 添加工具到 PATH
export PATH=/path/to/dist/linux-x86_64:$PATH

# 或者直接指定完整路径
/path/to/dist/linux-x86_64/spirv-val your_shader.spv
```

### Windows 使用方法

1. 将 `dist\windows-x86_64` 目录复制到 Windows 机器
2. 打开 CMD 或 PowerShell
3. 直接运行 `.exe` 文件：
```cmd
C:\path\to\dist\windows-x86_64\spirv-val.exe your_shader.spv
```
4. 或将目录加入 PATH 环境变量后直接使用工具名

> **好消息**: Windows 版本通过 MinGW 交叉编译生成，已将运行时库静态链接到可执行文件中。工具仅依赖 Windows 系统自带的 `KERNEL32.dll` 和 `msvcrt.dll`，**无需额外 DLL，可直接运行**。

---

## 9. 输入文件获取方式

SPIRV-Tools 处理的输入文件主要有两类：**SPIR-V 汇编文本** (`.spvasm`) 和 **SPIR-V 二进制** (`.spv`)。以下是获取方式：

### 方式一：使用本项目自带的样本文件

`dist/samples/` 目录已包含可直接使用的测试文件：

| 文件 | 类型 | 描述 |
|------|------|------|
| `simple_vertex.spvasm` | 汇编文本 | 简单顶点着色器 (输出 vec4(1,1,1,1)) |
| `simple_fragment.spvasm` | 汇编文本 | 简单片段着色器 (输出红色) |
| `simple_vertex.spv` | 二进制 | 已编译的顶点着色器 |
| `simple_fragment.spv` | 二进制 | 已编译的片段着色器 |

### 方式二：从 GLSL/HLSL 编译生成 (最常用)

这是实际开发中最常见的方式，使用着色器编译器将高级着色语言编译为 SPIR-V：

#### 使用 glslangValidator (Vulkan SDK 自带)

```bash
# 安装: Vulkan SDK 自带，或 apt install glslang-tools
# GLSL 顶点着色器 → SPIR-V
glslangValidator -V shader.vert -o shader.spv

# GLSL 片段着色器 → SPIR-V
glslangValidator -V shader.frag -o shader.spv

# GLSL 计算着色器 → SPIR-V
glslangValidator -V shader.comp -o shader.spv
```

#### 使用 dxc (DirectX Shader Compiler)

```bash
# 安装: https://github.com/microsoft/DirectXShaderCompiler
# HLSL 顶点着色器 → SPIR-V
dxc -T vs_6_0 -E main shader.hlsl -spirv -o shader.spv

# HLSL 片段着色器 → SPIR-V
dxc -T ps_6_0 -E main shader.hlsl -spirv -o shader.spv

# HLSL 计算着色器 → SPIR-V
dxc -T cs_6_0 -E main shader.hlsl -spirv -o shader.spv
```

### 方式三：手工编写 SPIR-V 汇编

直接编写 `.spvasm` 文件，然后用 `spirv-as` 编译：

```spirv
; SPIR-V
; Version: 1.0
; Generator: Khronos SPIR-V Tools Assembler; 0
; Bound: 7
; Schema: 0
               OpCapability Shader
          %1 = OpExtInstImport "GLSL.std.450"
               OpMemoryModel Logical GLSL450
               OpEntryPoint Vertex %main "main"
               OpSource GLSL 450
               OpName %main "main"
       %void = OpTypeVoid
          %3 = OpTypeFunction %void
       %main = OpFunction %void None %3
          %5 = OpLabel
               OpReturn
               OpFunctionEnd
```

编译为二进制：
```bash
spirv-as my_shader.spvasm -o my_shader.spv
```

### 方式四：从现有项目中获取

| 来源 | 说明 |
|------|------|
| Vulkan SDK 示例 | `VULKAN_SDK/examples/` 目录下的着色器 |
| Khronos SPIRV-Headers | `test/` 目录下的测试文件 |
| GPU 驱动工具 | RenderDoc、PIX 等可捕获着色器 |
| 开源游戏引擎 | Unity/Unreal/Godot 编译输出 |
| SPIRV-Tools 仓库 | `test/` 目录包含大量测试用 `.spvasm` 文件 |
| GraphicsFuzz | `test/fuzzers/corpora/spv/` 包含模糊测试语料 |

### 方式五：从 Vulkan 应用中提取

使用 RenderDoc 等图形调试工具捕获 Vulkan 应用的帧数据，可直接导出 SPIR-V 着色器二进制。

---

## 10. 详细测试使用说明

以下使用 `dist/samples/` 中的样本文件演示所有工具的完整使用流程。

### 10.1 汇编器 spirv-as — 将汇编文本编译为二进制

```bash
# 基本用法
spirv-as simple_vertex.spvasm -o simple_vertex.spv

# 保留数字 ID (不重新编号)
spirv-as --preserve-numeric-ids simple_vertex.spvasm -o simple_vertex.spv

# 指定目标环境
spirv-as --target-env spv1.3 simple_vertex.spvasm -o simple_vertex.spv

# 验证输出
spirv-dis simple_vertex.spv
```

**预期输出**: 无错误信息，生成 `.spv` 二进制文件。

### 10.2 反汇编器 spirv-dis — 将二进制转换为汇编文本

```bash
# 基本用法
spirv-dis simple_vertex.spv

# 输出到文件
spirv-dis simple_vertex.spv -o output.spvasm

# 不显示头部注释
spirv-dis --no-header simple_vertex.spv

# 使用友好名称
spirv-dis --friendly-names simple_vertex.spv

# 显示字节偏移
spirv-dis --offsets simple_vertex.spv

# 嵌套缩进 (更可读)
spirv-dis --nested-indent simple_vertex.spv

# 处理未知操作码
spirv-dis --handle-unknown-opcodes simple_vertex.spv
```

**预期输出**:
```
; SPIR-V
; Version: 1.0
; Generator: Khronos SPIR-V Tools Assembler; 0
; Bound: 15
; Schema: 0
               OpCapability Shader
          %1 = OpExtInstImport "GLSL.std.450"
               OpMemoryModel Logical GLSL450
               OpEntryPoint Vertex %main "main" %gl_Position
               ...
```

### 10.3 验证器 spirv-val — 验证 SPIR-V 模块合法性

```bash
# 基本验证
spirv-val simple_vertex.spv

# 指定目标环境
spirv-val --target-env vulkan1.2 simple_vertex.spv

# 放松逻辑指针规则
spirv-val --relax-logical-pointer simple_vertex.spv

# 放松存储结构规则
spirv-val --relax-store-structure simple_vertex.spv

# 使用标量块布局
spirv-val --scalar-block-layout simple_vertex.spv

# HLSL 合法化前的宽松验证
spirv-val --before-hlsl-legalization simple_vertex.spv

# 显示友好名称
spirv-val --friendly-names simple_vertex.spv
```

**预期输出** (合法模块): 无输出，退出码 0。

**非法模块输出示例**:
```
error: 13: OpStore Value <id> '14'ss type does not match Object <id> '13'ss type.
```

### 10.4 优化器 spirv-opt — 优化 SPIR-V 模块

```bash
# 性能优化 (-O)
spirv-opt -O simple_vertex.spv -o optimized_perf.spv

# 大小优化 (-Os)
spirv-opt -Os simple_fragment.spv -o optimized_size.spv

# HLSL 合法化
spirv-opt --legalize-hlsl shader.spv -o legalized.spv

# 指定单个优化 Pass
spirv-opt --strip-debug simple_vertex.spv -o stripped.spv
spirv-opt --inline-entry-points-exhaustive simple_vertex.spv -o inlined.spv
spirv-opt --eliminate-dead-code-aggressive simple_vertex.spv -o dce.spv
spirv-opt --constant-propagation simple_vertex.spv -o ccp.spv
spirv-opt --loop-unroll simple_vertex.spv -o unrolled.spv

# 组合多个 Pass
spirv-opt --strip-debug --eliminate-dead-code-aggressive \
          --merge-blocks --compact-ids \
          simple_vertex.spv -o fully_optimized.spv

# 查看所有可用 Pass
spirv-opt --help

# 打印每步优化结果
spirv-opt -O --print-all simple_vertex.spv -o optimized.spv

# 每步优化后验证
spirv-opt -O --validate-after-all simple_vertex.spv -o optimized.spv

# 保留绑定
spirv-opt -O --preserve-bindings simple_vertex.spv -o optimized.spv

# 保留专业化常量
spirv-opt -O --preserve-spec-constants simple_vertex.spv -o optimized.spv
```

**预期输出**: 无错误信息，生成优化后的 `.spv` 文件（通常更小）。

### 10.5 链接器 spirv-link — 合并多个 SPIR-V 模块

```bash
# 基本链接
spirv-link vertex.spv fragment.spv -o linked.spv

# 创建库 (保留导出符号)
spirv-link --create-library a.spv b.spv -o library.spv

# 允许部分链接
spirv-link --allow-partial-linkage a.spv b.spv -o partial.spv

# 验证 ID 唯一性
spirv-link --verify-ids a.spv b.spv -o linked.spv
```

### 10.6 控制流图 spirv-cfg — 导出 GraphViz 格式

```bash
# 导出 DOT 文件
spirv-cfg simple_vertex.spv -o cfg.dot

# 生成 PNG 图片 (需要 graphviz)
spirv-cfg simple_vertex.spv -o cfg.dot && dot -Tpng cfg.dot -o cfg.png

# 生成 SVG 图片
spirv-cfg simple_vertex.spv -o cfg.dot && dot -Tsvg cfg.dot -o cfg.svg
```

### 10.7 差异比较 spirv-diff — 比较两个模块

```bash
# 比较优化前后
spirv-diff original.spv optimized.spv

# 比较两个不同版本
spirv-diff v1.spv v2.spv
```

**预期输出** (带 `-` 和 `+` 标记):
```
-               OpName %4 "v"
+               OpName %3 "v"
```

### 10.8 代码检查 spirv-lint

```bash
spirv-lint simple_vertex.spv
```

### 10.9 对象转储 spirv-objdump

```bash
# 显示所有节
spirv-objdump -s simple_vertex.spv

# 显示源码
spirv-objdump --source simple_vertex.spv
```

### 10.10 缩减器 spirv-reduce

```bash
# 使用有趣性测试缩减
spirv-reduce buggy.spv --interestingness="spirv-val %spv" -o reduced.spv

# 指定步数限制
spirv-reduce buggy.spv --interestingness="your_test.sh %spv" \
          --step-limit=100 -o reduced.spv
```

### 完整测试流程示例

```bash
#!/bin/bash
# 完整的 SPIRV-Tools 测试流程

# 1. 汇编
spirv-as simple_vertex.spvasm -o vertex.spv
echo "汇编完成: vertex.spv"

# 2. 验证
spirv-val vertex.spv
echo "验证结果: $? (0=通过)"

# 3. 反汇编
spirv-dis vertex.spv -o vertex_dis.spvasm
echo "反汇编完成: vertex_dis.spvasm"

# 4. 性能优化
spirv-opt -O vertex.spv -o vertex_opt.spv
echo "优化完成: vertex_opt.spv"

# 5. 验证优化结果
spirv-val vertex_opt.spv
echo "优化后验证: $? (0=通过)"

# 6. 比较大小
echo "原始大小: $(wc -c < vertex.spv) bytes"
echo "优化大小: $(wc -c < vertex_opt.spv) bytes"

# 7. 差异比较
spirv-diff vertex.spv vertex_opt.spv

# 8. 去除调试信息
spirv-opt --strip-debug vertex.spv -o vertex_stripped.spv
echo "去调试信息大小: $(wc -c < vertex_stripped.spv) bytes"

# 9. 导出控制流图
spirv-cfg vertex.spv -o vertex_cfg.dot
echo "控制流图已导出"
```

---

## 11. macOS 编译指南

macOS 版本需要在 macOS 机器上原生编译（无法从 Linux 交叉编译）。步骤如下：

### 前提条件

- macOS 12.0 或更高版本
- Xcode Command Line Tools: `xcode-select --install`
- CMake: `brew install cmake`
- Python 3: `brew install python3`

### 编译步骤

```bash
# 1. 克隆源码
git clone https://github.com/KhronosGroup/SPIRV-Tools.git
cd SPIRV-Tools
python3 utils/git-sync-deps

# 2. 编译
mkdir build && cd build
cmake -DCMAKE_BUILD_TYPE=Release -DSPIRV_SKIP_TESTS=ON ..
cmake --build . -j$(sysctl -n hw.ncpu)

# 3. 工具位于
ls tools/spirv-*

# 4. 可选：安装到系统
sudo cmake --install .
```

### 生成 Universal Binary (Apple Silicon + Intel)

```bash
mkdir build-universal && cd build-universal
cmake -DCMAKE_BUILD_TYPE=Release \
      -DCMAKE_OSX_ARCHITECTURES="x86_64;arm64" \
      -DSPIRV_SKIP_TESTS=ON ..
cmake --build . -j$(sysctl -n hw.ncpu)
```

### 使用 Homebrew 安装 (最简单)

```bash
brew install spirv-tools
```

安装后工具位于 `/usr/local/bin/` 或 `/opt/homebrew/bin/`。

---

> 本文档基于 SPIRV-Tools v2026.2 源码分析生成，包含实际编译和测试结果。最后更新: 2026-05-30
