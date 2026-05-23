# Artifact 项目书规格说明 v1

基于 `docs/Discussion/5-FinalDesign.md` 最终设计，按工程规格对每个 Artifact 进行完整定义。

---

## 目录

1. [基础配置](#1-claudemd)
2. [自动化防线](#2-stop-hook)
3. [开发流程类](#3-7)
4. [提交/验证类](#8-9)
5. [知识/学习类](#10-13)
6. [基础设施类](#14-15)
7. [工作流路由规则](#16)

---

## 1. CLAUDE.md

### 标识

| 属性 | 值 |
|------|-----|
| ID | `CFG-001` |
| 名称 | CLAUDE.md |
| 形式 | `CLAUDE.md`（项目根目录） |
| 优先级 | P0 |
| 维护方式 | `/config` 命令 |

### 功能边界

**职责：**
- 定义项目的工程技术宪法：Commit 规范、测试分层策略、人机分工原则、双仓库架构与代码流向、文档策略
- 声明项目环境配置（硬编码）：LLVM 构建路径、Python 环境、工具链路径
- 作为 Skill 索引：列出所有可用命令及其用途，指导 AI 在何种上下文下调用哪个 Skill
- 作为工作流路由器：定义状态→建议的映射规则，在合适的时机建议用户调用下一个 Skill
- 声明禁止事项：不在手工库中直接运行无约束 AI、不自动提交代码

**不做什么：**
- 不执行操作、不替代 Skill 的领域知识
- 不记录具体的技术细节（交给 ADR）
- 不记录功能实现细节（交给测试用例和提交历史）

**输入：** 无（每次会话自动加载）

**输出：** 对 AI 行为的全局约束

**作用范围：** 当前项目所有 Claude Code 会话

### 内容结构规范

```markdown
# 项目名称

## 一、环境配置 (硬编码)
- LLVM_BUILD=/path/to/llvm/build
- OPT=${LLVM_BUILD}/bin/opt
- ...

## 二、工程技术规范
### 2.1 Commit 规范
### 2.2 测试分层策略
### 2.3 人机分工原则
### 2.4 双仓库架构
### 2.5 文档策略

## 三、可用命令索引
| 命令 | 用途 | 适用场景 |

## 四、工作流路由
| 当前状态 | 触发条件 | 建议操作 |

## 五、禁止事项
```

### 与其他 Artifact 的关系

- CLAUDE.md 是"索引"和"宪法"，不重复各 Skill 的详细指令
- 所有 Skill 通过 CLAUDE.md 被 AI 发现和路由
- `/config` 是该文件的唯一写入入口
- 环境配置被 `/verify`、`/test-gen` 等命令读取

---

## 2. Stop Hook

### 标识

| 属性 | 值 |
|------|-----|
| ID | `HOK-001` |
| 名称 | 任务结束测试提醒 |
| 形式 | `Stop` Hook（`settings.json`） |
| 优先级 | P1 |

### 功能边界

**职责：**
- 每次 Agent 任务结束后检查 `git status --short`
- 存在未提交变更时，输出提醒文本和建议的测试命令
- 不自动执行测试（由用户手动执行）

**不做什么：**
- 不自动运行测试
- 不启动 subagent
- 不替代 CI
- 不检查提交规范（交给 `/commit-review`）

**触发条件：** Claude Code `Stop` 事件

**输入：** `git status --short` 的结果

**输出：**
```
你有 3 个文件的未提交变更:
  - src/MyPass.cpp (modified)
  - tests/features/mypass/basic.ll (new)
  - tests/features/mypass/nested.ll (new)

建议运行测试:
  make check-my-pass

提交前可以运行: /commit-review
```

**作用范围：** 仅手工库（AI 库不配置此 Hook）

### 边界场景

| 场景 | 行为 |
|------|------|
| 无未提交变更 | 静默，不输出 |
| 仅 `.md` 文件变更 | 提醒，但标注"仅文档变更" |
| 处于 merge/rebase 冲突中 | 不提醒（避免干扰） |
| 用户在 AI 库中 | 不配置此 Hook |

---

## 3. `/pre-plan`

### 标识

| 属性 | 值 |
|------|-----|
| ID | `CMD-001` |
| 名称 | Plan前粒度审查 |
| 形式 | Slash Command（轻量 Skill） |
| 文件 | `.claude/skills/pre-plan/SKILL.md` |
| 优先级 | P1 |

### 功能边界

**职责：**
- 接收用户的需求描述，评估其粒度是否适合进入 Plan Mode
- 粒度过大时（预估 > 400 行或 ≥ 5 个独立步骤），阻止并输出分解建议
- 粒度合适时，放行并提供简短的 Plan Mode 入口指引

**不做什么：**
- 不替代 Plan Mode
- 不执行代码变更
- 不自行决定拆分方案——只给建议，用户决定

**输入：** 自然语言需求描述

**输出：**

情况 A——粒度合适：
```
✓ 粒度评估: 合适
  预估变更: ~150 行
  预估步骤: 2-3
  建议: 可以进入 Plan Mode
```

情况 B——粒度过大：
```
✗ 粒度评估: 过大
  预估变更: ~600 行
  原因: 此需求涉及 3 个独立关注点

  建议分解:
  1. [子需求A] 数据结构定义 (~120行)
  2. [子需求B] 核心模式匹配 (~200行)
  3. [子需求C] 边界处理与错误报告 (~150行)

  请选择其中一个子需求重新调用 /pre-plan
```

**适用库：** 仅手工库

### 评估标准

```
预估行数 = 新增函数体行数 + 新增测试行数 × 0.5
（不计算注释、空行、已有代码的位移）

过大判定：
  - 预估行数 > 400
  - 或独立步骤 ≥ 5
  - 或涉及 > 3 个文件的非测试代码
```

---

## 4. `/micro`

### 标识

| 属性 | 值 |
|------|-----|
| ID | `CMD-002` |
| 名称 | 微任务执行 |
| 形式 | Slash Command（轻量 Skill） |
| 文件 | `.claude/skills/micro/SKILL.md` |
| 优先级 | P1 |

### 功能边界

**职责：**
- 以最高行为约束等级执行单一原子任务（≤150 行净增代码）
- 完成后列出变更摘要，但不提交
- 生成建议的 commit message，提醒用户审查后手动提交

**不做什么：**
- 不自动提交
- 不拆分任务（拆分是 `/split-commit` 和 `/pre-plan` 的职责）
- 不跨文件重构（除非任务明确指定且总行数 ≤ 150）
- 不自行扩大任务范围

**输入：** 一个明确的、单一的任务描述

**输出：**
```
变更摘要:
  src/MyPass.cpp     +85 -12
  tests/features/mypass/basic.ll  +45 (新增)

建议 commit message:
  [MyPass] Add pattern matching for nested loop structures

  识别嵌套循环中的归约模式，调用 ReductionPattern 进行替换。
  目前仅处理双层嵌套，三层以上嵌套标记为 TODO 并原样返回。

⚠ 变更已完成，请审查后手动提交。建议运行 /commit-review 检查。
```

**适用库：** 仅手工库

### 行为约束 (与 CLAUDE.md 的关系)

```
CLAUDE.md 全局约束:
  - 每次任务聚焦单一目标
  - 每个 commit ≤ 200 行

/micro 最高警戒模式 (覆盖全局, 更严格):
  - 净增 ≤ 150 行
  - 禁止跨文件重构
  - 禁止自动提交
  - 完成后必须输出变更摘要和建议 commit message
```

### 边界场景

| 场景 | 行为 |
|------|------|
| 任务描述模糊 | 要求用户澄清后再执行 |
| 预估超 150 行 | 拒绝，建议调用 `/pre-plan` 分解 |
| 执行中发现需额外改动 | 暂停，报告发现，询问是否扩大范围或分步处理 |
| 改动涉及测试文件 | 测试文件不计入 150 行上限 |

---

## 5. `/commit-review`

### 标识

| 属性 | 值 |
|------|-----|
| ID | `CMD-003` |
| 名称 | 模拟提交审查 |
| 形式 | Slash Command（轻量 Skill） |
| 文件 | `.claude/skills/commit-review/SKILL.md` |
| 优先级 | P1 |

### 功能边界

**职责：**
- 对 `git diff --cached` 的内容进行一站式提交前审查
- 输出结构化审查报告：规模 → 测试覆盖 → 格式 → 核查表 → 综合建议
- 生成建议的 commit message（可被用户修改后使用）

**不做什么：**
- 不执行 `git commit`（用户始终手动提交）
- 不阻止用户提交（只给建议，不做硬拦截）
- 不运行测试（测试运行由用户手动或 `/verify` 的脚本完成）

**输入：** `git diff --cached`（暂存区内容）

**输出：** 五阶段审查报告

### 审查报告结构

**Phase 1: 规模检查**

```
□ 净增行数: 145 (阈值 200) — 通过
□ 文件清单:
    src/MyPass.cpp     +85 -12  (代码)
    tests/features/mypass/basic.ll  +45 (测试)
    tests/features/mypass/nested.ll +38 (测试)
□ 变更类型: 新功能 + 测试 — 单一职责 ✓
□ 意外文件: 无
□ 测试/代码行比: 83/73 = 1.14 — 良好 (>0.5)
```

**Phase 2: 测试覆盖检查**

```
□ src/MyPass.cpp 新增 3 个函数:
    runOnFunction()  — 有测试 (basic.ll RUN 1)
    matchPattern()   — 有测试 (basic.ll RUN 2)
    rewriteIR()      — 有测试 (nested.ll RUN 1)
□ 所有新增函数均有测试覆盖 ✓
□ 测试分配:
    features/:  2 文件 — 契约性测试 ✓
    regressions/: 0 — 如果修复了已知 bug，建议补充
```

**Phase 3: Commit Message 检查**

```
如果用户已提供 message (通过环境或参数):
□ 标题长度: 48 字符 (≤75) ✓
□ 组件标签: [MyPass] ✓
□ 标题正文空行: 有 ✓
□ 正文解释 WHY: 有 ✓
□ @ 提及: 无 ✓

如果用户未提供 message:
  生成建议的 commit message (见 Phase 5)
```

**Phase 4: 核查表 (10项)**

```
□ 1. 单一关注点        ✓
□ 2. 有测试            ✓ (2 文件, 3 条 RUN)
□ 3. 本地构建通过      ? (请手动确认)
□ 4. 本地测试通过      ? (请运行 /verify)
□ 5. 独立环境验证      ? (请手动确认)
□ 6. Commit message    ✓ (或已生成建议)
□ 7. PR规模合适        ✓ (145 行)
□ 8. 自审 diff 完成    ? (请手动逐行审查)
□ 9. 大改动需讨论      N/A
□ 10. CI全绿           N/A (本地项目)
```

**Phase 5: 综合建议**

```
综合评估: ⚠ 建议修改后提交

需要人工确认的项 (4项):
  - Phase 4 第3,4,5,8 项需要你手动确认

如果确认无误，建议的 commit message:
  ┌────────────────────────────────────────────┐
  │ [MyPass] Add pattern matching for nested   │
  │ loop structures                            │
  │                                            │
  │ 识别嵌套循环中的归约模式，调用              │
  │ ReductionPattern 进行替换。                │
  │ 目前仅处理双层嵌套，三层以上嵌套            │
  │ 标记为 TODO 并原样返回。                   │
  └────────────────────────────────────────────┘

  提交命令: git commit -e  (然后粘贴上述 message)
  提交后运行: /verify 生成独立验证命令
```

### 硬阻止 vs 软建议

`/commit-review` 只做软建议。如果用户需要硬阻止（如禁止 > 200 行提交），需额外配置 `.git/hooks/pre-commit`，逻辑从 `/commit-review` 的 Phase 1 移植。

---

## 6. `/verify`

### 标识

| 属性 | 值 |
|------|-----|
| ID | `CMD-004` |
| 名称 | 提交后独立验证命令生成 |
| 形式 | Slash Command（轻量 Skill） |
| 文件 | `.claude/skills/verify/SKILL.md` |
| 优先级 | P1 |

### 功能边界

**职责：**
- 读取最近一次 commit（或用户指定目录）的 lit 测试文件
- 从 RUN 行提取命令，替换 `%s`、`%t` 等 lit 变量，填入 CLAUDE.md 中的环境配置
- 终端输出：逐条打印每个 RUN 行（含来源文件 + 通过标志说明）
- 临时目录：为每个 `.ll` 文件生成一个 `set -e` 的独立 `.sh` 脚本
- 每条命令附带"通过标志"说明（什么输出算通过）

**不做什么：**
- 不自动运行测试
- 不生成新测试
- 不修改测试文件
- 不替代 lit（lit 的 `REQUIRES`/`UNSUPPORTED`/`XFAIL` 条件不会被翻译为 shell 逻辑）

**输入：**
- 默认：`git diff --name-only HEAD~1` 中的 `.ll` 测试文件
- 可选：用户指定的目录路径（如 `tests/features/mypass/`）

**输出：**

终端输出逐条 RUN：

```
# ============================================================
#  独立验证命令 - commit: abc123f "Add MyPass pattern matching"
#  生成时间: 2026-05-23 15:30
#  环境: LLVM 19.x @ /home/user/llvm-build
#  ⚠ 以下命令需要你在终端中手动执行
# ============================================================

── tests/features/mypass/basic.ll ─────────────────────────────
  RUN 1/2  通过标志: 输出末尾 '0 tests FAILED'

    /home/user/llvm-build/bin/opt \
      -load-pass-plugin=build/MyPass.so -passes=my-pass \
      tests/features/mypass/basic.ll -S \
      | /home/user/llvm-build/bin/FileCheck tests/features/mypass/basic.ll

  RUN 2/2  通过标志: stdout 出现 'Transformed: 3 functions'

    /home/user/llvm-build/bin/opt \
      -load-pass-plugin=build/MyPass.so -passes=my-pass \
      -enable-verbose tests/features/mypass/basic.ll -S \
      | grep "Transformed:"

... (每个文件的每条 RUN 行)

# ============================================================
#  文件级合并脚本已生成到: /tmp/verify-abc123f/
#    basic.sh   nested.sh   empty.sh
#  执行: cd /tmp/verify-abc123f && ./basic.sh
#  全部: for f in /tmp/verify-abc123f/*.sh; do "$f" || break; done
# ============================================================
```

临时目录中的 `.sh` 文件：

```bash
#!/bin/bash
set -e
# 来源: tests/features/mypass/basic.ll
# 通过标志: 所有 RUN 行的 FileCheck 均输出 '0 tests FAILED'

echo "=== basic.ll RUN 1/2 ==="
/home/user/llvm-build/bin/opt \
  -load-pass-plugin=build/MyPass.so -passes=my-pass \
  tests/features/mypass/basic.ll -S \
  | /home/user/llvm-build/bin/FileCheck tests/features/mypass/basic.ll

echo "=== basic.ll RUN 2/2 ==="
/home/user/llvm-build/bin/opt \
  -load-pass-plugin=build/MyPass.so -passes=my-pass \
  -enable-verbose tests/features/mypass/basic.ll -S \
  | grep "Transformed:"

echo "=== basic.ll: ALL RUNS PASSED ==="
```

### Lit 变量替换规则

| lit 变量 | 替换为 |
|----------|--------|
| `%s` | 当前 `.ll` 文件的绝对路径 |
| `%S` | 当前 `.ll` 文件所在目录 |
| `%t` | `/tmp/verify-XXXXX/test_name.tmp` |
| `%opt` | CLAUDE.md 中的 `OPT` 路径 |
| `%FileCheck` | CLAUDE.md 中的 `FILECHECK` 路径 |
| `%tool` (通用) | CLAUDE.md 中的对应路径 |

### 边界场景

| 场景 | 行为 |
|------|------|
| commit 无 `.ll` 文件 | 提示"此 commit 无可验证的 lit 测试" |
| lit 文件中有 `REQUIRES:` | 标注"此 RUN 需要条件: X"，但不自动翻译为 shell 逻辑 |
| lit 文件中有 `XFAIL:` | 标注"预期失败 (XFAIL): 可能不会显示错误" |
| `%t` 变量出现 | 创建 `/tmp/verify-XXXXX/` 临时目录并在脚本中引用 |
| 环境变量未配置 | 报错并列出缺失的变量，引导用户运行 `/config` |

### 与其他 Artifact 的关系

- 从 CLAUDE.md 读取环境配置（`LLVM_BUILD`、`OPT` 等）
- `/commit-review` Phase 5 提醒用户在提交后运行 `/verify`
- `/test-gen` 生成的测试文件被 `/verify` 读取

---

## 7. `/test-gen`

### 标识

| 属性 | 值 |
|------|-----|
| ID | `CMD-005` |
| 名称 | 测试生成与抽象降级检测 |
| 形式 | Skill |
| 文件 | `.claude/skills/test-gen/SKILL.md` |
| 优先级 | P2 |

### 功能边界

**职责：**

分为三个阶段，按需进入：

**Phase 1：四象限测试生成**
- 输入：Pass 源码 + 功能描述
- 生成正面/负面/边界/正交四类测试骨架（`.ll` + FileCheck 指令）

**Phase 2：抽象降级检测**
- 检查 Phase 1 生成的 IR 是否存在 8 项危险信号
- 无风险 → 完成
- 有风险 → 输出警告 + 进入 Phase 3

**Phase 3：IR 比对与缩减**
- 要求用户提供端到端测试的 C++ 代码
- 编译生成真实 IR
- 比对 Phase 1 的手写 IR 与 Phase 3 的真实 IR，输出差异分析
- 引导用户使用 `llvm-reduce` + `llvm-extract` 将真实 IR 精简为回归测试

**不做什么：**
- 不修改被测代码
- 不自动运行 `llvm-reduce`（工具辅助，最终判断由人完成）

**输入：**
- Pass 源码路径
- 功能描述（自然语言或设计文档引用）
- [可选] 手写 IR 测试草稿
- [Phase 3 时] 端到端 C++ 测试代码

**输出：**
- Phase 1：四象限 `.ll` 测试文件（放入 `tests/features/<pass-name>/`）
- Phase 2：抽象降级评估报告
- Phase 3：IR 差异分析 + 引导式缩减步骤 + 回归测试候选（放入 `tests/regressions/<pass-name>/`）

**适用库：** 手工库和 AI 库均可使用

### 抽象降级检测信号（8 项）

```
危险信号 (≥ 3 项触发警报):
  □ 1. IR中所有基本块都是单一前驱单一后继
  □ 2. 无任何 alloca/load/store (栈操作缺失)
  □ 3. 无任何 @llvm.* 内部调用
  □ 4. 函数调用全为直接调用, 无间接调用
  □ 5. 无任何 !dbg / !alias.scope 等 metadata
  □ 6. 类型系统过于简单 (无嵌套结构体, 无指针的指针)
  □ 7. 分支条件全为简单比较, 无复杂条件表达式
  □ 8. 函数签名无 noalias/readonly/dereferenceable 等属性

警报级别:
  ≥ 5 项: 高危 — 生成的测试几乎无法反映真实场景, 强烈建议进入 Phase 3
  3-4 项: 中危 — 测试可能遗漏重要边界, 建议进入 Phase 3
  ≤ 2 项: 低危 — 测试基本可靠, Phase 3 可选
```

### Phase 3 引导流程

```
Step 1: 用户提供端到端 C++ 代码
  → 编译生成真实 IR: clang++ -S -emit-llvm -O1 test.cpp -o real.ll

Step 2: 比对分析
  → 提取 Phase 1 测试覆盖的功能点在 real.ll 中的对应 IR 片段
  → 输出差异报告:
    "你的手写 IR 中函数调用是直接的, 但真实 IR 中出现了 3 层间接调用链。
     你的测试不会捕获间接调用的模式匹配失败。"

Step 3: llvm-extract 粗筛
  → llvm-extract --func=target_func real.ll -o extracted.ll

Step 4: 编写兴趣测试脚本
  → 创建 test.sh: opt -passes=your-pass extracted.ll -S | FileCheck ...

Step 5: llvm-reduce 自动缩减
  → llvm-reduce extracted.ll --test=test.sh -o reduced.ll

Step 6: 手工精修
  → 重命名变量、添加注释、确认断言准确
  → 产出放入 tests/regressions/<pass-name>/
```

---

## 8. `/split-commit`

### 标识

| 属性 | 值 |
|------|-----|
| ID | `CMD-006` |
| 名称 | Commit 历史分析与拆分 |
| 形式 | Skill |
| 文件 | `.claude/skills/split-commit/SKILL.md` |
| 优先级 | P2 |

### 功能边界

**职责：**
- 接收一个大 commit 的 hash，分析其 diff 内容
- 输出拆分方案：子 commit 列表、每个的文件清单、独立验证方式、依赖顺序

**不做什么：**
- 不自动执行拆分（决策始终由人确认）
- 不执行 `git rebase -i` 或 `git reset`
- 不适用于手工库的小 commit（手工库本身已经遵循 ≤200 行规范）

**输入：** commit hash（必须属于 AI 库或外部仓库）

**输出：**

```markdown
## 拆分方案: commit abc123f (总计 487 行, 8 文件)

### 建议拆分为 4 个独立 commit:

1. **[DataStructure] Add LoopNest and ReductionInfo types** (~85 行)
   文件: include/MyPass/LoopNest.h (+60), include/MyPass/ReductionInfo.h (+25)
   验证: 编译通过 (make MyPass)
   依赖: 无

2. **[MyPass] Implement pattern matching for single-level loops** (~120 行)
   文件: src/MyPass.cpp (+95), tests/features/mypass/single.ll (+25)
   验证: opt + FileCheck single.ll 通过
   依赖: commit 1

3. **[MyPass] Extend matching to nested loops** (~150 行)
   文件: src/MyPass.cpp (+110), tests/features/mypass/nested.ll (+40)
   验证: opt + FileCheck nested.ll 通过
   依赖: commit 2

4. **[MyPass] Add boundary handling and error reporting** (~132 行)
   文件: src/MyPass.cpp (+85), src/ErrorReport.cpp (+35), tests/features/mypass/errors.ll (+12)
   验证: opt + FileCheck errors.ll 通过, 检查错误输出格式
   依赖: commit 3

### 依赖图:
  [1] → [2] → [3] → [4]
```

---

## 9. `/adr`

### 标识

| 属性 | 值 |
|------|-----|
| ID | `CMD-007` |
| 名称 | 设计决策记录生成 |
| 形式 | Slash Command（轻量 Skill） |
| 文件 | `.claude/skills/adr/SKILL.md` |
| 优先级 | P2 |

### 功能边界

**职责：**
- 读取当前分支的变更内容，生成一份标准 ADR
- 存入 `docs/decisions/NNNN-title.md`
- ADR 编号自动递增

**不做什么：**
- 不做决策——只记录已完成的设计变更
- 不修改代码

**输入：** 当前分支的 diff 或最近 N 个 commit

**输出：** `docs/decisions/0004-use-indirect-call-analysis.md`

### ADR 模板

```markdown
# ADR-0004: Title

## Status
Accepted

## Context
为什么需要做这个决策？技术背景和约束是什么？

## Decision
做了什么决策？具体方案的核心逻辑是什么？

## Alternatives Considered
| 方案 | 优点 | 缺点 | 为何未选 |
|------|------|------|----------|

## Consequences
- 正面: ...
- 负面: ...
- 需要跟进: ...
```

---

## 10. `/archeology`

### 标识

| 属性 | 值 |
|------|-----|
| ID | `CMD-008` |
| 名称 | 仓库考古 |
| 形式 | Skill |
| 文件 | `.claude/skills/archeology/SKILL.md` |
| 优先级 | P3 |

### 功能边界

**职责：**
- 分析仓库的代码结构，输出模块拓扑图、入口函数、建议学习路径
- 分析 commit 历史，输出按主题分组的演进轨迹

**不做什么：**
- 不修改代码
- 不执行测试
- 仓库过大时拒绝全量分析

**输入：** 仓库路径 + 可选 `--branch` / `--since` / `--max-commits` / `--path`

**输出：**
1. 模块拓扑图（mermaid 或 ASCII）
2. 入口函数清单
3. 关键 commit 按主题分组
4. 建议阅读顺序

### 规模检测

```
检测逻辑 (在分析前强制执行):

1. git rev-list --count HEAD  → commit_count
2. find . -type f | wc -l      → file_count

决策:
  commit_count ≤ 500  → 全量分析
  500 < commit_count ≤ 2000 → 警告 + 建议缩小范围, 用户确认后继续
  commit_count > 2000 → 阻止, 必须指定 --max-commits 或 --since
  file_count > 10000 → 建议指定 --path 子目录
```

---

## 11. `/mentor`

### 标识

| 属性 | 值 |
|------|-----|
| ID | `CMD-009` |
| 名称 | 编译技术领域Mentor |
| 形式 | Skill |
| 文件 | `.claude/skills/mentor/SKILL.md` |
| 优先级 | P3 |

### 功能边界

**职责：**
- 回答 LLVM/MLIR/CUDA/SYCL/编译器设计相关的技术问题
- 提供 API 示例、文档链接、设计建议、调试思路

**不做什么：**
- 不写代码
- 不修改文件
- 不替代 Google/官方文档——而是帮助理解

**输入：** 自然语言问题或代码片段

**输出：** 解释 + 示例 + 文档引用 + 相关 Skill 推荐

### 知识范围声明

```
涵盖:
  ✓ LLVM IR/MLIR 的 API 与 Pass 开发
  ✓ CUDA/SYCL 编程模型与 GPU 优化
  ✓ 编译器设计模式 (Pass Pipeline, Visitor, Listener)
  ✓ 调试技巧 (Pass crash 分析, opt 工具链, lit 测试)
  ✓ TableGen / MLIR Dialect / Pattern Rewriting

不涵盖:
  ✗ 通用编程问题 (Python/Rust/Go 语法)
  ✗ 非编译器领域的算法设计
  ✗ Claude Code 使用问题 (给官方文档链接)

不确定时: 诚实说 "我不确定", 并建议查阅官方文档的具体章节
```

### 自动介入与路由

```
自动介入 (当用户消息匹配以下模式时主动提供帮助):
  - "为什么这段 IR ..."
  - "这个 Pass 应该..."
  - "opt 报错 ..."
  - "@llvm.*"
  - "llvm::" / "mlir::"

工作流路由 (讨论结束后):
  - 讨论 ≥ 3 轮且用户开始总结 → 建议调用 /learn-from 生成学习笔记
  - 讨论 API 设计问题 → 建议调用 /adr 记录
  - 讨论涉及具体代码 → 建议调用 /micro 实现
```

---

## 12. `/learn-from`

### 标识

| 属性 | 值 |
|------|-----|
| ID | `CMD-010` |
| 名称 | 学习工作簿生成 |
| 形式 | Skill |
| 文件 | `.claude/skills/learn-from/SKILL.md` |
| 优先级 | P2 |

### 功能边界

**职责：**
- 读取仓库 commit log，提取核心原理、关键设计决策、工程 trick
- 生成结构化学习工作簿：模板 + takeaway + 实验流程 + 留白
- **关键设计**：不留白的部分是脚手架，留白的部分才是学习发生的地方

**不做什么：**
- 不生成"完整笔记"替代人的理解和书写
- 不做没有实践路径的纯理论输出
- 不修改目标仓库

**输入：** 仓库路径 + commit 范围（默认 HEAD~200..HEAD）

**输出：** 学习工作簿（Markdown，存入 `docs/learning/<repo-name>/`）

### 工作簿模板

```markdown
# 学习工作簿: <仓库名>

## 项目架构概览 (AI生成)
[模块拓扑图, 入口函数清单]

---

## 知识点 1: <主题> (AI提取)

### 一句话 Takeaway
> 归约识别的核心不在于匹配循环结构，而在于追溯 use-def chain 上的内存依赖模式。

### 对应的代码位置
- 核心实现: `src/ReductionPattern.cpp:120-185`
- 相关数据结构: `include/LoopNest.h:45-78`
- 相关测试: `tests/features/reduction/`

### 实操实验
**目标**: 验证你对 use-def chain 追溯的理解

**前置条件**: LLVM 19.x built with assertions

**步骤**:
1. 在 `ReductionPattern.cpp:150` 处插入 `errs() << "Use: " << *U << "\n";`
2. 编译并运行: `make MyPass && opt ... tests/features/reduction/single.ll -S`
3. 观察输出: 你看到了哪些 Use？它们的顺序是什么？
4. 尝试修改 `tests/features/reduction/single.ll` 中的归约变量的使用方式（如增加一次额外赋值），再次运行。你看到了什么变化？

**预期观察**: Use list 的顺序可能与源码书写顺序不同。为什么？

### 我的理解 (*留白)
> [在此写下你在实验中的观察、疑问和思考]
> 
> 观察:
> 
> 疑问:
> 
> 与 takeaway 的印证/矛盾:

---

## 知识点 2: ...
```

### 实验设计原则

```
每个实验必须:
  1. 有具体的命令序列 (可复制粘贴执行)
  2. 有"修改代码 → 重新编译 → 运行观察"的闭环
  3. 有明确的观察目标 (不是"看看结果", 而是"检查 X 是否等于 Y")
  4. 有认知冲突的可能性 (结果不应完全可预测, 否则实验太简单)

不允许:
  ✗ "阅读 xxx.cpp 理解其实现" (这不是实验, 是阅读)
  ✗ "思考为什么这样设计" (没有实操步骤)
  ✗ 纯理论问题 (没有命令行操作)
```

---

## 13. `/check-turn`

### 标识

| 属性 | 值 |
|------|-----|
| ID | `CMD-011` |
| 名称 | 转向信号检测 |
| 形式 | Slash Command（轻量 Skill） |
| 文件 | `.claude/skills/check-turn/SKILL.md` |
| 优先级 | P3 |

### 功能边界

**职责：**
- 分析当前项目中的转向信号：同一 Pass 的补丁式修改次数、核心算法文件膨胀率、测试文件的适配性补丁
- 输出转向评估报告，供人决策

**不做什么：**
- 不做决策（是否转向始终由人决定）
- 不自动提醒（必须是手动触发 `/check-turn`）
- 不修改代码

**输入：** Pass 名称或文件路径（默认分析最近修改的 Pass）

**输出：**

```
转向评估报告: MyPass

核心算法文件: src/MyPass.cpp
  第一次实现 (commit a1b2c3): ~180 行
  当前: ~340 行
  膨胀率: +89%
  补丁式修改次数 (近10个commit): 6/10
    - "Add workaround for indirect calls" (commit d4e5f6)
    - "Handle edge case: multi-level pointer" (commit g7h8i9)
    - ...

测试文件适配性补丁:
  tests/features/mypass/basic.ll 从 25 行膨胀到 78 行 (+212%)
  新增的 53 行中有 40 行是为适配真实 IR 格式而添加的分支条件

转向信号强度: ⚠ 中高 (2/3 信号触发)
  ✓ 核心算法膨胀 > 50%
  ✓ 测试文件为适配真实IR大幅膨胀
  ✗ 还未达到三次实现 (当前是第二次)

建议: 可以考虑规划第三次实现，将前两次的经验整合进干净设计。
       在转向前确保: 所有现有测试 (2 feature + 1 regression) 都能在新实现下通过。
```

### 信号定义

```
信号 1: 补丁式修改密度
  过去 N 个 commit 中，描述含 "fix"/"workaround"/"handle edge case" 的比例 > 50%

信号 2: 核心文件膨胀率
  核心算法文件行数 / 第一次实现行数 > 1.5

信号 3: 测试适配性补丁
  测试文件中分支条件/特殊处理行数 / 测试总行数 > 0.3

信号 4: 三次实现
  已经是当前方案的第三次从头实现 (通过 commit log 中的重构标记判断)

触发转向建议: 信号 1+2 或 1+3 或 信号4 单独触发
```

---

## 14. `/config`

### 标识

| 属性 | 值 |
|------|-----|
| ID | `CMD-012` |
| 名称 | 描述性配置 CLAUDE.md |
| 形式 | Slash Command（轻量 Skill） |
| 文件 | `.claude/skills/config/SKILL.md` |
| 优先级 | P2 |

### 功能边界

**职责：**
- 接收用户对 CLAUDE.md 的自然语言修改需求
- 读取当前 CLAUDE.md → 理解需求 → 生成修改 → 用户确认 → 写入
- 支持的操作：添加/修改环境变量、调整规范参数、添加/删除 Skill 索引条目、修改路由规则

**不做什么：**
- 不让用户手动编辑 CLAUDE.md
- 不修改 Skill 的 SKILL.md 文件（那是 Skill 开发者的职责）
- 不修改 `settings.json`

**输入：** 自然语言配置需求

**输出：** 修改预览 → 用户确认 → 应用 → 输出 diff

### 使用示例

```
用户: /config 把 LLVM 构建路径改为 /opt/llvm-20/build

AI:
  当前: LLVM_BUILD=/home/user/llvm-build
  修改为: LLVM_BUILD=/opt/llvm-20/build
  影响范围: /verify 和 /test-gen 的命令生成将使用新路径
  确认修改? (y/n)

用户: y

AI: 已更新 CLAUDE.md 第 5 行。
```

### 边界约束

```
允许修改:
  ✓ 环境变量 (LLVM_BUILD, OPT, FILECHECK, ...)
  ✓ 规范参数 (行数阈值, 规模检测阈值)
  ✓ 可用命令索引
  ✓ 工作流路由规则

不允许修改:
  ✗ 工程技术规范的核心原则 (Commit/测试/文档策略)
  ✗ 禁止事项清单 (只能在特殊情况下追加, 不能删除)
  ✗ 其他 Skill 的 SKILL.md
```

---

## 15. `/check-turn` (续)

已在第 13 节定义。

---

## 16. 工作流路由规则

### 标识

| 属性 | 值 |
|------|-----|
| ID | `RTG-001` |
| 名称 | 工作流路由规则 |
| 形式 | CLAUDE.md 内置节 |
| 文件 | `CLAUDE.md` 第四节 |
| 优先级 | P1（随 CLAUDE.md 实施） |

### 路由规则表

```markdown
## 四、工作流路由

| 当前状态 | 触发条件 | 建议操作 | 理由 |
|----------|----------|----------|------|
| 与 /mentor 讨论技术问题 ≥ 3 轮 | 用户开始总结或表达"明白了" | 建议 `/learn-from` | 将讨论成果固化为学习记录 |
| 在 Plan Mode 完成规划 | 用户批准计划 | 建议 `/micro` 执行第一个原子任务 | 防止批准后又回到大Commit模式 |
| /micro 完成代码变更 | 变更已生成，待审查 | 建议 `/commit-review` | 引导完成提交前检查 |
| /commit-review 审查通过 | 用户决定提交 | 建议 `/verify` 生成验证命令 | 确保提交后能独立验证 |
| 从 AI 库提取了代码 | 用户提到"移植"或"整理" | 建议先用 `/archeology` 理解结构，再用 `/split-commit` 分析拆分 | 组合工具完成移植流程 |
| /test-gen Phase 2 触发警报 | 检测到 ≥ 3 项抽象降级信号 | 引导进入 Phase 3 (IR比对与缩减) | 闭合测试质量循环 |
| 用户手动 commit 后 | `git commit` 完成 | 提醒："/verify 可为此次提交生成独立验证命令" | 不遗漏验证环节 |
| 对话回合结束 | Stop Hook 触发 | 提醒有未提交变更时运行测试 | 独立于 Skill 体系的自动防线 |
```

### 路由规则的维护

```
添加新规则: /config → "添加路由规则: 当 X 时建议 Y"
删除规则: /config → "删除路由规则第 N 条"
修改规则: /config → "修改路由规则第 N 条的触发条件为..."

所有路由规则变更需输出 diff 供用户确认。
```

---

## 附录A：Artifact 清单总表

| ID | 名称 | 形式 | 文件位置 | 优先级 |
|----|------|------|----------|--------|
| CFG-001 | CLAUDE.md | CLAUDE.md | `./CLAUDE.md` | P0 |
| HOK-001 | Stop Hook | Hook | `.claude/settings.local.json` | P1 |
| CMD-001 | /pre-plan | Slash Command | `.claude/skills/pre-plan/SKILL.md` | P1 |
| CMD-002 | /micro | Slash Command | `.claude/skills/micro/SKILL.md` | P1 |
| CMD-003 | /commit-review | Slash Command | `.claude/skills/commit-review/SKILL.md` | P1 |
| CMD-004 | /verify | Slash Command | `.claude/skills/verify/SKILL.md` | P1 |
| CMD-005 | /test-gen | Skill | `.claude/skills/test-gen/SKILL.md` | P2 |
| CMD-006 | /split-commit | Skill | `.claude/skills/split-commit/SKILL.md` | P2 |
| CMD-007 | /adr | Slash Command | `.claude/skills/adr/SKILL.md` | P2 |
| CMD-008 | /archeology | Skill | `.claude/skills/archeology/SKILL.md` | P3 |
| CMD-009 | /mentor | Skill | `.claude/skills/mentor/SKILL.md` | P3 |
| CMD-010 | /learn-from | Skill | `.claude/skills/learn-from/SKILL.md` | P2 |
| CMD-011 | /check-turn | Slash Command | `.claude/skills/check-turn/SKILL.md` | P3 |
| CMD-012 | /config | Slash Command | `.claude/skills/config/SKILL.md` | P2 |
| RTG-001 | 工作流路由 | CLAUDE.md 节 | `./CLAUDE.md` §4 | P1 |

## 附录B：实施顺序

```
Phase 1 (基础): CFG-001 → CMD-012 (/config)
  产出: 可工作的 CLAUDE.md + 描述性配置能力

Phase 2 (防线): HOK-001 → CMD-003 (/commit-review) → CMD-004 (/verify)
  产出: 提交前审查 + 提交后验证的完整闭环

Phase 3 (开发): CMD-001 (/pre-plan) → CMD-002 (/micro)
  产出: 任务粒度控制的事前+事后机制

Phase 4 (学习): CMD-005 (/test-gen) → CMD-006 (/split-commit) → CMD-008 (/archeology)
  产出: 测试生成 + 代码分析的工具体系

Phase 5 (知识): CMD-010 (/learn-from) → CMD-007 (/adr) → CMD-009 (/mentor)
  产出: 学习与决策记录系统

Phase 6 (反思): CMD-011 (/check-turn)
  产出: 转向信号检测
```
