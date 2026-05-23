# MigMaster

> 基于 Claude Code 的软件开发方法论工具包，为 AI 辅助编译器开发强制执行工业级的提交规范和测试纪律。

MigMaster 将 LLVM、GCC、Triton、IREE 等成熟项目的贡献规范转化为一套可执行的 Artifact——Skill、Hook、Slash Command——将 AI 的代码生成行为约束到手工程序员的工程水准。目标用户是用 AI 工具探索编译器代码，但拒绝让 AI 侵蚀自身代码理解力的研究人员和研究生。

## 要解决的问题

AI 编程 Agent 产生的是大块、铁板一块的变更。你提一个需求，回来 500 行代码。代码可能跑通了，但你审不动、追溯不了设计决策、半年后无法恢复上下文。在编译器开发中这个问题是致命的：手写 IR 测试通过，端到端测试失败，"理想化 IR" 和 "真实 IR" 之间的裂缝静悄悄地扩大。

## 解决思路

MigMaster 将 AI 的输出视为**原材料**，而非成品。它提供：

- **门禁 Hook**——在你提交前提醒运行测试和审查变更
- **工作流命令**——将大型任务分解为可独立验证、净增 ≤150 行的原子提交
- **测试 Skill**——检测"抽象降级"：手写 IR 与真实 SYCL/模板展开 IR 之间的漂移
- **学习脚手架**——将仓库的 commit 历史转化为带可实操实验的结构化学习工作簿

核心理念：**AI 生成，人蒸馏。** 提交历史成为永久的文档——每个小 commit 都是一个可验证、可理解的独立知识单元。

## 文档导航

| 文档 | 用途 |
|------|------|
| [`docs/Principle/COMMIT_PRINCIPLES.md`](docs/Principle/COMMIT_PRINCIPLES.md) | 基于 LLVM/GCC/Triton/IREE 提炼的 Commit/PR 规则 |
| [`docs/Principle/TESTING_PYRAMID.md`](docs/Principle/TESTING_PYRAMID.md) | 面向编译器项目的五层测试体系 |
| [`docs/Principle/DOC_CLASSIFY.md`](docs/Principle/DOC_CLASSIFY.md) | 文档分类与维护策略 |
| [`docs/Discussion/1-Motivation.md`](docs/Discussion/1-Motivation.md) | 问题陈述与动机 |
| [`docs/FormalDesign-v1.md`](docs/FormalDesign-v1.md) | Artifact 完整规格说明（15 个） |

## Artifact 速览

| 层次 | Artifact |
|------|----------|
| **宪法** | `CLAUDE.md`——工程规范、环境配置、工作流路由 |
| **防线** | `Stop` Hook——任务结束后提醒运行测试 |
| **开发流程** | `/pre-plan`、`/micro`、`/commit-review`、`/verify` |
| **测试与分析** | `/test-gen`（四象限生成 + 抽象降级检测 + IR缩减）、`/split-commit` |
| **知识** | `/mentor`、`/learn-from`（学习工作簿生成）、`/archeology`（仓库考古）、`/adr` |
| **反思** | `/check-turn`（转向信号检测）、`/config`（描述性配置） |

完整的功能边界、输入输出契约、边界场景处理见 [`docs/FormalDesign-v1.md`](docs/FormalDesign-v1.md)。

## 快速开始

### 前置条件

- [Claude Code](https://claude.ai/code) CLI
- LLVM 19+（用于 IR 缩减和测试工作流）
- Python 3.8+

### 安装

```bash
# 克隆到任何你想要应用这套方法论的已有项目中
git clone https://github.com/your-org/MigMaster.git
cd MigMaster

# .claude/ 目录包含 Skill 和 Hook 定义
# CLAUDE.md 会被 Claude Code 在此目录中自动加载
```

### 第一步

1. **阅读动机文档**：[`docs/Discussion/1-Motivation.md`](docs/Discussion/1-Motivation.md)
2. **配置环境**：在 Claude Code 中运行 `/config`，设置 LLVM 构建路径和工具链位置
3. **从小任务开始**：编写代码前先用 `/pre-plan` 验证任务粒度
4. **原子执行**：每个 ≤150 行的实现步骤使用 `/micro`
5. **提交前审查**：使用 `/commit-review` 检查是否符合规范
6. **提交后验证**：使用 `/verify` 生成可手动执行的测试命令

## 项目结构

```
MigMaster/
├── CLAUDE.md                         # 项目宪法（待实现）
├── README.md                         # 英文入口
├── README_zhCN.md                    # 本文档（中文入口）
├── .claude/
│   └── settings.local.json           # Hook 配置
│   └── skills/                       # Skill 定义（待实现）
│       ├── pre-plan/SKILL.md
│       ├── micro/SKILL.md
│       ├── commit-review/SKILL.md
│       ├── verify/SKILL.md
│       ├── test-gen/SKILL.md
│       ├── split-commit/SKILL.md
│       ├── adr/SKILL.md
│       ├── archeology/SKILL.md
│       ├── mentor/SKILL.md
│       ├── learn-from/SKILL.md
│       ├── check-turn/SKILL.md
│       └── config/SKILL.md
└── docs/
    ├── FormalDesign-v1.md            # Artifact 规格说明
    ├── Principle/                    # 工程原则
    │   ├── COMMIT_PRINCIPLES.md
    │   ├── TESTING_PYRAMID.md
    │   └── DOC_CLASSIFY.md
    └── Discussion/                   # 设计讨论记录
        ├── 1-Motivation.md
        ├── 2-ARTIFACT_ANALYSIS.md
        ├── 3-Reply.md
        ├── 4-Analysis.md
        └── 5-FinalDesign.md
```

## 实现状态

- [x] 工程原则文档
- [x] Artifact 正式规格说明 (v1)
- [ ] CLAUDE.md (P0)
- [ ] Hook (P1)
- [ ] 核心命令：`/pre-plan`、`/micro`、`/commit-review`、`/verify` (P1)
- [ ] 分析 Skill：`/test-gen`、`/split-commit` (P2)
- [ ] 知识 Skill：`/learn-from`、`/adr`、`/config` (P2)
- [ ] 扩展 Skill：`/archeology`、`/mentor`、`/check-turn` (P3)

## 设计原则

1. **AI 生成，人蒸馏。** 进入手工库的每一行代码都经过人的判断。
2. **一个 commit 只做一件事。** 禁止在同一个 commit 中混合 bugfix、新功能和重构。
3. **测试不可妥协。** 功能性测试定义契约；回归测试抵御真实世界的复杂性。
4. **Revert-to-green。** 构建红了先 revert，后调查。主干绿色高于一切。
5. **小而可验证。** PR ≤ 200 行，每个 commit 独立编译并通过测试。

## 开源协议

Apache 2.0——与本工具包所借鉴的 LLVM 生态保持一致。

## 致谢

本文档化的工程实践提炼自以下项目的贡献者指南：

- [LLVM](https://llvm.org/docs/DeveloperPolicy.html)
- [GCC](https://gcc.gnu.org/contribute.html)
- [Triton](https://github.com/triton-lang/triton)
- [IREE](https://iree.dev/developers/general/contributing/)
