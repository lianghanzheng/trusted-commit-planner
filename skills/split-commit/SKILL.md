---
name: split-commit
description: 此 Skill 用于分析大型 commit 的模块级结构。触发场景：用户提供 commit hash 并要求"拆分"、"分解"、"分析这个 commit"、"/split-commit"等。此 Skill 只做模块级骨架分析——识别涉及的模块、推断依赖关系、基于 `#include` 聚类耦合文件组，输出文件组清单和 /decompose 调用建议。不加载 COMMIT_PRINCIPLES.md，不做细粒度拆分，不做测试匹配。后续由 /decompose 处理文件组内部分解，由 /micro 处理原则对齐和独立验证。
argument-hint: "<commit-hash> [--repo <path>]"
---

# Commit 拆分分析

## 职责

分析一个 commit 的**模块级结构**——识别涉及的模块、推断依赖关系、基于 `#include` 聚类耦合文件组，输出模块骨架和 `/decompose` 调用建议。**不加载原则，不做细粒度拆分，不做测试匹配。** 职责在文件组边界停止，后续由 `/decompose` 和 `/micro` 接手。

## 不做什么

- 不执行任何 git 写操作
- 不输出 ≤200 行的子 commit——细粒度拆分交给 `/decompose`
- 不做测试匹配（测试是 `/micro` 的职责）
- 不适用于本身已 ≤200 行且单模块的 commit（直接告知"无需拆分"）

## 输入

- **必需**: commit hash（AI 库或外部仓库）
- **可选**: `--repo <path>` 指定仓库路径（默认从对话上下文推断）

## 工作流

### Step 1: 路径级扫描

```bash
git -C <repo> show <commit-hash> --name-only    # 仅文件路径，不取 diff 内容
```

统计总文件数和净增行数（`git show <hash> --stat` 的最后一行）。

### Step 2: 模块预扫描

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

  < 3 个模块 → 单模块模式 → 直接进入 Step 4
  ≥ 3 个模块 → 多模块模式 → 进入 Step 3
```

**注意**：即使 commit < 200 行，如果触及 ≥ 3 个模块也应触发多模块预扫描（可能是横切变更，需确认依赖关系后才能正确拆分）。

### Step 3: 多模块询问

仅基于 Step 1 的 `--name-only` 输出（纯路径，**不读取任何文件内容**）。列出涉及的模块名和文件数，询问用户依赖关系：

```
此 commit 涉及 4 个模块:

  llvm/lib/Analysis          (2 文件)
  llvm/lib/Transforms/Scalar (3 文件)
  llvm/lib/Transforms/Utils  (1 文件)
  llvm/test/Transforms/Scalar (5 文件)

请描述这些模块之间的依赖关系 (自由文本)，例如:

  Scalar 依赖 Analysis
  Scalar 依赖 Utils
  test 与 Scalar 绑定

如果跳过此步，将在 Step 4 通过逐层披露机制自动推断。
```

**用户提供依赖后**：记录依赖描述，进入 Step 4。

**用户跳过时**：不在本步做任何推断。标注"依赖关系将由 Step 4 自动推断"，进入 Step 4。

### Step 4: 依赖推断 + 文件聚类

**先定披露规则，再逐层读取，最后聚类。**

#### 4a. 内容逐层披露

所有文件内容读取遵循三级递进。`#include` 行作为 Layer 1 先锋，函数签名为 Layer 2 补充，完整内容为 Layer 3 底线——绝不越过用户直接进入下一层。

```
Layer 1: #include 分析 (先锋，极低 token)

  git -C <repo> show <commit-hash> -p | grep "^+#include"

  提取: 所有新增 #include 行
  可完成: 初步模块归属、直接依赖关系
  局限: 无法检测接口级隐式依赖

Layer 2: 函数声明查阅

  对 Layer 1 聚类不确定的文件，只读前 80 行:
    git -C <repo> show <commit-hash> -- <file> | head -80

  提取: 类/函数签名、模板参数、公开方法声明
  目的: 确认文件间的接口级依赖关系

Layer 3: 完整内容申请（需用户确认）

  若 Layer 2 仍无法判断依赖关系，列出需要全量读取的文件清单，
  并说明每个文件的读取理由:

    "以下文件在 Layer 1-2 后仍无法确定其依赖归属:

       - compiler/lib/Foo/FooImpl.cpp (420 行)
         原因: 前 80 行仅有 include 和构造函数，核心逻辑在后半部分
         读取目的: 确认是否依赖 Bar 模块的接口

     是否允许读取上述文件的完整内容？(回复 y 确认)"

  用户未确认前，绝不读取任何文件的完整内容。
  标注相应文件组为"耦合度不确定"，建议用户手动审查。
```

#### 4b. 模块间依赖推断

基于 Layer 1 的 `#include` 结果（以及 Layer 2 的函数声明，若已触发）。若用户在 Step 3 跳过了依赖描述，在此自动推断：

```
- Analysis 中的头文件无新增 #include → 无外部依赖
- Transforms/Scalar 中的文件 #include "Analysis/..." → Scalar 依赖 Analysis
- Transforms/Scalar 中的文件 #include "Transforms/Utils/..." → Scalar 依赖 Utils
```

推断结果标注"自动推断"供用户验证。若用户在 Step 3 已提供依赖描述，跳过本步。

#### 4c. 文件耦合聚类

综合 Layer 1-2 的披露结果，按以下规则聚类：

1. 属于同一模块、且通过 `#include` 相互引用的 `.h` + `.cpp` → 一个文件组
2. 同一 `.cpp` 文件对应的 `CMakeLists.txt` 变更 → 归入该文件组
3. 无 include 关联但同属一个模块的文件 → 各自独立为文件组
4. 纯数据结构头文件（无 `.cpp`）→ 独立文件组，排在所有依赖它的文件组之前
5. 触发 Layer 3 但用户未授权的文件 → 暂标注"耦合度不确定"，建议人工审查后手动分组

### Step 5: 输出模块骨架

格式固定如下——模块依赖概览在前，文件组清单在后，不再深入到子 commit 粒度。

```markdown
## 模块骨架分析: commit <hash> "<title>"

来源: <repo-path>
原始规模: <total> 行, <N> 文件
涉及模块: <M>

---

### 模块依赖概览

  llvm/lib/Analysis → (无依赖，基础设施)
  llvm/lib/Transforms/Scalar → 依赖 Analysis
  llvm/test/Transforms/Scalar → 与 Scalar 功能绑定

  (此依赖关系由<用户提供/AI 推断>)

---

### 文件组清单 (按实现顺序)

---

#### 文件组 1/4: `[Analysis] LoopSafetyInfo 接口` (~45 行)
**所属模块**: llvm/lib/Analysis
**文件**:
  - llvm/include/llvm/Analysis/LoopSafetyInfo.h (+45)
  - llvm/lib/Analysis/CMakeLists.txt (+2)
**耦合度**: 高 — 头文件定义纯接口，CMakeLists 声明新 target
**行数估算**: ~45
**建议**: /decompose llvm/include/llvm/Analysis/LoopSafetyInfo.h llvm/lib/Analysis/CMakeLists.txt
  (预期输出 1 Phase，可直接交 /micro)

---

#### 文件组 2/4: `[LoopUnswitch] 核心实现` (~150 行)
**所属模块**: llvm/lib/Transforms/Scalar
**文件**:
  - llvm/lib/Transforms/Scalar/LoopUnswitch.cpp (+150)
**耦合度**: 高 — 单文件，依赖文件组 1 的接口
**行数估算**: ~150
**建议**: /decompose llvm/lib/Transforms/Scalar/LoopUnswitch.cpp
  (预期输出 2-3 Phase — 此文件含 ~150 行实现，需细粒度分解)

---

#### 文件组 3/4: `[LoopUnswitch] 最简测试` (~30 行)
**所属模块**: llvm/test/Transforms/Scalar
**文件**:
  - llvm/test/Transforms/LoopUnswitch/trivial.ll (+30)
**耦合度**: 测试 — 验证文件组 2 的核心功能
**行数估算**: ~30
**建议**: 直接交 /micro (无需 /decompose)

---

#### 文件组 4/4: `[LoopUnswitch] 边界测试` (~25 行)
**所属模块**: llvm/test/Transforms/Scalar
**文件**:
  - llvm/test/Transforms/LoopUnswitch/edge-cases.ll (+25)
**耦合度**: 测试 — 验证文件组 2 的边界处理
**行数估算**: ~25
**建议**: 直接交 /micro (无需 /decompose)

---

### 建议执行顺序

  1. /decompose llvm/include/llvm/Analysis/LoopSafetyInfo.h llvm/lib/Analysis/CMakeLists.txt
     → 预期输出 1 Phase，交 /micro

  2. /decompose llvm/lib/Transforms/Scalar/LoopUnswitch.cpp
     → 预期输出 ~3 Phase，逐 Phase 交 /micro
     → 每个 Phase 的测试由 /micro 自动匹配 (对应 llvm/test/Transforms/LoopUnswitch/)

  3. /micro "添加 trivial unswitch 最简测试"
  4. /micro "添加 edge cases 边界测试"
```

### Step 6: 若 commit 无需拆分

若 commit 满足以下所有条件，不输出模块骨架：

```
净增行数 ≤ 200 且 模块数 < 3 且 文件组数 ≤ 1
```

此时输出：

```
此 commit 无需拆分。
  规模: XX 行
  模块: <name>
  建议: 可直接通过 /micro 执行
```

## 边界情况

| 场景 | 行为 |
|------|------|
| commit 不在目标仓库中 | 报错"未找到 commit <hash>"，检查仓库路径 |
| commit 为 merge commit | 提示"merge commit 不适合拆分，选择被合并的 commit" |
| commit 中全是自动生成文件 | 提示"自动生成代码，不适合手工拆分移植" |
| 触及 ≥ 3 个模块但总行数 < 200 | 仍触发多模块预扫描，向用户说明"规模小但模块多，可能是横切变更" |
| 用户在依赖询问时回复"不知道" | 等同于跳过，Step 4 自动推断并标注 |
| 自动推断的依赖关系不确定 | 在依赖概览中标注"推断"、"不确定"，提醒用户验证 |
| 新增 include 为空（无头文件变更） | 跳过聚类，按目录结构分组 |
| 用户指定的仓库没有 git 历史 | 报错引导 |

## 与下游 Skill 的协作

```
/split-commit → 模块骨架 + 文件组清单 + /decompose 调用建议
/decompose    → 文件组内细粒度分解 (Phase 计划)
/micro        → 原子执行 + 原则对齐 + 测试匹配 + 独立验证
```

`/split-commit` 是三层漏斗的入口——它不替代 `/decompose` 和 `/micro`，只为它们提供正确的输入。

## 重要提醒

每次输出模块骨架后，追加以下提醒：

> 此骨架由 AI 基于路径扫描和 include 分析生成。模块依赖关系和文件耦合度的判断可能存在遗漏，请人工验证后再进入 /decompose 阶段。
