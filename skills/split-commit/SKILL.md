---
name: split-commit
description: 此 Skill 用于分析大型 commit 并生成拆分方案。触发场景：用户提供 commit hash 并要求"拆分"、"分解"、"分析这个 commit"、"/split-commit"等。此 Skill 读取目标仓库中指定 commit 的 diff，按功能边界和独立验证原则将其分解为多个子 commit，输出结构化拆分方案。适用对象为 AI 库或外部仓库的大 commit（>200 行），不适用于已遵循规范的小 commit。
argument-hint: "<commit-hash> [--repo <path>]"
---

# Commit 拆分分析

## 职责

读取一个大 commit 的完整 diff，理解其变更语义，将其分解为可独立验证、可独立提交的子 commit 序列。**不执行任何 git 操作**——只输出拆分方案，供用户审查后通过 `/micro` 逐个执行。

## 不做什么

- 不执行 `git rebase -i`、`git reset`、`git cherry-pick` 等任何 git 写操作
- 不适用于本身已 ≤200 行且单一职责的 commit（直接告诉用户"此 commit 无需拆分"）
- 不拆分后直接提交——每个子 commit 需用户通过 `/micro` 单独实现和审查

## 输入

- **必需**: commit hash（AI 库或外部仓库）
- **可选**: `--repo <path>` 指定仓库路径（默认从对话上下文推断）

## 工作流

### Step 1: 路径级扫描（极省 token）

```bash
git -C <repo> show <commit-hash> --name-only    # 仅文件路径，不取 diff 内容
```

统计总文件数和净增行数（`git show <hash> --stat` 的最后一行）。

### Step 1.5: 加载项目规范

读取 `docs/Principle/COMMIT_PRINCIPLES.md` 中的以下核心规则。在后续拆分中，**任何违反这些规则的子 commit 方案都必须被拒绝**：

- **规则 2（测试完备性）**：所有代码变更必须附带测试。新功能没有测试 = 不合规。
- **规则 3（独立验证）**：每个 commit 必须可独立编译和测试。如果某个子 commit 的"独立验证"栏写不出具体可执行的命令，说明拆分有误。
- **规则 4（单一关注点）**：一个 commit 只做一件事。一个子 commit 不能同时包含 bugfix 和新功能。

### Step 1.6: 模块预扫描

使用 Step 1 的 `--name-only` 输出，提取非测试目录的路径，识别"模块"。

**模块定义**：项目源码根下第一级有意义的子目录。`tests/` 不算独立模块，测试归属跟随其被测模块。

示例——不同项目的模块识别规则：

| 项目 | 模块根 | 模块示例 | 说明 |
|------|--------|---------|------|
| LLVM | `llvm/lib/<name>` | `Transforms/Scalar`, `Analysis`, `CodeGen` | llvm/lib 下的顶级目录为模块；`Transforms/` 子目录也各自独立 |
| Triton | `python/triton/<name>` 或 `lib/<name>` | `language`, `runtime`, `compiler` | 按语言层/运行时/编译器分层 |
| MLIR | `lib/<name>` | `Dialect`, `Transforms`, `Conversion` | 每个 Dialect 或 Transform 组为一个模块 |
| GCC | `gcc/<name>` | `tree-ssa`, `tree-vect`, `ipa` | 每个优化/分析组件为一个模块 |

**判定逻辑**：

```
统计 distinct 模块数量 (不包含 tests/)

  < 3 个模块 → 单模块模式 → 直接进入 Step 2（加载完整 diff）
  ≥ 3 个模块 → 多模块模式 → 进入 Step 1.7
```

**注意**：即使 commit < 200 行，如果触及 ≥ 3 个模块也应触发多模块预扫描（可能是横切变更，需确认依赖关系后才能正确拆分）。

### Step 1.7: 多模块路径

列出涉及的模块及各自的文件数和变更行数。示例如下：

```
此 commit 涉及 4 个模块:

  llvm/lib/Analysis          (2 文件, ~95 行)
  llvm/lib/Transforms/Scalar (3 文件, ~210 行)
  llvm/lib/Transforms/Utils  (1 文件, ~30 行)
  llvm/test/Transforms/Scalar (5 文件, ~280 行)

请描述依赖关系:

  Scalar 依赖 Analysis
  Scalar 依赖 Utils
  test 与 Scalar 绑定

如果跳过此步，我将从 CMakeLists.txt 和 #include 引用自动推断。
>
```

**用户提供依赖后**：按描述的依赖顺序确定模块的拆分优先级（被依赖者先拆）。

**用户跳过时**：AI 自动执行以下推断：

```bash
# 1. 查看各模块 CMakeLists.txt 中的 target_link_libraries
# 2. 查看头文件 #include 关系
# 3. 输出推断的依赖关系，标注"自动推断"供用户验证
```

模块排序完成后，进入 Step 2。

### Step 2: 加载 diff 并理解变更语义

**单模块模式**（< 3 个模块）：直接加载完整 diff。

```bash
git -C <repo> show <commit-hash>
```

**多模块模式**（≥ 3 个模块）：按 Step 1.7 确定的模块顺序，逐模块从 diff 中提取变更片段。每个模块只加载属于它的文件变更，不一次性加载全部 diff。

逐文件阅读 diff 时，识别以下信号：

| 信号 | 含义 | 拆分优先级 |
|------|------|-----------|
| 新增头文件/纯类型定义（无成员函数实现） | 纯数据结构 | 模块内第1个子commit（可暂不绑定测试） |
| 新增 Pass/功能函数 | 功能单元 | 核心子commit（必须绑定对应测试） |
| 修改已有函数 | 重构/适配 | 与被影响的测试绑定 |
| CMakeLists.txt / BUILD 变更 | 构建配置 | 与对应的新增文件打包 |
| 仅格式/注释变更 | NFC | 独立 commit，标记 `[NFC]` |
| 错误处理/边界处理 | 辅助逻辑 | 与所服务的功能绑定 |

**注意**：测试没有独立的拆分优先级。测试文件的存在意义是验证实现代码——它们不独立构成子 commit。

### Step 3: 构建拆分方案

按以下优先级组织子 commit（数字越小优先级越高，违反时需要明确理由）：

1. **实现代码和其测试永远在同一个子 commit 中**。这是最优先硬约束。唯一例外：纯数据结构定义（头文件仅含 struct/class 声明，无成员函数实现）。
2. **每个子 commit 可独立编译**。编译 = 头文件 + 实现 + 测试 + 必要的 CMakeLists 变更打包。
3. **每个子 commit 可独立验证**。如果"独立验证"栏写不出具体可执行命令，说明拆分有问题。
4. **跨目录打包**。测试与实现可能在不同目录——实现目录和测试目录的分离是大多数编译项目的常规布局。独立验证优先于目录纯洁性。例如：
   - LLVM: 被测代码在 `llvm/lib/Transforms/Scalar/`，测试在 `llvm/test/Transforms/Scalar/` → 同一个子 commit
   - Triton: 被测代码在 `python/triton/compiler/`，测试在 `python/test/unit/compiler/` → 同一个子 commit
   - GCC: 被测代码在 `gcc/tree-ssa/`，测试在 `gcc/testsuite/gcc.dg/tree-ssa/` → 同一个子 commit
5. **基础设施先于功能**。数据结构 → 核心算法 → 边界处理。多模块模式下，被依赖模块先于依赖模块。
6. **每个子 commit ≤ 200 行**。如果实现+测试合并后超标，将功能拆为多个更小的子功能，每个子功能 = 自己的实现 + 自己的测试。

### Step 4: 一次性输出完整拆分方案

格式固定如下——多模块模式下模块依赖概览在前，各模块的细粒度拆分在后，所有内容一次性输出。

```markdown
## 拆分方案: commit <hash> "<title>"

来源: <repo-path>
原始规模: <total> 行, <N> 文件, <M> 模块

---

### 模块依赖概览

  llvm/lib/Analysis → (无依赖，基础设施)
  llvm/lib/Transforms/Scalar → 依赖 Analysis
  llvm/test/Transforms/Scalar → 与 Scalar 功能绑定

  (此依赖关系由<用户提供/AI 推断>)

---

### 建议拆分为 <K> 个独立 commit:

---

### Commit 1/4: `[Component] <标题>` (~XX 行)
**模块**: <模块目录>
**文件**:
  - path/to/interface.h (+XX)
  - path/to/CMakeLists.txt (+XX)
**测试**: 无（纯数据结构/接口，无行为函数）
**内容**: <一句话描述>
**独立验证**: 编译通过 (ninja <target>)
**依赖**: 无

---

### Commit 2/4: `[Component] <标题>` (~XX 行)
**模块**: <模块目录>
**文件**:
  - path/to/core_impl.cpp (+XX)
  - test/path/to/basic.ll (+XX)
**测试**: basic.ll — 验证最简场景下的正确性
**内容**: <一句话描述>
**独立验证**: opt -passes=... basic.ll -S | FileCheck basic.ll
**依赖**: Commit 1 (依赖接口定义)

---

### Commit 3/4: `[Component] <标题>` (~XX 行)
**模块**: <模块目录>
**文件**:
  - path/to/extended_impl.cpp (+XX)
  - test/path/to/complex.ll (+XX)
**测试**: complex.ll — 验证复杂场景下的正确性
**内容**: <一句话描述>
**独立验证**: opt -passes=... complex.ll -S | FileCheck complex.ll
**依赖**: Commit 2

---

### Commit 4/4: `[Component] <标题>` (~XX 行)
**模块**: <模块目录>
**文件**:
  - path/to/edge_handler.cpp (+XX)
  - test/path/to/edge-cases.ll (+XX)
**测试**: edge-cases.ll — 验证边界条件
**内容**: <一句话描述>
**独立验证**: opt -passes=... edge-cases.ll -S | FileCheck edge-cases.ll
**依赖**: Commit 3

---

### 依赖图:
  [1] → [2] → [3] → [4]

### 建议实施顺序:
  1. /micro "Commit 1/4 的内容描述"
  2. /micro "Commit 2/4 的内容描述"
  3. /micro "Commit 3/4 的内容描述"
  4. /micro "Commit 4/4 的内容描述"
```

以下是该模板在 LLVM 中的一个完整填充示例——commit `d4e5f6`，为一个新增的 LoopUnswitch Pass：

```markdown
## 拆分方案: commit d4e5f6 "[LoopUnswitch] Add trivial unswitching"

来源: llvm-project
原始规模: ~420 行, 11 文件, 3 模块

---

### 模块依赖概览

  llvm/lib/Analysis → (无依赖，分析接口)
  llvm/lib/Transforms/Scalar → 依赖 Analysis
  llvm/test/Transforms/Scalar → 与 Scalar 功能绑定

  (此依赖关系由 AI 推断)

---

### 建议拆分为 4 个独立 commit:

---

### Commit 1/4: `[Analysis] Add LoopSafetyInfo interface` (~65 行)
**模块**: llvm/lib/Analysis
**文件**:
  - llvm/include/llvm/Analysis/LoopSafetyInfo.h (+45)
  - llvm/lib/Analysis/CMakeLists.txt (+2)
**测试**: 无（纯数据结构/接口定义，无行为函数）
**内容**: 定义 LoopSafetyInfo 接口，封装循环不变性检查逻辑
**独立验证**: 编译通过 (ninja LLVMAnalysis)
**依赖**: 无

---

### Commit 2/4: `[LoopUnswitch] Implement trivial unswitch for invariant branches` (~150 行)
**模块**: llvm/lib/Transforms/Scalar
**文件**:
  - llvm/lib/Transforms/Scalar/LoopUnswitch.cpp (+120)
  - llvm/test/Transforms/LoopUnswitch/trivial.ll (+30)
**测试**: trivial.ll — 验证最简场景（单分支、条件为常数）
**内容**: 实现 trivial unswitch 的核心变换逻辑
**独立验证**: opt -passes=loop-unswitch trivial.ll -S | FileCheck trivial.ll
**依赖**: Commit 1 (依赖 LoopSafetyInfo)

---

### Commit 3/4: `[LoopUnswitch] Handle non-trivial conditions with MemorySSA` (~145 行)
**模块**: llvm/lib/Transforms/Scalar
**文件**:
  - llvm/lib/Transforms/Scalar/LoopUnswitch.cpp (+105)
  - llvm/test/Transforms/LoopUnswitch/nontrivial.ll (+40)
**测试**: nontrivial.ll — 验证含内存依赖的复杂条件
**内容**: 扩展 trivial unswitch 到需 MemorySSA 分析的场景
**独立验证**: opt -passes=loop-unswitch nontrivial.ll -S | FileCheck nontrivial.ll
**依赖**: Commit 2

---

### Commit 4/4: `[LoopUnswitch] Add boundary handling for switch-inst and empty loops` (~60 行)
**模块**: llvm/lib/Transforms/Scalar
**文件**:
  - llvm/lib/Transforms/Scalar/LoopUnswitch.cpp (+35)
  - llvm/test/Transforms/LoopUnswitch/edge-cases.ll (+25)
**测试**: edge-cases.ll — 验证 switch-inst 和空循环体
**内容**: 边界处理——不支持的输入原样返回
**独立验证**: opt -passes=loop-unswitch edge-cases.ll -S | FileCheck edge-cases.ll
**依赖**: Commit 3

---

### 依赖图:
  [1] → [2] → [3] → [4]

### 建议实施顺序:
  1. /micro "Commit 1/4: 定义 LoopSafetyInfo 接口"
  2. /micro "Commit 2/4: 实现 trivial unswitch + 最简测试"
  3. /micro "Commit 3/4: 扩展 MemorySSA 复杂场景 + 测试"
  4. /micro "Commit 4/4: switch-inst 边界处理 + 测试"
```

### Step 5: 若 commit 无需拆分

若 commit 满足以下所有条件，不输出拆分方案：

```
净增行数 ≤ 200 且 模块数 < 3 且 变更类型单一
```

此时输出：

```
此 commit 无需拆分。
  规模: XX 行
  模块: <name>
  类型: [数据结构/功能单元/...]
  建议: 可直接通过 /micro 移植
```

## 边界情况

| 场景 | 行为 |
|------|------|
| commit 不在目标仓库中 | 报错"未找到 commit <hash>"，检查仓库路径 |
| commit 为 merge commit | 提示"merge commit 不适合拆分，选择被合并的 commit" |
| commit 中全是自动生成文件 | 提示"自动生成代码，不适合手工拆分移植" |
| 触及 ≥ 3 个模块但总行数 < 200 | 仍触发多模块预扫描，向用户说明"规模小但模块多，可能是横切变更" |
| 用户在依赖询问时回复"不知道" | 等同于跳过，AI 自动推断并标注 |
| 自动推断的依赖关系不确定 | 在依赖概览中标注"推断"、"不确定"，提醒用户验证 |
| 实现+测试合并后 > 200 行 | 不分离测试。将功能拆为更小的子功能，每个 = 自己的实现 + 自己的测试 |
| 测试文件在另一目录（如 test/path 对应 lib/path） | 跨目录打包，归为同一个子 commit。在输出中标注跨目录关系 |
| 用户指定的仓库没有 git 历史 | 报错引导 |

## 与 `/micro` 的协作

`/split-commit` 的输出末尾的"建议实施顺序"直接以 `/micro` 命令形式给出。用户复制即用，无需手动翻译拆分方案为任务描述。

## 重要提醒

每次输出拆分方案后，追加以下提醒：

> ⚠ 此拆分方案由 AI 基于 diff 语义分析生成，可能存在误判。请逐项审查拆分边界是否合理、依赖关系是否正确。每个子 commit 的执行由 `/micro` 约束，你始终保留对"什么是一个有意义的原子任务"的最终解释权。
