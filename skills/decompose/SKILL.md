---
name: decompose
description: 此 Skill 用于在文件组粒度上进行内部细粒度分解。触发场景：用户提供一组耦合文件（.h+.cpp+CMakeLists）要求"分解"、"/decompose"、从 /split-commit 输出中选择一个文件组进行处理等。此 Skill 分析头文件依赖、函数声明、控制流结构，输出分阶段实现计划（接口→算法→边界），标注每阶段的置信度。
argument-hint: "<file1.h> <file1.cpp> [file2.h] ... [--repo <path>]"
---

# 文件组细粒度分解

## 职责

接收一组耦合文件（.h + .cpp + CMakeLists），将其内部变更分解为可独立实现的子阶段。输出分阶段计划（接口→核心算法→边界处理），供 `/micro` 逐阶段执行。**核心价值在于：1. 分解文件内部的依赖关系为若干可线性提交的commit组件；2. 处理超级函数——数百行包含大量分支处理的函数，识别其控制流边界并评估分解可行性。**

## 不做什么

- 不做测试匹配（测试是 `/micro` 的职责）
- 不执行代码变更
- 不物理切割文件——只输出阶段化实现计划

## 输入

- **必需**：一组耦合文件的路径（.h + .cpp + 对应的 CMakeLists.txt）

## 工作流

### Step 1: 确认文件组

列出接收的文件组：

```
文件组:

  接口:   include/MyPass.h           (XX 行)
  实现:   lib/MyPass/MyPass.cpp       (~XXX 行)
  构建:   lib/MyPass/CMakeLists.txt   (X 行)
```

若缺少明显耦合文件（如无 .h 或无 CMakeLists），提示用户补充。

### Step 2: 头文件分析

读取头文件，提取类/接口声明和公开方法签名。

```
接口分析: MyPass.h

  类:     MyPass : public PassInfoMixin<MyPass>
  公开方法:
    PreservedAnalyses run(Function &F, FunctionAnalysisManager &AM)
  依赖:
    #include "llvm/IR/PassManager.h"
    #include "llvm/Analysis/LoopInfo.h"
```

### Step 3: 函数级分析

读取实现文件，识别主要函数的控制流结构和核心分支。

**普通函数**（< 80 行）：直接标注，无需分支分类。

**超级函数**（≥ 80 行）：基于控制流节点进行分支分类。

```
超级函数分析: MyPass::run (Function &F, ...)  ~280 行

  控制流结构:
    │
    ├── [A] 初始化: 获取 LoopInfo, MemorySSA      (lines 45-70, ~25 行)
    │   置信度: 高 — 独立初始化逻辑
    │
    ├── [B] 核心匹配: 遍历循环，匹配归约模式       (lines 71-140, ~70 行)
    │   置信度: 高 — 主循环体，与周边松耦合
    │
    ├── [C] 正常替换: 双层嵌套循环的规约替换       (lines 141-210, ~70 行)
    │   置信度: 高 — 仅在 [B] 匹配成功后触发
    │
    ├── [D] 边缘路径: 单层循环透传                  (lines 211-240, ~30 行)
    │   置信度: 中 — 和 [B] 共享部分匹配状态，但逻辑独立
    │
    └── [E] 错误处理: 不支持的模式标记 + 原样返回   (lines 241-275, ~35 行)
        置信度: 高 — 独立错误分支
```

**置信度标注规则**：

| 标注 | 含义 | 处理方式 |
|------|------|---------|
| **高** | 边界清晰，控制流独立，可安全拆为独立 Phase | 正常分解 |
| **中** | 与相邻分支共享少量局部状态，但逻辑边界仍可识别 | 分解，但标注"注意状态依赖" |
| **不确定** | 控制流纠缠——多处 goto/状态共享/多重嵌套条件 | 归入相邻高置信度 Phase，或整体保留，建议人工截断 |

### Step 4: 构建分阶段实现计划

将分析结果转化为有序的 Phase 序列。

```
分阶段实现计划: MyPass 文件组

估算总行数: ~310 (含已有变更)
建议分解为 4 个 Phase:

---

Phase 1: 接口定义与 Pass 骨架
  文件: include/MyPass.h, lib/MyPass/CMakeLists.txt
  行数: ~40
  方法: run() 声明, 空实现 return PreservedAnalyses::all()
  测试入口建议 (供 /micro 参考):
    - 验证: opt -load-pass-plugin=... -passes=my-pass <empty.ll> 正常退出
  置信度: 高

---

Phase 2: 核心匹配逻辑
  文件: lib/MyPass/MyPass.cpp
  对应分支: [A] 初始化 + [B] 核心匹配
  行数: ~95
  测试入口建议 (供 /micro 参考):
    - 正常场景: 双层嵌套循环 IR，验证匹配成功 (FileCheck: "Matched: 1 loops")
    - 负面场景: 非归约循环 IR，验证不匹配 (FileCheck: "Matched: 0 loops")
  置信度: 高

---

Phase 3: 正常替换 + 边缘路径
  文件: lib/MyPass/MyPass.cpp
  对应分支: [C] 正常替换 + [D] 单层循环透传
  行数: ~100
  测试入口建议 (供 /micro 参考):
    - 正常场景: 双层嵌套循环 IR → 验证 IR 变换后输出
    - 边缘场景: 单层循环 IR → 验证不做变换 (输出与输入一致)
  置信度: 中 (Phase 2 和 Phase 3 共享匹配状态，建议 Phase 3 在 Phase 2 测试全部通过后实现)

---

Phase 4: 错误处理
  文件: lib/MyPass/MyPass.cpp
  对应分支: [E] 不支持模式
  行数: ~35
  测试入口建议 (供 /micro 参考):
    - 含非规约模式 IR → 验证输出含 "unsupported" 标记且原始 IR 未变
  置信度: 高
```

### Step 5: 纠缠边界处理

若 Step 3 中检测到"不确定"标注的代码区域：

```
⚠ 检测到纠缠边界:

  函数 MyPass::run 的 lines 150-175 与相邻分支 [B] 和 [C] 之间存在
  多处局部状态共享 (LoopNestState, MatchResult)，无法在高置信度下
  分离此段代码。

  建议: 将 lines 150-175 归入 Phase 2，并在 Phase 3 实现时
        重构接口以消除隐式状态依赖。当前方案暂不强行分解此区域。
```

## 输出边界

- 若文件组 ≤40 行且仅含单一函数/结构 → "此文件组结构简单，无需分解。可直接交由 `/micro` 执行。"
- 若文件组不含实现文件（.cpp/.c） → 跳过 Step 3，直接按头文件+构建输出 Phase

## 与 `/micro` 的协作

`/decompose` 输出的每个 Phase 带"测试入口建议"字段，为 `/micro` 提供测试场景参考。`/micro` 根据此字段定位测试文件并执行测试匹配——`/decompose` 不实际匹配测试。

## 与 `/split-commit` 的协作

`/split-commit` 输出文件组清单并标注耦合关系。`/decompose` 接收其中一个文件组，进行内部细粒度分解。用户工作流：

```
/split-commit <hash>
  → 输出: 3 个模块, 5 个耦合文件组

/decompose lib/Foo/FooPass.h lib/Foo/FooPass.cpp lib/Foo/CMakeLists.txt
  → 输出: 4 个 Phase

/micro "Phase 1: 接口定义与 Pass 骨架"
/micro "Phase 2: 核心匹配逻辑"
...
```
