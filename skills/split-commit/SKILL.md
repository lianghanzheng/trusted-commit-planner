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

### Step 1: 获取 diff

```bash
git -C <repo> show <commit-hash> --stat      # 先看文件清单和行数
git -C <repo> show <commit-hash> --no-stat    # 再看完整 diff
```

### Step 2: 理解变更语义

逐文件阅读 diff，识别以下信号：

| 信号 | 含义 | 拆分优先级 |
|------|------|-----------|
| 新增头文件/类型定义 | 数据结构定义 | 第1个子commit |
| 新增测试文件 (`.ll`, `.cpp`) | 测试 | 建议紧跟被测代码，或作为独立commit |
| 新增 Pass/功能函数 | 核心逻辑 | 中间子commit |
| 修改已有函数 | 重构/适配 | 可能与新增分离 |
| CMakeLists.txt / BUILD 变更 | 构建配置 | 与对应的新增文件放在一起 |
| 仅格式/注释变更 | NFC | 独立 commit，标记 `[NFC]` |
| 错误处理/边界处理 | 辅助逻辑 | 最后的子commit |

### Step 3: 构建拆分方案

按以下原则组织子 commit：

1. **每个子 commit 可独立编译**——如果 commit A 新增了一个被 commit B 引用的数据结构，A 必须在 B 之前
2. **测试跟着被测代码走**——如果新增函数 `foo()` 的测试在 `test_foo.ll`，它们应在同一个子 commit 中（或紧邻）
3. **基础设施先于功能**——数据结构、构建配置、工具函数 → 核心算法 → 边界处理 → 文档
4. **每个子 commit ≤ 200 行**
5. **每个子 commit 有明确的独立验证方式**

### Step 4: 输出拆分方案

格式固定如下：

```markdown
## 拆分方案: commit <hash> "<title>"

来源: <repo-path>
原始规模: <total> 行, <N> 文件

### 建议拆分为 <M> 个独立 commit:

---

### Commit 1/3: `[Component] <标题>` (~XX 行)
**文件**:
  - path/to/file1.cpp (+XX)
  - path/to/file2.h (+XX)
**内容**: <一句话描述>
**独立验证**: <如何验证这个 commit 正确——编译、指定测试、检查输出>
**依赖**: 无

---

### Commit 2/3: `[Component] <标题>` (~XX 行)
**文件**:
  - path/to/file3.cpp (+XX)
**内容**: <一句话描述>
**独立验证**: opt -passes=... tests/... | FileCheck tests/...
**依赖**: Commit 1 (依赖其中的 <具体数据结构/函数>)

---

### Commit 3/3: `[Component] <标题>` (~XX 行)
**文件**:
  - path/to/file4.cpp (+XX)
**内容**: <一句话描述>
**独立验证**: <命令>
**依赖**: Commit 2

---

### 依赖图:
  [1] → [2] → [3]

### 建议实施顺序:
  1. /micro "Commit 1/3 的内容描述"
  2. /micro "Commit 2/3 的内容描述"
  3. /micro "Commit 3/3 的内容描述"
```

### Step 5: 若 commit 无需拆分

若 commit 满足以下所有条件，不输出拆分方案：

```
原始行数 ≤ 200 且 变更类型单一（仅一种信号）
```

此时输出：

```
此 commit 无需拆分。
  规模: XX 行
  类型: [数据结构/核心逻辑/测试/...]
  建议: 可直接通过 /micro 移植
```

## 边界情况

| 场景 | 行为 |
|------|------|
| commit 不在目标仓库中 | 报错"未找到 commit <hash>"，建议检查仓库路径 |
| commit 为 merge commit | 提示"merge commit 不适合拆分，请选择被合并的 commit" |
| commit 中全是自动生成文件 | 提示"此 commit 为自动生成代码，不适合手工拆分移植" |
| commit 规模极大（>1000 行） | 先输出变更概览（按目录/模块分组），询问用户"你想先聚焦哪个模块？"，再对焦点模块详细拆分 |
| 用户指定的仓库没有 git 历史 | 报错引导 |

## 与 `/micro` 的协作

`/split-commit` 的输出末尾的"建议实施顺序"直接以 `/micro` 命令形式给出。用户复制即用，无需手动翻译拆分方案为任务描述。

## 重要提醒

每次输出拆分方案后，追加以下提醒：

> ⚠ 此拆分方案由 AI 基于 diff 语义分析生成，可能存在误判。请逐项审查拆分边界是否合理、依赖关系是否正确。每个子 commit 的执行由 `/micro` 约束，你始终保留对"什么是一个有意义的原子任务"的最终解释权。
