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
8. [项目运行方式](#8-项目运行方式)
9. [测试体系](#9-测试体系)

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
| `table2.cpp/h` | 第二代语法查找表 |
| `name_mapper.cpp/h` | ID 到名称的映射 |
| `print.cpp/h` | SPIR-V 打印/格式化 |
| `to_string.cpp/h` | 枚举值到字符串转换 |
| `cfa.h` | 控制流分析算法 |
| `enum_set.h` | 枚举集合模板 |
| `instruction.h` | 核心指令结构定义 |
| `macro.h` | 预处理器宏 |

#### 关键函数 (C API)

```c
// 汇编: 文本 -> 二进制
spv_result_t spvTextToBinary(spv_const_context, const char* text,
                              size_t length, spv_binary* binary,
                              spv_diagnostic* diagnostic);

// 反汇编: 二进制 -> 文本
spv_result_t spvBinaryToText(spv_const_context, const uint32_t* binary,
                              size_t word_count, uint32_t options,
                              spv_text* text, spv_diagnostic* diagnostic);

// 二进制解析 (回调式)
spv_result_t spvBinaryParse(spv_const_context, void* user_data,
                             const uint32_t* words, size_t num_words,
                             spv_parsed_header_fn_t,
                             spv_parsed_instruction_fn_t,
                             spv_diagnostic* diagnostic);

// 验证
spv_result_t spvValidate(spv_const_context, spv_const_binary,
                          spv_diagnostic* diagnostic);
spv_result_t spvValidateWithOptions(spv_const_context,
                                     spv_const_validator_options,
                                     spv_const_binary,
                                     spv_diagnostic* diagnostic);
```

#### 关键数据结构

```c
// SPIR-V 解析后的指令
typedef struct spv_parsed_instruction_t {
    const uint32_t* words;         // 指令字数组
    uint16_t num_words;            // 字数
    uint16_t opcode;               // 操作码
    spv_ext_inst_type_t ext_inst_type;  // 扩展指令类型
    uint32_t type_id;              // 类型 ID
    uint32_t result_id;            // 结果 ID
    const spv_parsed_operand_t* operands;  // 操作数数组
    uint16_t num_operands;         // 操作数数量
} spv_parsed_instruction_t;

// 结果状态码
typedef enum spv_result_t {
    SPV_SUCCESS = 0,
    SPV_UNSUPPORTED = 1,
    SPV_ERROR_INVALID_BINARY = -4,
    SPV_ERROR_INVALID_TEXT = -5,
    SPV_ERROR_INVALID_CFG = -11,
    // ...
} spv_result_t;
```

---

### 4.2 优化器模块 (source/opt/)

优化器是 SPIRV-Tools 中最大的模块，包含 70+ 个优化 Pass，用于对 SPIR-V 模块进行各种代码变换和优化。

#### 核心类层次

```
Pass (基类)
├── PassToken::Impl (包装器)
├── AggressiveDCEPass
├── BlockMergePass
├── CCPass
├── CFGCleanupPass
├── CodeSinkingPass
├── CombineAccessChainsPass
├── CompactIdsPass
├── ConvertToHalfPass
├── CopyPropagateArraysPass
├── DeadBranchElimPass
├── DeadInsertElimPass
├── DeadVariableEliminationPass
├── EliminateDeadConstantPass
├── EliminateDeadFunctionsPass
├── EliminateDeadMembersPass
├── FlattenDecorationPass
├── FoldSpecConstantOpAndCompositePass
├── FreezeSpecConstantValuePass
├── GraphicsRobustAccessPass
├── IfConversionPass
├── InlineExhaustivePass
├── InlineOpaquePass
├── LICMPass
├── LocalAccessChainConvertPass
├── LocalMultiStoreElimPass
├── LocalRedundancyEliminationPass
├── LocalSingleBlockElimPass
├── LocalSingleStoreElimPass
├── LoopFissionPass
├── LoopFusionPass
├── LoopPeelingPass
├── LoopUnrollerPass
├── LoopUnswitchPass
├── MergeReturnPass
├── PrivateToLocalPass
├── RedundancyEliminationPass
├── ScalarReplacementPass
├── SimplificationPass
├── StrengthReductionPass
├── StripDebugInfoPass
├── StripNonSemanticInfoPass
├── SSARewritePass
├── UnifyConstantPass
├── VectorDCEPass
├── ... (更多)
└── NullPass / EmptyPass
```

#### 关键核心类

##### `IRContext` — IR 上下文

IRContext 是优化器的核心容器，持有 SPIR-V 模块的所有信息和分析结果。

```cpp
class IRContext {
public:
    // 构造与模块管理
    IRContext(spv_target_env env, MessageConsumer consumer);
    Module* module();                    // 获取模块
    std::unique_ptr<Module> CloneModule(); // 克隆模块

    // 分析管理器访问
    DefUseManager* get_def_use_mgr();    // 定义-使用管理器
    DecorationManager* get_decoration_mgr(); // 装饰管理器
    TypeManager* get_type_mgr();         // 类型管理器
    FeatureManager* get_feature_mgr();   // 特性管理器
    CFG* cfg();                          // 控制流图
    DominatorAnalysis* GetDominatorAnalysis(const Function*); // 支配树
    LoopDescriptor* GetLoopDescriptor(const Function*);       // 循环描述

    // 构建与修改
    bool BuildIdToNameMap();             // 构建 ID->名称映射
    void BuildInvalidAnalyses(IRContext::Analysis); // 重建分析
    void InvalidateAnalysesExceptFor(IRContext::Analysis); // 使分析失效

    // 指令操作
    uint32_t TakeNextId();               // 获取下一个可用 ID
    bool IsConsistent();                 // 检查上下文一致性
};
```

##### `Module` — SPIR-V 模块

```cpp
class Module {
public:
    // 遍历指令
    InstructionList& types_values();     // 类型/常量/全局变量
    std::list<Function>& functions();    // 函数列表

    // 模块级指令
    Instruction* GetMemoryModel();       // 内存模型
    std::vector<Instruction*>& entry_points(); // 入口点

    // ID 管理
    uint32_t IdBound();                  // ID 上限
    void SetIdBound(uint32_t);           // 设置 ID 上限
};
```

##### `Function` — SPIR-V 函数

```cpp
class Function {
public:
    // 基本块访问
    std::list<BasicBlock>& blocks();     // 基本块列表
    BasicBlock* entry();                 // 入口基本块

    // 参数与返回值
    uint32_t result_id();                // 函数结果 ID
    uint32_t GetReturnTypeId();          // 返回类型 ID

    // 指令遍历
    iterator begin();                    // 遍历所有指令
    iterator end();
};
```

##### `BasicBlock` — 基本块

```cpp
class BasicBlock {
public:
    uint32_t id();                       // 基本块 ID (标签)
    InstructionList& instructions();     // 指令列表
    bool IsLoopHeader();                 // 是否为循环头
    BasicBlock* GetMergeInst();          // 获取合并指令
};
```

##### `Instruction` — 指令

```cpp
class Instruction {
public:
    SpvOp opcode();                      // 操作码
    uint32_t type_id();                  // 类型 ID
    uint32_t result_id();                // 结果 ID
    uint32_t GetSingleWordOperand(uint32_t index); // 获取操作数
    void SetOperand(uint32_t index, std::vector<uint32_t>&& operands); // 设置操作数

    // 遍历
    iterator begin();                    // 遍历操作数
    iterator end();

    // 修改
    void InsertBefore(std::unique_ptr<Instruction>&&); // 在此指令前插入
    void RemoveFromList();               // 从列表移除
};
```

##### `Pass` — 优化 Pass 基类

```cpp
class Pass {
public:
    virtual ~Pass() = default;

    // 执行优化，返回模块是否被修改
    virtual Status Process() = 0;

    // Pass 名称
    virtual const char* name() const = 0;

    // 上下文访问
    IRContext* context();                // 获取 IR 上下文
    void SetMessageConsumer(MessageConsumer); // 设置消息消费者
};
```

##### `PassManager` — Pass 管理器

```cpp
class PassManager {
public:
    void AddPass(std::unique_ptr<Pass> pass); // 添加 Pass
    uint32_t NumPasses();                       // Pass 数量
    Pass* GetPass(uint32_t index);              // 获取 Pass
    Status Run(IRContext* context);             // 运行所有 Pass
    void SetMessageConsumer(MessageConsumer);   // 设置消息消费者
};
```

##### `CFG` — 控制流图

```cpp
class CFG {
public:
    CFG(IRContext* context);
    void AddEdges(BasicBlock* block);    // 添加基本块边
    std::vector<BasicBlock*> preds(BasicBlock* block);  // 前驱
    std::vector<BasicBlock*> succs(BasicBlock* block);  // 后继
};
```

##### `DefUseManager` — 定义-使用管理器

```cpp
class DefUseManager {
public:
    Instruction* GetDef(uint32_t id);    // 获取 ID 的定义指令
    const std::vector<Instruction*>& GetUses(uint32_t id); // 获取 ID 的使用
    void AnalyzeInstDef(Instruction* inst);  // 分析指令定义
    void AnalyzeInstUse(Instruction* inst);  // 分析指令使用
};
```

##### `TypeManager` — 类型管理器

```cpp
class TypeManager {
public:
    Instruction* GetType(uint32_t id);   // 通过 ID 获取类型
    uint32_t GetId(const Type* type);    // 通过类型获取 ID
    bool IsSameType(uint32_t id1, uint32_t id2); // 类型比较
};
```

##### `DecorationManager` — 装饰管理器

```cpp
class DecorationManager {
public:
    std::vector<Instruction*> GetDecorationsFor(uint32_t id, bool include_groups = true);
    bool HasDecoration(uint32_t id, SpvDecoration decoration);
    void AddDecoration(uint32_t target, SpvDecoration decoration, ...);
};
```

##### `FeatureManager` — 特性管理器

```cpp
class FeatureManager {
public:
    bool HasCapability(SpvCapability cap);  // 检查能力
    bool HasExtension(Extension ext);       // 检查扩展
    bool IsCapabilityEnabled(SpvCapability cap);
};
```

#### 优化 Pass 完整列表

##### 简化类 Pass

| Pass | 创建函数 | 描述 |
|------|----------|------|
| StripDebugInfoPass | `CreateStripDebugInfoPass()` | 移除所有调试指令 |
| StripNonSemanticInfoPass | `CreateStripNonSemanticInfoPass()` | 移除非语义信息 |
| FlattenDecorationPass | `CreateFlattenDecorationPass()` | 将分组装饰替换为等价的非分组装饰 |
| CompactIdsPass | `CreateCompactIdsPass()` | 将 ID 重新映射为紧凑连续范围 |
| CanonicalizeIdsPass | `CreateCanonicalizeIdsPass()` | 规范化 ID 以改善压缩 |
| RemoveDuplicatesPass | `CreateRemoveDuplicatesPass()` | 移除重复的能力/导入/类型/装饰 |
| CFGCleanupPass | `CreateCFGCleanupPass()` | 清理控制流图中的冗余 |

##### 专业化常量类 Pass

| Pass | 创建函数 | 描述 |
|------|----------|------|
| SetSpecConstantDefaultValuePass | `CreateSetSpecConstantDefaultValuePass()` | 设置专业化常量默认值 |
| FreezeSpecConstantValuePass | `CreateFreezeSpecConstantValuePass()` | 冻结专业化常量为其默认值 |
| FoldSpecConstantOpAndCompositePass | `CreateFoldSpecConstantOpAndCompositePass()` | 折叠 OpSpecConstantOp/Composite |
| UnifyConstantPass | `CreateUnifyConstantPass()` | 去重常量 |
| EliminateDeadConstantPass | `CreateEliminateDeadConstantPass()` | 移除死常量 |

##### 代码缩减类 Pass

| Pass | 创建函数 | 描述 |
|------|----------|------|
| InlineExhaustivePass | `CreateInlineExhaustivePass()` | 穷举内联所有函数调用 |
| InlineOpaquePass | `CreateInlineOpaquePass()` | 内联含不透明类型的函数 |
| LocalAccessChainConvertPass | `CreateLocalAccessChainConvertPass()` | 将局部访问链转换为插入/提取 |
| LocalSingleBlockElimPass | `CreateLocalSingleBlockLoadStoreElimPass()` | 单块内局部变量加载/存储消除 |
| LocalSingleStoreElimPass | `CreateLocalSingleStoreElimPass()` | 单存储局部变量消除 |
| LocalMultiStoreElimPass | `CreateLocalMultiStoreElimPass()` | 多存储局部变量 SSA 消除 |
| InsertExtractElimPass | `CreateInsertExtractElimPass()` | 插入/提取消除 |
| DeadInsertElimPass | `CreateDeadInsertElimPass()` | 死插入消除 |
| AggressiveDCEPass | `CreateAggressiveDCEPass()` | 激进死代码消除 |
| DeadBranchElimPass | `CreateDeadBranchElimPass()` | 死分支消除 |
| BlockMergePass | `CreateBlockMergePass()` | 合并单前驱/单后继基本块 |
| EliminateDeadFunctionsPass | `CreateEliminateDeadFunctionsPass()` | 移除死函数 |
| EliminateDeadMembersPass | `CreateEliminateDeadMembersPass()` | 移除未使用的结构体成员 |
| DeadVariableEliminationPass | `CreateDeadVariableEliminationPass()` | 移除未引用的模块级变量 |
| MergeReturnPass | `CreateMergeReturnPass()` | 将多返回合并为单返回 |
| SSARewritePass | `CreateSSARewritePass()` | 将局部变量转换为 SSA 形式 |

##### 代码改进类 Pass

| Pass | 创建函数 | 描述 |
|------|----------|------|
| CCPass | `CreateCCPPass()` | 条件常量传播 |
| IfConversionPass | `CreateIfConversionPass()` | if-then-else 转换为 OpSelect |
| LICMPass | `CreateLoopInvariantCodeMotionPass()` | 循环不变代码外提 |
| LoopFissionPass | `CreateLoopFissionPass(threshold)` | 循环分裂 |
| LoopFusionPass | `CreateLoopFusionPass(max_registers)` | 循环融合 |
| LoopPeelingPass | `CreateLoopPeelingPass()` | 循环剥离 |
| LoopUnswitchPass | `CreateLoopUnswitchPass()` | 循环开关外提 |
| LoopUnrollPass | `CreateLoopUnrollPass(fully, factor)` | 循环展开 |
| SimplificationPass | `CreateSimplificationPass()` | 指令简化 |
| StrengthReductionPass | `CreateStrengthReductionPass()` | 强度削减 |
| ScalarReplacementPass | `CreateScalarReplacementPass(limit)` | 标量替换 |
| PrivateToLocalPass | `CreatePrivateToLocalPass()` | 私有变量转局部变量 |
| RedundancyEliminationPass | `CreateRedundancyEliminationPass()` | 全局值编号冗余消除 |
| LocalRedundancyEliminationPass | `CreateLocalRedundancyEliminationPass()` | 局部值编号冗余消除 |
| CopyPropagateArraysPass | `CreateCopyPropagateArraysPass()` | 数组拷贝传播 |
| VectorDCEPass | `CreateVectorDCEPass()` | 向量死代码消除 |
| ReduceLoadSizePass | `CreateReduceLoadSizePass(threshold)` | 减小加载大小 |
| CodeSinkingPass | `CreateCodeSinkingPass()` | 代码下沉 |

##### 规范化/兼容类 Pass

| Pass | 创建函数 | 描述 |
|------|----------|------|
| ConvertToHalfPass | `CreateConvertRelaxedToHalfPass()` | 转换为半精度 |
| RelaxFloatOpsPass | `CreateRelaxFloatOpsPass()` | 标记浮点操作为 RelaxedPrecision |
| FixStorageClassPass | `CreateFixStorageClassPass()` | 修复存储类不匹配 |
| ReplaceInvalidOpcodePass | `CreateReplaceInvalidOpcodePass()` | 替换无效操作码 |
| UpgradeMemoryModelPass | `CreateUpgradeMemoryModelPass()` | 升级内存模型到 VulkanKHR |
| AmdExtToKhrPass | `CreateAmdExtToKhrPass()` | AMD 扩展转 KHR |
| InterpolateFixupPass | `CreateInterpolateFixupPass()` | 修复插值指令 |
| GraphicsRobustAccessPass | `CreateGraphicsRobustAccessPass()` | 注入缓冲区边界检查 |
| Workaround1209Pass | `CreateWorkaround1209Pass()` | 驱动 bug 规避 |
| WrapOpKillPass | `CreateWrapOpKillPass()` | 包装 OpKill 为函数调用 |
| CombineAccessChainsPass | `CreateCombineAccessChainsPass()` | 合并链式访问链 |
| SpreadVolatileSemanticsPass | `CreateSpreadVolatileSemanticsPass()` | 传播 Volatile 语义 |
| TrimCapabilitiesPass | `CreateTrimCapabilitiesPass()` | 裁剪未使用的能力 |
| StructPackingPass | `CreateStructPackingPass(name, rule)` | 结构体打包 |
| SwitchDescriptorSetPass | `CreateSwitchDescriptorSetPass(from, to)` | 切换描述符集 |
| InvocationInterlockPlacementPass | `CreateInvocationInterlockPlacementPass()` | 插入互锁指令 |
| ModifyMaximalReconvergencePass | `CreateModifyMaximalReconvergencePass(add)` | 添加/移除最大重汇聚 |
| SplitCombinedImageSamplerPass | `CreateSplitCombinedImageSamplerPass()` | 拆分组合图像采样器 |
| ResolveBindingConflictsPass | `CreateResolveBindingConflictsPass()` | 解决绑定冲突 |
| DescriptorScalarReplacementPass | `CreateDescriptorScalarReplacementPass()` | 描述符标量替换 |
| InterfaceVariableScalarReplacementPass | `CreateInterfaceVariableScalarReplacementPass()` | 接口变量标量替换 |
| ReplaceDescArrayAccessUsingVarIndexPass | `CreateReplaceDescArrayAccessUsingVarIndexPass()` | 替换变量索引描述符访问 |
| LegalizeMultidimArrayPass | `CreateLegalizeMultidimArrayPass()` | 合法化多维数组 |
| RemoveDontInlinePass | `CreateRemoveDontInlinePass()` | 移除 DontInline 标记 |
| FixFuncCallArgumentsPass | `CreateFixFuncCallArgumentsPass()` | 修复函数调用参数 |
| OpExtInstForwardRefFixupPass | `CreateOpExtInstWithForwardReferenceFixupPass()` | 修复前向引用 |
| EliminateDeadInputComponentsPass | `CreateEliminateDeadInputComponentsPass()` | 消除死输入组件 |
| EliminateDeadOutputComponentsPass | `CreateEliminateDeadOutputComponentsPass()` | 消除死输出组件 |
| AnalyzeLiveInputPass | `CreateAnalyzeLiveInputPass()` | 分析活跃输入 |
| EliminateDeadOutputStoresPass | `CreateEliminateDeadOutputStoresPass()` | 消除死输出存储 |
| ConvertToSampledImagePass | `CreateConvertToSampledImagePass()` | 转换为采样图像 |

##### 预设优化配方

| 配方 | 注册方法 | 描述 |
|------|----------|------|
| 性能优化 (`-O`) | `RegisterPerformancePasses()` | 优化执行性能 |
| 大小优化 (`-Os`) | `RegisterSizePasses()` | 优化代码大小 |
| 合法化 (`--legalize-hlsl`) | `RegisterLegalizationPasses()` | 合法化 HLSL 生成的 SPIR-V |

---

### 4.3 验证器模块 (source/val/)

验证器检查 SPIR-V 模块是否符合 SPIR-V 规范中的验证规则。

#### 关键类

##### `ValidationState_t` — 验证状态

验证过程中的全局状态，跟踪所有验证信息。

```cpp
class ValidationState_t {
public:
    // 模块信息
    spv_target_env target_env();         // 目标环境
    uint32_t id_bound();                 // ID 上限

    // 指令查找
    Instruction* FindDef(uint32_t id);   // 通过 ID 查找指令

    // 函数信息
    Function* current_function();        // 当前验证的函数
    bool in_function_body();             // 是否在函数体内

    // 诊断
    spv_result_t diagnostic();           // 验证结果
};
```

##### `Function` (val) — 验证用函数

```cpp
class Function {
public:
    uint32_t id();                       // 函数 ID
    std::vector<BasicBlock*>& blocks();  // 基本块列表
    std::vector<uint32_t>& parameters(); // 参数列表
    bool IsCompatibleWithExecutionModel(SpvExecutionModel model);
};
```

##### `BasicBlock` (val) — 验证用基本块

```cpp
class BasicBlock {
public:
    uint32_t id();                       // 基本块 ID
    std::set<BasicBlock*>& predecessors();  // 前驱
    std::set<BasicBlock*>& successors();    // 后继
    bool reachable();                    // 是否可达
    SpvLoopControl loop_control();       // 循环控制
};
```

##### `Instruction` (val) — 验证用指令

```cpp
class Instruction {
public:
    SpvOp opcode();                      // 操作码
    uint32_t id();                       // 结果 ID
    uint32_t type_id();                  // 类型 ID
    const std::vector<Operand>& operands(); // 操作数列表
};
```

##### `Construct` — 结构化控制流构造

```cpp
class Construct {
public:
    ConstructType type();                // 构造类型 (Selection/Loop/Case)
    BasicBlock* entry_block();           // 入口块
    BasicBlock* merge_block();           // 合并块
};
```

#### 验证分类 (validate_*.cpp)

| 文件 | 验证内容 |
|------|----------|
| `validate_adjacency.cpp` | 指令邻接性 |
| `validate_annotation.cpp` | 注解指令 (OpDecorate 等) |
| `validate_arithmetics.cpp` | 算术指令 |
| `validate_atomics.cpp` | 原子指令 |
| `validate_barriers.cpp` | 屏障指令 |
| `validate_bitwise.cpp` | 位运算指令 |
| `validate_builtins.cpp` | 内建变量 |
| `validate_capability.cpp` | 能力声明 |
| `validate_cfg.cpp` | 控制流图 |
| `validate_composites.cpp` | 复合类型指令 |
| `validate_constants.cpp` | 常量指令 |
| `validate_conversion.cpp` | 类型转换指令 |
| `validate_debug.cpp` | 调试指令 |
| `validate_decorations.cpp` | 装饰规则 |
| `validate_derivatives.cpp` | 导数指令 |
| `validate_dot_product.cpp` | 点积指令 |
| `validate_execution_limitations.cpp` | 执行限制 |
| `validate_extensions.cpp` | 扩展指令 |
| `validate_function.cpp` | 函数指令 |
| `validate_graph.cpp` | 图指令 |
| `validate_group.cpp` | 组指令 |
| `validate_id.cpp` | ID 使用规则 |
| `validate_image.cpp` | 图像指令 |
| `validate_interfaces.cpp` | 接口变量 |
| `validate_layout.cpp` | 指令布局 |
| `validate_literals.cpp` | 字面量 |
| `validate_logicals.cpp` | 逻辑指令 |
| `validate_memory.cpp` | 内存指令 |
| `validate_mesh_shading.cpp` | 网格着色 |
| `validate_misc.cpp` | 杂项指令 |
| `validate_modes.cpp` | 执行模式 |
| `validate_non_uniform.cpp` | 非统一指令 |
| `validate_primitives.cpp` | 图元指令 |
| `validate_ray_query.cpp` | 光线查询 |
| `validate_ray_tracing.cpp` | 光线追踪 |
| `validate_scopes.cpp` | 内存/作用域 |
| `validate_small_type_uses.cpp` | 小类型使用 |
| `validate_type.cpp` | 类型声明 |
| `validate_type_unique.cpp` | 类型唯一性 |

---

### 4.4 链接器模块 (source/link/)

链接器将多个 SPIR-V 二进制模块合并为一个模块。

#### 关键文件

| 文件 | 职责 |
|------|------|
| `linker.cpp` | 链接器核心实现 |
| `fnvar.cpp/h` | 函数变体 (SPV_INTEL_function_variants) 支持 |

#### 关键函数

```cpp
// C++ API
spv_result_t Link(const Context& context,
                  const std::vector<std::vector<uint32_t>>& binaries,
                  std::vector<uint32_t>* linked_binary,
                  const LinkerOptions& options = LinkerOptions());
```

#### `LinkerOptions` — 链接选项

| 选项 | 默认值 | 说明 |
|------|--------|------|
| `GetCreateLibrary()` | false | 生成库(保留导出) vs 可执行文件 |
| `GetVerifyIds()` | false | 验证合并后 ID 唯一性 |
| `GetAllowPartialLinkage()` | false | 允许部分链接(未解析的导入) |
| `GetUseHighestVersion()` | false | 使用最高 SPIR-V 版本 |
| `GetAllowPtrTypeMismatch()` | false | 允许指针类型不匹配 |

---

### 4.5 模糊测试模块 (source/fuzz/)

模糊测试器对 SPIR-V 二进制模块应用语义保持的变换，产生等价模块，用于发现 SPIR-V 处理工具中的 bug。

#### 核心类

##### `Fuzzer` — 模糊测试器主类

```cpp
class Fuzzer {
public:
    Fuzzer(spv_target_env env, MessageConsumer consumer,
           std::unique_ptr<RandomGenerator> rng,
           bool enable_all_passes, bool is_wgsl_compatible);

    // 运行模糊测试
    Status Run(const std::vector<uint32_t>& binary,
               const spvtools::FuzzerOptions& options,
               std::vector<uint32_t>* transformed_binary,
               FuzzerResult* result);

    // 重放变换序列
    Status Replay(const std::vector<uint32_t>& binary,
                  const std::vector<protobufs::Transformation>& transformations,
                  std::vector<uint32_t>* transformed_binary);
};
```

##### `FuzzerPass` — 模糊测试 Pass 基类

```cpp
class FuzzerPass {
public:
    virtual ~FuzzerPass() = default;
    virtual void Apply() = 0;           // 应用变换

    FuzzerContext* GetFuzzerContext();   // 获取模糊上下文
    opt::IRContext* GetIRContext();      // 获取 IR 上下文
};
```

##### `Transformation` — 变换基类

```cpp
class Transformation {
public:
    virtual ~Transformation() = default;
    virtual bool IsApplicable(opt::IRContext* context,
                              const TransformationContext& context) const = 0;
    virtual void Apply(opt::IRContext* context,
                       TransformationContext* context) const = 0;
};
```

#### 模糊测试 Pass 列表 (部分)

| Pass | 描述 |
|------|------|
| FuzzerPassAddAccessChains | 添加访问链 |
| FuzzerPassAddBitInstructionSynonyms | 添加位指令同义词 |
| FuzzerPassAddCompositeExtract | 添加复合提取 |
| FuzzerPassAddCompositeInserts | 添加复合插入 |
| FuzzerPassAddCopyMemory | 添加内存拷贝 |
| FuzzerPassAddDeadBlocks | 添加死代码块 |
| FuzzerPassAddDeadBreaks | 添加死 break |
| FuzzerPassAddDeadContinues | 添加死 continue |
| FuzzerPassAddEquationInstructions | 添加等式指令 |
| FuzzerPassAddFunctionCalls | 添加函数调用 |
| FuzzerPassAddGlobalVariables | 添加全局变量 |
| FuzzerPassAddLoads | 添加加载 |
| FuzzerPassAddLocalVariables | 添加局部变量 |
| FuzzerPassAddLoopPreheaders | 添加循环预头 |
| FuzzerPassAddOpPhiSynonyms | 添加 OpPhi 同义词 |
| FuzzerPassAddParameters | 添加函数参数 |
| FuzzerPassAddStores | 添加存储 |
| FuzzerPassAddSynonyms | 添加同义词 |
| FuzzerPassConstructComposites | 构造复合值 |
| FuzzerPassCopyObjects | 拷贝对象 |
| FuzzerPassDonateModules | 捐赠模块 |
| FuzzerPassInlineFunctions | 内联函数 |
| FuzzerPassMergeBlocks | 合并基本块 |
| FuzzerPassObfuscateConstants | 混淆常量 |
| FuzzerPassOutlineFunctions | 提取函数 |
| FuzzerPassPermuteBlocks | 排列基本块 |
| FuzzerPassSplitBlocks | 分割基本块 |
| ... | (更多) |

---

### 4.6 缩减器模块 (source/reduce/)

缩减器简化/缩小 SPIR-V 模块，同时保持用户定义的"有趣性"条件。

#### 核心类

##### `Reducer` — 缩减器主类

```cpp
class Reducer {
public:
    Reducer(spv_target_env env, MessageConsumer consumer);

    // 运行缩减
    Status Run(const std::vector<uint32_t>& binary,
               const spvtools::ReducerOptions& options,
               std::vector<uint32_t>* reduced_binary);
};
```

##### `ReductionOpportunity` — 缩减机会基类

```cpp
class ReductionOpportunity {
public:
    virtual ~ReductionOpportunity() = default;
    virtual bool PreconditionHolds() = 0;  // 前置条件检查
    virtual void Apply() = 0;              // 应用缩减
};
```

##### `ReductionOpportunityFinder` — 缩减机会查找器基类

```cpp
class ReductionOpportunityFinder {
public:
    virtual ~ReductionOpportunityFinder() = default;
    virtual std::vector<std::unique_ptr<ReductionOpportunity>>
        GetAvailableOpportunities(opt::IRContext* context) const = 0;
    virtual std::string name() const = 0;
};
```

#### 内置缩减策略

| Finder | 描述 |
|--------|------|
| `ConditionalBranchToSimpleConditionalBranchOpportunityFinder` | 条件分支简化 |
| `MergeBlocksReductionOpportunityFinder` | 合并基本块 |
| `OperandToConstReductionOpportunityFinder` | 操作数替换为常量 |
| `OperandToUndefReductionOpportunityFinder` | 操作数替换为 undef |
| `OperandToDominatingIdReductionOpportunityFinder` | 操作数替换为支配 ID |
| `RemoveBlockReductionOpportunityFinder` | 移除基本块 |
| `RemoveFunctionReductionOpportunityFinder` | 移除函数 |
| `RemoveInstructionReductionOpportunityFinder` | 移除指令 |
| `RemoveSelectionReductionOpportunityFinder` | 移除选择 |
| `RemoveUnusedInstructionReductionOpportunityFinder` | 移除未使用指令 |
| `RemoveUnusedStructMemberReductionOpportunityFinder` | 移除未使用结构体成员 |
| `SimpleConditionalBranchToBranchOpportunityFinder` | 简单条件分支转无条件 |
| `StructuredConstructToBlockReductionOpportunityFinder` | 结构化构造转基本块 |
| `StructuredLoopToSelectionReductionOpportunityFinder` | 结构化循环转选择 |

---

### 4.7 差异比较模块 (source/diff/)

差异比较工具对两个 SPIR-V 模块进行 diff 风格的比较。

#### 关键文件

| 文件 | 职责 |
|------|------|
| `diff.cpp/h` | 差异比较核心实现 |
| `lcs.h` | 最长公共子序列算法 |

#### 关键函数

```cpp
// 执行差异比较
spv_result_t Diff(spv_const_context context,
                  const std::vector<uint32_t>& src,
                  const std::vector<uint32_t>& dst,
                  spv_text* diff_text,
                  uint32_t options);
```

---

### 4.8 工具库 (source/util/)

通用工具库，提供基础数据结构和算法支持。

| 文件 | 描述 |
|------|------|
| `bit_vector.cpp/h` | 紧凑位向量 |
| `bitutils.h` | 位操作工具 |
| `hash_combine.h` | 哈希组合工具 |
| `hex_float.h` | 十六进制浮点数解析 |
| `ilist.h` | 侵入式双向链表 |
| `ilist_node.h` | 侵入式链表节点 |
| `index_range.h` | 索引范围迭代器 |
| `make_unique.h` | MakeUnique 工具 |
| `parse_number.cpp/h` | 数字解析 |
| `small_vector.h` | 小型向量优化 (SSO) |
| `span.h` | 非拥有视图 (span) |
| `status.h` | 状态码定义 |
| `string_utils.cpp/h` | 字符串工具 |
| `timer.cpp/h` | 计时器 |

---

## 5. 公共 API 接口

### C API (libspirv.h)

| 函数 | 描述 |
|------|------|
| `spvContextCreate(env)` | 创建上下文 |
| `spvContextDestroy(context)` | 销毁上下文 |
| `spvTextToBinary(...)` | 汇编: 文本 -> 二进制 |
| `spvTextToBinaryWithOptions(...)` | 带选项汇编 |
| `spvBinaryToText(...)` | 反汇编: 二进制 -> 文本 |
| `spvBinaryParse(...)` | 二进制解析(回调式) |
| `spvValidate(...)` | 验证 |
| `spvValidateWithOptions(...)` | 带选项验证 |
| `spvValidateBinary(...)` | 原始二进制验证 |
| `spvOptimizerCreate(env)` | 创建优化器 |
| `spvOptimizerDestroy(optimizer)` | 销毁优化器 |
| `spvOptimizerRegisterPassFromFlag(...)` | 注册优化 Pass |
| `spvOptimizerRun(...)` | 运行优化 |
| `spvDiagnosticCreate(...)` | 创建诊断对象 |
| `spvDiagnosticDestroy(...)` | 销毁诊断对象 |
| `spvSoftwareVersionString()` | 获取版本字符串 |

### C++ API (libspirv.hpp)

| 类 | 描述 |
|----|------|
| `Context` | RAII 包装的 spv_context |
| `SpirvTools` | 汇编/反汇编/验证接口 |
| `ValidatorOptions` | 验证器选项 |
| `OptimizerOptions` | 优化器选项 |
| `ReducerOptions` | 缩减器选项 |
| `FuzzerOptions` | 模糊测试选项 |

#### `SpirvTools` 类方法

```cpp
class SpirvTools {
public:
    explicit SpirvTools(spv_target_env env);
    void SetMessageConsumer(MessageConsumer consumer);

    bool Assemble(const std::string& text, std::vector<uint32_t>* binary,
                  uint32_t options = kDefaultAssembleOption) const;
    bool Disassemble(const std::vector<uint32_t>& binary, std::string* text,
                     uint32_t options = kDefaultDisassembleOption) const;
    bool Parse(const std::vector<uint32_t>& binary, ...);
    bool Validate(const std::vector<uint32_t>& binary) const;
    bool Validate(const uint32_t* binary, size_t binary_size,
                  spv_validator_options options) const;
};
```

### C++ 优化器 API (optimizer.hpp)

```cpp
class Optimizer {
public:
    explicit Optimizer(spv_target_env env);
    Optimizer& RegisterPass(PassToken&& pass);
    Optimizer& RegisterPerformancePasses();
    Optimizer& RegisterSizePasses();
    Optimizer& RegisterLegalizationPasses();
    bool RegisterPassesFromFlags(const std::vector<std::string>& flags);
    bool Run(const uint32_t* original_binary, size_t size,
             std::vector<uint32_t>* optimized_binary) const;
};

// 创建各种 Pass Token
Optimizer::PassToken CreateStripDebugInfoPass();
Optimizer::PassToken CreateAggressiveDCEPass();
// ... (见 4.2 节完整列表)
```

### C++ 链接器 API (linker.hpp)

```cpp
class LinkerOptions { /* 见 4.4 节 */ };

spv_result_t Link(const Context& context,
                  const std::vector<std::vector<uint32_t>>& binaries,
                  std::vector<uint32_t>* linked_binary,
                  const LinkerOptions& options = LinkerOptions());
```

---

## 6. 命令行工具

| 工具 | 路径 | 描述 |
|------|------|------|
| `spirv-as` | `tools/as/as.cpp` | 汇编器: 文本 -> 二进制 |
| `spirv-dis` | `tools/dis/dis.cpp` | 反汇编器: 二进制 -> 文本 |
| `spirv-val` | `tools/val/val.cpp` | 验证器 |
| `spirv-opt` | `tools/opt/opt.cpp` | 优化器 |
| `spirv-link` | `tools/link/linker.cpp` | 链接器 |
| `spirv-cfg` | `tools/cfg/cfg.cpp` | 控制流图导出 (GraphViz) |
| `spirv-fuzz` | `tools/fuzz/fuzz.cpp` | 模糊测试器 |
| `spirv-reduce` | `tools/reduce/reduce.cpp` | 缩减器 |
| `spirv-diff` | `tools/diff/diff.cpp` | 差异比较 |
| `spirv-lint` | `tools/lint/lint.cpp` | 代码检查 |
| `spirv-objdump` | `tools/objdump/objdump.cpp` | 对象转储 |

### 常用命令示例

```bash
# 汇编
spirv-as input.spvasm -o output.spv

# 反汇编
spirv-dis input.spv -o output.spvasm

# 验证
spirv-val input.spv

# 优化 (性能)
spirv-opt -O input.spv -o output.spv

# 优化 (大小)
spirv-opt -Os input.spv -o output.spv

# 优化 (指定 pass)
spirv-opt --strip-debug --inline-entry-points-exhaustive input.spv -o output.spv

# 链接
spirv-link a.spv b.spv -o linked.spv

# 控制流图
spirv-cfg input.spv -o graph.dot

# 差异比较
spirv-diff src.spv dst.spv

# 缩减
spirv-reduce input.spv --interestingness="spirv-val %spv"

# 模糊测试
spirv-fuzz input.spv -o output.spv --seed=42
```

---

## 7. 模块间依赖关系

```
                    ┌──────────────────┐
                    │    命令行工具     │
                    │ (spirv-as, etc.) │
                    └────────┬─────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
    ┌─────────▼──────┐ ┌────▼─────┐ ┌──────▼──────┐
    │  libSPIRV-Tools │ │ Optimizer│ │   Linker    │
    │  (核心库)       │ │ (优化库)  │ │  (链接库)   │
    │                │ │          │ │             │
    │ • 汇编/反汇编  │ │ • Pass   │ │ • 合并模块  │
    │ • 二进制解析   │ │ • IR     │ │ • 符号解析  │
    │ • 验证器      │ │ • 分析   │ │             │
    │ • 诊断       │ │          │ │             │
    └────────┬───────┘ └────┬─────┘ └──────┬──────┘
             │              │              │
             │         ┌────▼─────┐        │
             │         │ libSPIRV │◄───────┘
             │         │ -Tools   │
             │         │ (核心库)  │
             │         └──────────┘
             │
    ┌────────▼───────────────────────────────────────┐
    │              SPIRV-Headers                      │
    │  (语法 JSON 文件, 枚举定义, 头文件)              │
    └────────────────────────────────────────────────┘

    扩展模块依赖:
    ┌────────┐    ┌────────┐    ┌────────┐
    │ Fuzzer │───▶│Reducer │───▶│  Diff  │
    │        │    │        │    │        │
    └───┬────┘    └───┬────┘    └───┬────┘
        │             │              │
        ▼             ▼              ▼
    ┌─────────────────────────────────────┐
    │        libSPIRV-Tools-opt           │
    │        (优化器库)                    │
    └─────────────────┬───────────────────┘
                      │
                      ▼
    ┌─────────────────────────────────────┐
    │        libSPIRV-Tools               │
    │        (核心库)                      │
    └─────────────────────────────────────┘
```

### 依赖关系说明

1. **libSPIRV-Tools** (核心库): 无其他内部库依赖，仅依赖 SPIRV-Headers
2. **libSPIRV-Tools-opt** (优化器库): 依赖 libSPIRV-Tools
3. **libSPIRV-Tools-link** (链接器库): 依赖 libSPIRV-Tools-opt 和 libSPIRV-Tools
4. **Fuzzer**: 依赖 libSPIRV-Tools-opt + protobuf
5. **Reducer**: 依赖 libSPIRV-Tools-opt
6. **Diff**: 依赖 libSPIRV-Tools-opt
7. **所有命令行工具**: 依赖对应的库

---

## 8. 项目运行方式

### 获取源码

```bash
git clone https://github.com/KhronosGroup/SPIRV-Tools.git spirv-tools
cd spirv-tools
python3 utils/git-sync-deps
```

### 使用 CMake 构建

```bash
mkdir build && cd build
cmake [-G <generator>] <spirv-dir>
cmake --build . [--config Debug]
```

### 构建 Fuzzer (需要 protobuf)

```bash
git clone --depth=1 --branch v3.13.0.1 \
    https://github.com/protocolbuffers/protobuf external/protobuf
mkdir build && cd build
cmake <spirv-dir> -DSPIRV_BUILD_FUZZER=ON
cmake --build . --config Debug
```

### 使用 Bazel 构建

```bash
bazel build :all
bazel test --cxxopt=-std=c++17 :all
```

### 构建 Android 静态库

```bash
export ANDROID_NDK=/path/to/ndk
mkdir build && cd build
$ANDROID_NDK/ndk-build -C ../android_test \
    NDK_PROJECT_PATH=. \
    NDK_LIBS_OUT=`pwd`/libs \
    NDK_APP_OUT=`pwd`/app
```

### 构建 WebAssembly 模块

```bash
# 需要 Emscripten SDK
./source/wasm/build.sh
node ./test/wasm/test.js
```

### 运行测试

```bash
# CMake
ctest -j$(nproc)
ctest -R 'spirv-tools-test_opt'

# Bazel
bazel test --cxxopt=-std=c++17 :all
```

---

## 9. 测试体系

### 测试框架

- **googletest**: C++ 单元测试框架
- **Effcee**: 状态匹配测试 (用于优化器输出验证)

### 测试目录结构

| 目录 | 内容 |
|------|------|
| `test/` (根) | 核心库测试 (汇编/反汇编/解析等) |
| `test/opt/` | 优化器测试 |
| `test/val/` | 验证器测试 |
| `test/link/` | 链接器测试 |
| `test/fuzz/` | 模糊测试器测试 |
| `test/reduce/` | 缩减器测试 |
| `test/diff/` | 差异比较测试 |
| `test/lint/` | 代码检查测试 |
| `test/util/` | 工具库测试 |
| `test/fuzzers/` | libFuzzer 目标 |
| `test/tools/` | 命令行工具测试 |
| `test/wasm/` | WebAssembly 测试 |

### 测试规模

- 核心库测试: ~50+ 测试文件
- 优化器测试: ~80+ 测试文件
- 验证器测试: ~50+ 测试文件
- 模糊测试器测试: ~60+ 测试文件 (transformation 测试)
- 缩减器测试: ~15+ 测试文件

---

> 本文档基于 SPIRV-Tools v2026.2 源码分析生成，最后更新: 2026-05-30
