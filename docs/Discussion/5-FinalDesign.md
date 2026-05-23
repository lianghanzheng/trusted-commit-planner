# 最终设计确认

## 对五个问题的确认

| # | 决策 |
|---|------|
| 1 | `/transplant` 不独立建，CLAUDE.md 路由引导组合 `/archeology` + `/split-commit` |
| 2 | `/commit-review` 基本符合预期，需增加**提交后验证命令生成**能力 |
| 3 | Stop Hook 仅提醒有人参与，选方案C |
| 4 | 环境配置硬编码进 CLAUDE.md |
| 5 | `/archeology` 规模阈值 500/2000 commits，确认 |

---

## 第2条的深化设计：`/verify` 命令

### 问题本质

你在 AI 仓库吃的亏有一个明确的结构：

```
AI 说: "测试通过了"
lit 实际行为: opt 进程静默退出 (exit 0)
你看到的: 什么都没看到
你的心理状态: "到底是通过了还是根本没跑？"
```

这是因为 lit 的默认行为是**静默通过**——只有失败时才输出。这与你需要"亲眼看见验证过程"的需求根本冲突。

### 解决方案

新增 `/verify` 命令。定位：**提交后的独立验证命令生成器**。

它不做抽象检查，不做覆盖率分析，只做一件事：从 lit 测试文件中提取 RUN 行，转化为你可以手动复制粘贴执行的命令，并告诉你每一条命令"通过时应该看到什么"。

### 功能边界

```
输入:
  - 当前 commit 的变更文件列表 (git diff --name-only HEAD~1)
  - 或者用户指定目录 (如 tests/features/my-pass/)

输出:
  - 每条测试的独立验证命令
  - 每条测试的"通过标志"说明
  - 一条"全部验证"的组合命令
  - 明确标注: "以下命令需要你手动执行，AI 不会替你运行"

不做什么:
  - 不自动运行测试
  - 不生成新测试
  - 不修改测试文件
```

### 输出规则

两条规则，缺一不可：

1. **终端输出**：逐条打印每个 RUN 行（含来源文件 + 通过标志说明），用户可逐条阅读、复制
2. **临时目录**：为每个 `.ll` 文件生成一个合并脚本（该文件的所有 RUN 行串成 `.sh`），按文件粒度独立执行

### 终端输出示例

```
# ============================================================
#  独立验证命令 - commit: abc123f "Add MyPass pattern matching"
#  生成时间: 2026-05-23 15:30
#  环境要求: LLVM 19.x built with assertions
#  ⚠ 以下命令需要你在终端中手动执行
# ============================================================

── tests/features/mypass/basic.ll ────────────────────────────
  RUN 1/2  通过标志: 输出末尾显示 '0 tests FAILED'

    opt -load-pass-plugin=build/MyPass.so -passes=my-pass \
      tests/features/mypass/basic.ll -S \
      | FileCheck tests/features/mypass/basic.ll

  RUN 2/2  通过标志: stdout 出现 'Transformed: 3 functions'

    opt -load-pass-plugin=build/MyPass.so -passes=my-pass \
      -enable-verbose tests/features/mypass/basic.ll -S \
      | grep "Transformed:"

── tests/features/mypass/nested.ll ───────────────────────────
  RUN 1/1  通过标志: FileCheck 输出 '0 tests FAILED'
  ⚠ 注意: 此测试依赖 basic.ll 所在的同一 Pass 插件

    opt -load-pass-plugin=build/MyPass.so -passes=my-pass \
      tests/features/mypass/nested.ll -S \
      | FileCheck tests/features/mypass/nested.ll

── tests/features/mypass/empty.ll ────────────────────────────
  RUN 1/1  通过标志: opt 正常退出 (exit 0), 无 segfault

    opt -load-pass-plugin=build/MyPass.so -passes=my-pass \
      tests/features/mypass/empty.ll -S \
      | FileCheck tests/features/mypass/empty.ll

# ============================================================
#  文件级合并脚本已生成到: /tmp/verify-abc123f/
#    basic.sh   nested.sh   empty.sh
#  执行方式: cd /tmp/verify-abc123f && ./basic.sh
#  或一键全部: for f in /tmp/verify-abc123f/*.sh; do "$f" || break; done
# ============================================================
```

### 临时目录中的合并脚本示例

`/tmp/verify-abc123f/basic.sh`:

```bash
#!/bin/bash
set -e
echo "=== basic.ll RUN 1/2 ==="
opt -load-pass-plugin=build/MyPass.so -passes=my-pass \
  tests/features/mypass/basic.ll -S \
  | FileCheck tests/features/mypass/basic.ll
echo "=== basic.ll RUN 2/2 ==="
opt -load-pass-plugin=build/MyPass.so -passes=my-pass \
  -enable-verbose tests/features/mypass/basic.ll -S \
  | grep "Transformed:"
echo "=== basic.ll: ALL RUNS PASSED ==="
```

每个 `.sh` 文件 `set -e`（遇错即停），文件内的 RUN 行按原始顺序排列，用户可单文件执行也可全部串行。

### 关键设计细节

**"通过标志"说明**：每条命令必须附带明确的、人类可读的通过条件。不能只说"运行这个命令"——必须说"运行时你应该看到 X，说明通过了"。

常见的通过标志：
```
FileCheck 匹配成功          → "输出末尾显示 '0 tests FAILED'"
opt 编译通过，无崩溃         → "opt 正常退出 (exit code 0), 无 segfault"
输出包含预期字符串           → "stdout 中出现 'Transformed: 3 functions'"
输出不包含某字符串           → "stdout 中不出现 'unreachable'"
已知失败 (XFAIL)             → "预期失败, 输出包含 'expected failure'"
```

**环境感知**：`/verify` 读取 CLAUDE.md 中的环境配置，自动填充正确的 `-load-pass-plugin=` 路径、`opt` 二进制路径等。

**与 FileCheck 的交互**：当检测到 lit 测试使用 `update_test_checks.py` 自动生成的检查行时，额外提示用户"此测试的检查行由工具自动生成，如果 Pass 输出格式有变化但语义正确，请运行 `update_test_checks.py` 更新检查行"。

### 触发方式

```
手动调用: /verify                    → 对最近一次 commit 生成验证命令
手动调用: /verify tests/features/foo → 对指定目录生成验证命令

CLAUDE.md 路由: /commit-review 完成后
              → 如果用户决定提交
              → 提醒: "提交后可以运行 /verify 生成独立验证命令"
```

### `/commit-review` 与 `/verify` 的关系

```
/commit-review     → 提交前的检查 (能不能提交？有什么问题？)
/verify            → 提交后的验证 (提交的代码真的有用吗？）

两者互补：
  /commit-review 回答: "这个提交符合规范吗？"
  /verify 回答:    "这个提交声称的功能，你能亲眼验证吗？"
```

---

## 最终 Artifact 清单 (v3)

```
┌──────────────────────────────────────────────────────────────────┐
│                       CLAUDE.md (宪法 + 路由器 + 环境)            │
│                                                                   │
│  静态: Commit规范, 测试分层, 人机分工, 双仓库, 文档策略           │
│  环境: LLVM_BUILD=/path/to/build, LLVM_VERSION=19, ...           │
│  路由: Skill间衔接规则 (状态→建议)                                │
│  维护: /config 命令                                               │
├──────────────────────────────────────────────────────────────────┤
│  Hook (1个)                                                       │
│  ┌─────────────────────────────────────────────────┐             │
│  │ Stop Hook: 检查未提交变更 → 提醒运行测试        │             │
│  └─────────────────────────────────────────────────┘             │
├──────────────────────────────────────────────────────────────────┤
│  Skill / Slash Command (11个)                                     │
│                                                                   │
│  开发流程           知识/学习          提交/验证                   │
│  ┌──────────┐      ┌──────────┐       ┌────────────────┐         │
│  │/pre-plan │      │/mentor   │       │/commit-review  │         │
│  │粒度审查   │      │编译器问答 │       │提交前一站式审查 │         │
│  └──────────┘      └──────────┘       └────────────────┘         │
│  ┌──────────┐      ┌──────────┐       ┌────────────────┐         │
│  │/micro    │      │/archeology│      │/verify    [新增]│         │
│  │原子任务   │      │仓库考古   │       │提交后独立验证   │         │
│  │≤150行    │      │+规模检测  │       │命令生成器       │         │
│  └──────────┘      └──────────┘       └────────────────┘         │
│  ┌──────────┐      ┌──────────────┐                               │
│  │/test-gen │      │/learn-from   │                               │
│  │测试生成   │      │学习工作簿     │                              │
│  │+抽象降级  │      │模板+实验+留白 │                              │
│  │+IR缩减    │      └──────────────┘                               │
│  └──────────┘                                                      │
│  ┌──────────────┐  ┌──────────┐  ┌──────────┐                    │
│  │/split-commit │  │/adr      │  │/config   │                    │
│  │Commit拆分    │  │设计决策   │  │描述性配置 │                    │
│  └──────────────┘  └──────────┘  └──────────┘                    │
│  ┌──────────────┐                                                  │
│  │/check-turn   │                                                  │
│  │转向信号检测   │                                                 │
│  │纯手动触发     │                                                 │
│  └──────────────┘                                                  │
├──────────────────────────────────────────────────────────────────┤
│  CLAUDE.md 工作流路由引导 (不独立建):                              │
│  /archeology + /split-commit → 移植流程                           │
│  /mentor + /learn-from → 学习流程                                 │
└──────────────────────────────────────────────────────────────────┘
```

### 统计

| 类别 | 数量 | 明细 |
|------|------|------|
| CLAUDE.md | 1 | + 路由规则 + 环境变量 |
| Hook | 1 | Stop (仅提醒, 不自动执行) |
| Skill/Slash Command | 11 | 新增 `/config`, `/verify`; 合并4→1 `/commit-review`; 合并2→1 `/test-gen` |

---

## 所有问题已确认

`/verify` 的粒度：**终端逐条打印每个 RUN + 临时目录每文件一个合并 `.sh`**。两者兼具，不冲突。


---
```
P0: CLAUDE.md (含环境变量)
P1: Stop Hook
P2: /commit-review, /verify, /micro, /pre-plan
P3: /test-gen, /archeology
P4: 其余 Skill
```
