---
name: verify
description: 此 Skill 用于从 lit 测试文件中提取独立验证命令。触发场景：用户提交代码后要求"生成验证命令"、"/verify"、在 /commit-review 完成后被提醒、需要亲眼确认测试是否通过等。此 Skill 读取 lit 测试的 RUN 行，替换变量为实际路径，终端逐条输出每个 RUN 行及其通过标志，同时在 /tmp 下为每个 .ll 文件生成独立的 set -e 可执行脚本。不自动运行测试。
argument-hint: "[tests/features/<pass-name> | <commit-hash>]"
---

# 提交后独立验证命令生成

## 职责

从 lit 测试文件中提取验证命令，转化为用户可以手动复制执行的形式。核心目标：让用户亲眼看到测试结果，而非信任 AI 的"测试通过了"。

## 不做什么

- 不自动运行测试（始终由用户手动执行）
- 不生成新测试
- 不修改测试文件
- 不替代 lit——`REQUIRES`/`UNSUPPORTED`/`XFAIL` 条件不会翻译为 shell 逻辑，仅标注提示

## 输入

- **默认**: `git diff --name-only HEAD~1` 中的 `.ll` 测试文件
- **可选**: 用户指定的目录路径（如 `tests/features/mypass/`）
- **可选**: commit hash（生成该 commit 的验证命令）

## 前置条件

执行前读取 CLAUDE.md 中的环境变量（`LLVM_BUILD`、`OPT`、`FILECHECK`、`LLVM_EXTRACT`、`LLVM_REDUCE`、`CLANG`、`PYTHON`）。若有未配置的关键变量，报错并引导用户运行 `/setup`。

---

## 工作流

### Step 1: 收集测试文件

```bash
# 若用户未指定目录，从最近一次 commit 获取
git diff --name-only HEAD~1 | grep '\.ll$'

# 若用户指定了目录
find <dir> -name '*.ll'
```

若未找到任何 `.ll` 文件 → "此 commit 无可验证的 lit 测试文件。"

### Step 2: 解析每个 `.ll` 文件的 RUN 行

对每个 `.ll` 文件，提取所有 `RUN:` 行：

```
RUN: opt ... | FileCheck ...
RUN: opt ... -enable-verbose ...
```

### Step 3: 变量替换

将 lit 变量替换为实际值：

| lit 变量 | 替换来源 | 替换为 |
|----------|---------|--------|
| `%s` | 文件系统 | 当前 `.ll` 文件的绝对路径 |
| `%S` | 文件系统 | 当前 `.ll` 文件所在目录 |
| `%t` | 生成 | `/tmp/verify-XXXXX/<test_name>.tmp` |
| `opt` | CLAUDE.md | `${OPT}` 或 `${LLVM_BUILD}/bin/opt` |
| `FileCheck` | CLAUDE.md | `${FILECHECK}` 或 `${LLVM_BUILD}/bin/FileCheck` |
| `llvm-extract` | CLAUDE.md | `${LLVM_EXTRACT}` |
| `llvm-reduce` | CLAUDE.md | `${LLVM_REDUCE}` |
| `clang` | CLAUDE.md | `${CLANG}` |
| `%python` | CLAUDE.md | `${PYTHON}` |
| `%%` | — | `%` |

> 替换后的命令中，所有工具使用绝对路径（如 `/opt/llvm-20/build/bin/opt`），确保用户在任何目录下都可执行。

### Step 4: 生成临时目录和脚本

创建 `/tmp/verify-<commit-hash前7位>/`：

```
/tmp/verify-abc123f/
├── basic.sh      # tests/features/mypass/basic.ll 的所有 RUN 行
├── nested.sh     # tests/features/mypass/nested.ll 的所有 RUN 行
├── empty.sh      # tests/features/mypass/empty.ll 的所有 RUN 行
└── all.sh        # 全部串行
```

每个 `.sh` 文件：

```bash
#!/bin/bash
set -e
# 来源: tests/features/mypass/basic.ll
# 通过标志: 所有 RUN 行的 FileCheck 均输出 '0 tests FAILED'
# 生成时间: 2026-05-23 15:30

echo "=== basic.ll RUN 1/2 ==="
/opt/llvm-20/build/bin/opt -load-pass-plugin=build/MyPass.so \
  -passes=my-pass tests/features/mypass/basic.ll -S \
  | /opt/llvm-20/build/bin/FileCheck tests/features/mypass/basic.ll

echo "=== basic.ll RUN 2/2 ==="
/opt/llvm-20/build/bin/opt -load-pass-plugin=build/MyPass.so \
  -passes=my-pass -enable-verbose tests/features/mypass/basic.ll -S \
  | grep "Transformed:"

echo "=== basic.ll: ALL RUNS PASSED ==="
```

`all.sh`：

```bash
#!/bin/bash
set -e
echo "========================================="
echo "  全部测试: commit abc123f"
echo "========================================="
for f in /tmp/verify-abc123f/basic.sh \
         /tmp/verify-abc123f/nested.sh \
         /tmp/verify-abc123f/empty.sh; do
  echo ""
  echo ">>> 执行: $(basename $f)"
  "$f" || { echo "FAILED: $f"; exit 1; }
done
echo ""
echo "=== ALL TESTS PASSED ==="
```

### Step 5: 终端输出

```
# =================================================================
#  独立验证命令 — commit: abc123f "[MyPass] Add pattern matching"
#  生成时间: 2026-05-23 15:30
#  环境: LLVM 19.x @ /opt/llvm-20/build
#  ⚠ 以下命令需要你在终端中手动执行。AI 不会替你运行。
# =================================================================

── tests/features/mypass/basic.ll ────────────────────────────
  RUN 1/2  通过标志: 输出末尾显示 '0 tests FAILED'

    /opt/llvm-20/build/bin/opt -load-pass-plugin=build/MyPass.so \
      -passes=my-pass tests/features/mypass/basic.ll -S \
      | /opt/llvm-20/build/bin/FileCheck tests/features/mypass/basic.ll

  RUN 2/2  通过标志: stdout 出现 'Transformed: 3 functions'

    /opt/llvm-20/build/bin/opt -load-pass-plugin=build/MyPass.so \
      -passes=my-pass -enable-verbose tests/features/mypass/basic.ll -S \
      | grep "Transformed:"

── tests/features/mypass/nested.ll ───────────────────────────
  RUN 1/1  通过标志: FileCheck 输出 '0 tests FAILED'
  ⚠ 此测试依赖 basic.ll 所在的同一 Pass 插件

    /opt/llvm-20/build/bin/opt -load-pass-plugin=build/MyPass.so \
      -passes=my-pass tests/features/mypass/nested.ll -S \
      | /opt/llvm-20/build/bin/FileCheck tests/features/mypass/nested.ll

── tests/features/mypass/empty.ll ────────────────────────────
  RUN 1/1  通过标志: opt 正常退出 (exit 0), 无 segfault

    /opt/llvm-20/build/bin/opt -load-pass-plugin=build/MyPass.so \
      -passes=my-pass tests/features/mypass/empty.ll -S \
      | /opt/llvm-20/build/bin/FileCheck tests/features/mypass/empty.ll

# =================================================================
#  文件级脚本已生成到: /tmp/verify-abc123f/
#    basic.sh   nested.sh   empty.sh   all.sh
#
#  执行单个文件:
#    cd /tmp/verify-abc123f && ./basic.sh
#
#  执行全部:
#    cd /tmp/verify-abc123f && ./all.sh
# =================================================================
```

---

## 通过标志说明

每条 RUN 命令必须附带明确的"通过标志"——告诉用户运行时应该看到什么才算通过。常见类型：

| 测试类型 | 通过标志 |
|----------|---------|
| FileCheck 匹配 | 输出末尾显示 `0 tests FAILED` |
| opt 编译通过 | opt 正常退出 (exit code 0), stdout 无 segfault/error |
| 输出包含预期字符串 | stdout 出现 `Transformed: N functions` |
| 输出不包含某字符串 | stdout 不出现 `unreachable` 或 `error:` |
| 已知失败 (XFAIL) | ⚠ 预期失败, 输出可能包含 `expected failure` 或非零 exit |
| 需要特定条件 (REQUIRES) | ⚠ 需要: `<条件>`, 不满足时此测试可能被 lit 跳过 |
| grep 检查 | grep 返回 0 (有匹配) |

---

## Lit 条件标注

对于包含 `REQUIRES:`、`UNSUPPORTED:`、`XFAIL:` 的 RUN 行，在命令后追加标注，但**不翻译为 shell 逻辑**：

```
⚠ 此 RUN 有 lit 条件: REQUIRES: asserts
   若当前环境不满足此条件, lit 会跳过此测试, 但此脚本会直接执行。
   如果只是想看当前环境下的行为, 可以忽略此标注。
```

---

## 边界情况

| 场景 | 行为 |
|------|------|
| commit 无 `.ll` 文件 | "此 commit 无可验证的 lit 测试文件。如果是纯代码变更，建议确保有对应的测试。" |
| `.ll` 文件无 RUN 行 | "X.ll 中无 RUN 行，跳过。" |
| `%t` 变量出现 | 创建临时目录 `/tmp/verify-XXXXX/`，在脚本中引用 |
| 环境变量未配置 | 报错并列出缺失变量，引导用户运行 `/setup` |
| CLAUDE.md 不存在 | "无法读取环境配置。请先运行 /setup 初始化 CLAUDE.md。" |
| 用户指定了不存在的目录 | 报错 "目录 X 不存在" |
| 测试文件数量 > 20 | 终端仅显示每个文件的第一条 RUN，完整命令在脚本中 |

## 与其他 Artifact 的关系

- 从 CLAUDE.md §一 读取 `LLVM_BUILD`、`OPT`、`FILECHECK` 等环境变量
- 被 `/commit-review` Phase 5 提醒调用
- `/test-gen` 生成的 `.ll` 测试文件同样可被本命令提取验证
