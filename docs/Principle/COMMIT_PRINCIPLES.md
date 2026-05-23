# Commit/PR 规则化工作流程

基于 GCC、LLVM、Triton、IREE 四个编译器项目的贡献实践整理。

---

## 一、提交前自检清单

### 规则 1：单一关注点 (Single Concern)

一个 commit = 一个目的。

- **禁止**：同一个 commit 中混合 bugfix + 新功能 + 重构
- **判断标准**：无法用一句话描述这个 commit 做了什么，就需要拆分
- **NFC 分离**：格式化、重命名、注释修正等非功能性改动单独一个 commit，标题加 `[NFC]` 标记
- **来源**：GCC "changes for different reasons must be sent individually" + LLVM NFC 约定

### 规则 2：测试完备性 (Test Coverage)

- 新功能 → 测试
- Bugfix → 回归测试
- 重构 → 现有测试全绿

| 改动类型   | 测试要求                                                  |
| ---------- | --------------------------------------------------------- |
| 新功能     | 必须附带测试，覆盖正常路径 + 边界条件 + 错误路径          |
| Bugfix     | 必须附带最小化回归测试，能复现原始 bug                    |
| 重构       | 不改测试，但所有现有测试必须通过                          |
| 性能优化   | 必须有基准测试数据支撑（否则可能因复杂度收益比不佳被拒）  |

- **最小化原则**：不允许把整个失败程序扔进测试集，用工具裁剪到最小复现用例（参考 LLVM `llvm-reduce` 思路）
- **来源**：LLVM + GCC + Triton（三者完全一致）

### 规则 3：独立验证 (Independent Verification)

提交前，本地完成以下三项验证，缺一不可：

**3a. 构建验证**
- C/C++ 项目：至少在一个平台上 `-Wall -Werror` 零警告通过
- 编译器项目（GCC 级要求）：需完成完整的 bootstrap 构建
- 多平台项目：在所有能接触到的平台上验证

**3b. 测试套件验证**
```
# 最小验证（必须）
./run_tests.sh  # 或等效命令

# 完整验证（涉及核心逻辑时必须）
./run_full_test_suite.sh
```

**3c. 独立环境验证**
- 在不同的机器/容器/CI 环境中重新验证
- 如果在 Docker 中开发，在裸机上也验证一次（反过来同理）
- 核心理念：不能只在你自己的机器上能跑
- **来源**：GCC 要求附上 host/target 组合和测试结果 + IREE "all checks passing before merge"

### 规则 4：Commit Message 格式

```
[Component] 一句话摘要（不超过 75 字符）

空行

主体部分说明：
- 为什么要做这个改动（WHAT 读者能看代码，WHY 才是你该写的）
- 如果修的是某个 commit 引入的 bug，引用那个 commit hash
- 关联的 issue/PR 链接

Signed-off-by: Your Name <email>      # 如果需要 DCO
Co-authored-by: Other Name <email>    # 如果多人协作
```

- **禁止** `@` 提及他人（cherry-pick 时产生垃圾通知）
- **PR 标题 = squash 后的 commit 标题**，最终编辑由作者和 reviewer 共同完成
- **来源**：LLVM + GCC + IREE

---

## 二、PR 生命周期规则

### 规则 5：PR 规模控制

一个 PR = 一个 reviewer 能在一个 review session 内理解完的改动量。

- **量化参考**：超过 400 行净增代码的 PR 应拆分为多个独立 PR
- 大改动拆分为 (1) 基础设施 → (2) 核心逻辑 → (3) 集成开关 的增量序列
- 拆分后的每个小 PR 必须独立可合入、独立可通过测试
- **来源**：LLVM 增量开发原则 + IREE "small and focused"

### 规则 6：审查前自审 (Self-Review)

在点击 "Request Review" 之前，必须自己完整过一遍 diff。

操作步骤：
1. `git diff main...HEAD`，逐文件看一遍
2. 检查是否有未删除的调试代码（`console.log`、`printf`、`FIXME`）
3. 检查是否有意外引入的空白变化
4. 确认每个改动都能解释"为什么"
5. PR 描述中写明：做了什么、为什么这样做、如何测试的、有什么风险
6. 如果用了 AI 工具辅助生成代码 → 标注 `Assisted-by`

**来源**：LLVM "PR description should evolve during review" + IREE AI 工具策略 + Triton "自行确保高质量"

### 规则 7：审查反馈处理

| 反馈类型        | 处理方式                                     |
| --------------- | -------------------------------------------- |
| 阻塞性意见      | 必须修改后才能合并                           |
| nit / optional  | 可自行决定是否修改，不阻塞合并               |
| post-merge 反馈 | 重大反馈 → revert 后处理，不在原 commit 上堆补丁 |

**来源**：LLVM "substantial post-commit feedback → revert" + IREE review 约定

---

## 三、合并与回溯规则

### 规则 8：合并条件

以下三项全部满足才可合并：

1. 所有 CI 检查绿色
2. 至少一位合格 reviewer 批准
3. 所有阻塞性审查意见已解决

**来源**：四个项目的共同原则

### 规则 9：Revert-to-Green

| 场景                              | 行动                    |
| --------------------------------- | ----------------------- |
| CI / 构建 bot 红了                | 立即 revert，然后调查   |
| 合并后收到重大审查意见            | revert，处理后再提交    |
| 合并后发现问题，可快速修复 (<1h)  | 发修复 PR               |
| 合并后发现问题，原因不明          | revert 为先，不猜测     |

**原则**：revert 是正常操作，不是耻辱。主干保持绿色比任何单个 PR 都重要。

**来源**：LLVM revert-to-green 文化

---

## 四、核查表 (Checklist)

每个 commit/PR 前逐项对照：

```
□ 1. 这个 commit 只做了一件事？（是 / 拆分）
□ 2. 有测试吗？测试覆盖了正常路径和边界吗？
□ 3. 本地构建零警告通过了吗？
□ 4. 本地测试套件全绿了吗？
□ 5. 在独立环境中验证过了吗？
□ 6. Commit message 标题 ≤ 75 字符，包含组件标签，正文解释了 WHY 吗？
□ 7. PR 规模合适吗（≤ 400 行净增）？
□ 8. 自己逐行过完 diff 了吗（无调试代码、无意外改动）？
□ 9. 如果是大改动，发 RFC 或提前讨论了吗？
□ 10. CI 全绿 + reviewer 批准 + 阻塞意见已解决？
```

---

## 五、参考来源

- [LLVM Developer Policy](https://llvm.org/docs/DeveloperPolicy.html)
- [GCC Contribution Guidelines](https://gcc.gnu.org/contribute.html)
- [Triton CONTRIBUTING.md](https://github.com/triton-lang/triton/blob/main/CONTRIBUTING.md)
- [IREE Contributing Guide](https://iree.dev/developers/general/contributing/)
