---
name: setup
description: 此 Skill 用于维护项目的 CLAUDE.md 配置文件。触发场景：用户要求"配置环境"、"设置 LLVM 路径"、"修改 CLAUDE.md"、"添加路由规则"、"更新配置"、"setup"等。此 Skill 读取 CLAUDE.md（或 CLAUDE.md.template），理解用户的自然语言配置意图，生成修改预览，经用户确认后写入。
argument-hint: "[变量名=值 | 配置描述]"
---

# 描述性配置 CLI

## 职责

允许用户以自然语言描述配置需求，而非手动编辑 CLAUDE.md。核心原则：**读模板 → 理解意图 → 生成修改 → 用户确认 → 写入。**

## 不做什么

- 不手动修改 CLAUDE.md（必须通过本命令）
- 不修改 `settings.json`（那是 Hook 配置的领域）
- 不修改其他 Skill 的 SKILL.md
- 不删除工程技术规范的核心原则
- 不删除禁止事项清单（仅可追加）

## 工作流

### Step 1: 确定目标文件

```
若 ./CLAUDE.md 存在 → 以此为当前配置，修改它
若 ./CLAUDE.md 不存在 → 复制 ./CLAUDE.md.template 到 ./CLAUDE.md，然后修改
```

### Step 2: 理解用户意图

解析用户输入，识别操作类型：

| 用户输入示例 | 操作类型 | 说明 |
|-------------|---------|------|
| `LLVM_BUILD=/opt/llvm/build` | 设置环境变量 | 更新 §一 中的对应 `{{VAR}}` |
| `把 llvm 路径改成 /opt/llvm-20/build` | 设置环境变量 | 同上，需识别指代 |
| `添加路由规则：当 /micro 完成后建议 /verify` | 添加路由规则 | 在 §四 表格中新增行 |
| `删除路由规则第 3 条` | 删除路由规则 | 从 §四 表格删除 |
| `显示当前配置` | 查看配置 | 仅读取和显示，不修改 |
| `把所有路径配好，LLVM 在 /opt/llvm-19` | 批量推导 | 从 LLVM_BUILD 推导 FILECHECK、LLVM_EXTRACT 等路径 |

### Step 3: 解析变量名

`CLAUDE.md.template` 中的占位符格式为 `{{VAR_NAME}}`。当用户设置一个变量时，按推导规则自动填充关联变量。

**推导规则**——当用户仅设置 `LLVM_BUILD` 时：

```
FILECHECK      = ${LLVM_BUILD}/bin/FileCheck        # 若未单独指定
LLVM_EXTRACT   = ${LLVM_BUILD}/bin/llvm-extract     # 若未单独指定
LLVM_REDUCE     = ${LLVM_BUILD}/bin/llvm-reduce      # 若未单独指定
CLANG          = ${LLVM_BUILD}/bin/clang             # 若未单独指定
```

**占位符完整列表：**

| 占位符 | 含义 | 默认值推导 |
|--------|------|-----------|
| `{{LLVM_BUILD}}` | LLVM 构建目录 | 无默认值，必须设置 |
| `{{FILECHECK}}` | FileCheck 二进制 | `${LLVM_BUILD}/bin/FileCheck` |
| `{{LLVM_EXTRACT}}` | llvm-extract 二进制 | `${LLVM_BUILD}/bin/llvm-extract` |
| `{{LLVM_REDUCE}}` | llvm-reduce 二进制 | `${LLVM_BUILD}/bin/llvm-reduce` |
| `{{PYTHON}}` | Python 解释器 | `python3` |
| `{{CLANG}}` | clang 二进制 | `${LLVM_BUILD}/bin/clang` |
| `{{TEST_SUITE_DIR}}` | 端到端测试目录 | 无默认值（可选） |

### Step 4: 生成修改预览

**绝不直接写入。** 先输出预览：

```
配置修改预览:

  环境变量:
    LLVM_BUILD:  /home/user/llvm-build → /opt/llvm-20/build
    FILECHECK:   未配置 → /opt/llvm-20/build/bin/FileCheck (推导)
    LLVM_EXTRACT: 未配置 → /opt/llvm-20/build/bin/llvm-extract (推导)
    LLVM_REDUCE:   未配置 → /opt/llvm-20/build/bin/llvm-reduce (推导)
    CLANG:          未配置 → /opt/llvm-20/build/bin/clang (推导)

  影响范围:
    - /verify 将使用新路径生成验证命令
    - /test-gen Phase 3 将使用新路径进行 IR 缩减

  确认修改? (回复 y 确认, 或提供调整意见)
```

### Step 5: 用户确认后写入

收到 `y` 后：
1. 对目标文件执行对应的字符串替换
2. 输出 diff（old → new）
3. 确认写入完成

收到调整意见后：回 Step 4，输出更新后的预览。

### Step 6: 写入后提醒

```
配置已更新: CLAUDE.md

如果新配置改变了工具路径，建议:
  - 运行 /verify 验证工具链是否就绪
  - 如果路径无效，/verify 会在命令生成时报错
```

## 路由规则的增删改

### 添加路由规则

用户：`添加路由规则：/commit-review 完成后建议 /verify`

预览：
```
工作流路由 新增:

  | /commit-review 通过 | 用户决定提交 | 建议 /verify — 提交后自动生成验证命令 |
```

### 删除路由规则

用户：`删除路由规则第 5 条`

预览后确认删除。

### 修改路由规则

用户：`修改路由规则第 3 条，建议操作改为 /commit-review`

预览：
```
工作流路由 修改:

  旧:  | /micro 完成代码变更 | 变更满足要求 | 建议 /pre-plan |
  新:  | /micro 完成代码变更 | 变更满足要求 | 建议 /commit-review |
```

## 允许修改的范围

| 节 | 允许操作 |
|----|---------|
| §一 环境配置 | 修改变量值 |
| §三 命令索引 | 增加/更新条目（Skill 实现后） |
| §四 工作流路由 | 增/删/改规则 |
| §六 快速参考 | 添加新的典型工作流 |

## 禁止修改的范围

| 节 | 原因 |
|----|------|
| §二 工程技术规范 | 核心方法论，不应通过工具临时修改 |
| §五 禁止事项 | 仅可追加，不可删除 |

## 处理 CLAUDE.md 文件尚未生成的情况

当用户首次运行 `/config` 且 `./CLAUDE.md` 不存在时：

1. 检查 `./CLAUDE.md.template` 是否存在，若存在则复制为 `./CLAUDE.md`
2. 然后按正常流程修改 `./CLAUDE.md`

当用户设置了 `LLVM_BUILD` 后，在执行 Step 5 写入时：
- 将 `{{LLVM_BUILD}}` 替换为实际路径
- 将 `{{FILECHECK}}` 等有推导值的占位符一并替换
- `{{TEST_SUITE_DIR}}` 等可选变量保持不变（留空）
- 将模板格式 `{{VAR}}` 转换为注释标记格式：`LLVM_BUILD=/opt/llvm/build`

## Edge Cases

| 场景 | 行为 |
|------|------|
| 用户设置的路径不存在 | 警告但允许（路径可能在后续构建后才存在） |
| 用户设置的值与推导值冲突 | 以用户显式设置的值为准，并在推导项上标注"跳过（用户已手动设置）" |
| 用户尝试修改被禁止的节 | 拒绝，给出原因和替代建议 |
| CLAUDE.md 和 CLAUDE.md.template 都不存在 | 报错"无法找到配置文件模板，请确认仓库完整性" |
| 用户输入不含配置意图 | 引导："你可以设置环境变量（如 LLVM_BUILD=...）、添加路由规则、或修改命令索引" |
