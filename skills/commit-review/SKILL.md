---
name: commit-review
description: 此 Skill 用于提交前的全面审查。触发场景：用户准备提交代码、运行"提交前检查"、"/commit-review"、在 /micro 完成后被提醒等。此 Skill 读取 git 暂存区内容，按规模、测试覆盖、commit message 格式、核查表四个阶段输出结构化审查报告，并给出是否可提交的综合建议。只做软建议，不执行 git commit。
argument-hint: "[--message '<commit-message>']"
---

# 模拟提交审查

## 职责

对 `git diff --cached` 的内容进行一站式提交前审查。输出四阶段审查报告，让用户在不执行 `git commit` 的情况下提前发现问题。只建议，不拦截。

## 不做什么

- 不执行 `git commit`
- 不阻止用户提交（只给建议等级）
- 不自动运行测试
- 不在暂存区为空时强行审查

## 输入

- **默认**: 读取 `git diff --cached` 的完整内容
- **可选**: `--message "<commit-message>"` 用户已拟好的 message（用于 Phase 3 检查）

## 前置检查

审查开始前先执行：

```bash
git diff --cached --stat
```

- 若暂存区为空 → "暂存区无变更。请先 `git add` 要提交的文件。"
- 若有变更 → 进入 Phase 1

---

## Phase 1: 规模检查

### 收集数据

```bash
git diff --cached --numstat          # 净增删行数
git diff --cached --name-only         # 文件清单
git diff --cached --diff-filter=A --name-only  # 仅新增文件
```

### 输出

```
── Phase 1: 规模检查 ──────────────────────────────────────────

  净增行数: 145 行 (阈值 200)

  文件:
    src/MyPass.cpp       +85 -12  (代码)
    tests/features/mypass/basic.ll  +45 (测试)
    tests/features/mypass/nested.ll +38 (测试)

  测试/代码行比: 83 / 73 = 1.14  ✓ (> 0.5)

  检查项:
    □ 净增 ≤ 200            ✓
    □ 单一职责(仅一种变更类型)  ✓ (新功能)
    □ 无意外文件              ✓
    □ 有对应测试变更            ✓ (2 个测试文件)
```

### 判断逻辑

```
净增 ≤ 200         → ✓ 通过
净增 200-300       → ⚠ 偏大，建议考虑拆分
净增 > 300          → ✗ 建议拆分后再提交

文件类型检查:
  仅含 .cpp/.h/.ll 等源码         → 正常
  含调试文件、临时文件、.tmp       → ✗ 意外文件
  不含测试文件但含源码变更          → ⚠ 缺少测试
```

---

## Phase 2: 测试覆盖检查

### 收集数据

```bash
git diff --cached --name-only               # 变更文件列表
git diff --cached --unified=0                # 变更内容（定位新增函数）
```

### 分析

对比源码变更和测试变更：

1. 列出新增/修改的函数
2. 列出新增/修改的测试文件
3. 逐一匹配：每个新增函数是否有至少一个测试用例
4. 检查测试是否分配到了正确的目录（features/ vs regressions/）

### 输出

```
── Phase 2: 测试覆盖检查 ──────────────────────────────────────

  新增/修改函数:
    src/MyPass.cpp:
      + runOnFunction()    → 有测试: basic.ll RUN 1  ✓
      + matchPattern()     → 有测试: basic.ll RUN 2  ✓
      + rewriteIR()         → 有测试: nested.ll RUN 1 ✓

  测试文件分配:
    tests/features/mypass/  → 2 个文件 (契约性测试) ✓
    tests/regressions/     → 0 个文件 ⚠ (如果修复了已知 bug，建议补充回归测试)

  综合: 3/3 函数有测试覆盖 ✓
```

### 判断逻辑

```
所有新增函数有测试    → ✓
有函数缺失测试          → ✗ 严重，列出未覆盖函数
修复类变更无回归测试     → ⚠ 建议补充
纯重构无新增测试         → 正常（需在 commit message 中标注 [NFC]）
```

---

## Phase 3: Commit Message 检查

### 输入来源

1. 若用户传入 `--message "..."` → 检查它
2. 若暂存区有 `COMMIT_EDITMSG` → 检查它
3. 若都没有 → 不检查，在 Phase 5 生成建议 message

### 输出

若用户提供了 message：

```
── Phase 3: Commit Message 检查 ───────────────────────────────

  标题: "[MyPass] Add pattern matching for nested loops" (48 字符)

  检查项:
    □ 标题 ≤ 75 字符           ✓ (48)
    □ 包含组件标签               ✓ ([MyPass])
    □ 标题与正文间有空行          ✓
    □ 正文解释了 WHY            ✓
    □ 无 @ 提及                 ✓
    □ 无 "fix" "update" 等空泛词 ✗ (标题含 "Add" 可以接受)

  综合: 格式符合规范 ✓
```

若用户未提供 message：

```
── Phase 3: Commit Message 检查 ───────────────────────────────

  (未提供 message, 将在 Phase 5 生成建议)
```

### 检查项说明

| 检查项 | 违规示例 | 判定 |
|--------|---------|------|
| 标题长度 | 超过 75 字符 | ✗ |
| 组件标签 | 缺少 `[Component]` | ⚠ |
| 空行分隔 | 标题正文连在一起 | ✗ |
| WHY 而非 WHAT | "修改了 foo.cpp" | ✗ |
| @ 提及 | `@username` | ✗ 绝对禁止 |
| 空泛标题 | "fix bug" "update" | ⚠ |

---

## Phase 4: 核查表

逐项核对 `docs/Principle/COMMIT_PRINCIPLES.md` 中的 10 项检查。

### 输出

```
── Phase 4: 核查表 ────────────────────────────────────────────

  □  1. 单一关注点            ✓
  □  2. 有测试                 ✓ (2 文件, 3 条 RUN)
  □  3. 本地构建零警告         ? (请手动确认)
  □  4. 本地测试全绿           ? (请运行 /verify 手动验证)
  □  5. 独立环境验证           ? (请手动确认)
  □  6. Commit message 格式    ✓ (或待 Phase 5 生成)
  □  7. PR 规模合适 (≤400行)   ✓ (145 行)
  □  8. 自审 diff 完成         ? (请手动逐行审查)
  □  9. 大改动提前讨论          N/A (非重大变更)
  □ 10. CI 全绿                N/A (本地项目)

  已通过: 5 / 待人工确认: 4 / 不适用: 1
```

---

## Phase 5: 综合建议

汇总四个 Phase 的评估结果。

### 输出

```
── Phase 5: 综合建议 ──────────────────────────────────────────

  规模:     ✓ 145 行, 测试比 1.14
  覆盖:     ✓ 3/3 函数有测试
  格式:     — 未提供 message
  核查:     5/10 通过, 4 项需人工确认

  综合评估: ⚠ 建议确认后提交

  待人工确认:
    - 本地构建是否通过？
    - 本地测试是否通过？（可运行 /verify 生成验证命令）
    - 是否在独立环境中验证过？
    - 是否逐行审查过 diff？

  建议 commit message:
  ┌──────────────────────────────────────────────────────────┐
  │ [MyPass] Add pattern matching for nested loop structures │
  │                                                          │
  │ 识别嵌套循环中的归约模式，调用 ReductionPattern           │
  │ 进行替换。目前仅处理双层嵌套，三层以上标记 TODO           │
  │ 并原样返回。                                             │
  └──────────────────────────────────────────────────────────┘

  确认无误后的操作:
    1. 若未提供 message: 使用上述建议 message
    2. 手动提交: git commit -e
    3. 提交后: /verify 生成独立验证命令
```

### 评估等级

```
✓ 建议提交:     所有自动检查通过，人工确认项 ≤ 2
⚠ 建议修改后提交: 有 warning 但非阻塞
✗ 强烈建议不提交: 有严重问题（超大、无测试、message 空泛）
```

## 边界情况

| 场景 | 行为 |
|------|------|
| 暂存区为空 | "暂存区无变更。请先 `git add`。" |
| 暂存区仅含文档文件 (.md) | 跳过 Phase 2（测试覆盖），标注"文档变更，无需测试" |
| 暂存区含自动生成文件 | 标注"包含自动生成文件: X"，不计入净增行数 |
| revert commit | 宽限规模检查，标注"Revert commit" |
| 用户中断审查 | 不做任何操作 |
