# 编译器项目的测试金字塔实践

基于 GCC、LLVM、Triton、IREE 四个编译器的测试体系总结。

---

## 一、测试金字塔总览

经典测试金字塔提出：底层是大量快速、廉价的小测试，越往上测试规模越大、越慢、数量越少。编译器项目对此做了领域适配，形成了独特的 **五层金字塔结构**。

```
                        ┌─────────────┐
                        │  性能/基准   │  ← 最慢, 按需/夜间运行
                        │  Benchmarks  │
                        ├─────────────┤
                        │  端到端测试  │  ← 完整程序编译+执行+结果比对
                        │  E2E Tests   │
                        ├─────────────┤
                        │  集成测试    │  ← Pass/Pipeline/子系统联合验证
                        │ Integration  │
                        ├─────────────┤
                        │  回归测试    │  ← 核心层: IR片段, FileCheck断言, 数量最多
                        │ Regression   │
                        ├─────────────┤
                        │  单元测试    │  ← 数据结构/工具库, Google Test风格
                        │  Unit Tests   │
                        └─────────────┘
```

### 各层关键指标

| 层次     | 数量级     | 单用例耗时  | 运行频率     | 失败含义                       |
| -------- | ---------- | ----------- | ------------ | ------------------------------ |
| 单元     | ~千级      | < 10ms      | 每次 commit  | 数据结构/工具逻辑出错           |
| 回归     | ~万级      | < 100ms     | 每次 commit  | IR变换/PASS行为不符合预期       |
| 集成     | ~千级      | < 1s        | 每次 PR      | 子系统/Pass Pipeline交互出错    |
| 端到端   | ~百级      | 秒~分钟     | 每次 PR/合并 | 完整编译链路或运行时行为出错    |
| 性能基准 | ~十~百级   | 分钟~小时   | 夜间/按需    | 性能退化或编译资源消耗异常      |

---

## 二、LLVM 的实践

### 2.1 层级结构

```
Layer 1: 单元测试 (unittests/)
  框架: Google Test + Google Mock
  范围: 支持库和通用数据结构 (ADT, Support, etc.)
  运行: make check-llvm-unit
  原则: "reserved for targeting the support library and other generic data structure"
        对 IR 变换不做单元测试, 用回归测试替代

Layer 2: 回归测试 (test/)
  框架: Lit + FileCheck
  范围: IR变换、优化Pass、代码生成、分析Pass
  形式: .ll 文件 + RUN: 行 + FileCheck 断言
  运行: make check-llvm  (单文件: llvm-lit <path>)
  原则: "should be run before every commit"
        推荐使用自动生成的检查行 (update_llc_test_checks.py 等脚本)

Layer 3: 集成测试 (test/ 中的多Pass测试)
  框架: Lit (与回归测试同一框架, 但测试链更长)
  范围: 多Pass联合、前后端联合、LTO Pipeline
  形式: 多 RUN: 行, 逐步验证Pipeline各阶段输出

Layer 4: test-suite (独立仓库 llvm-test-suite)
  框架: Lit 驱动编译+执行+结果比对
  范围: 完整的 C/C++/Fortran 程序
  目录: SingleSource/ MultiSource/ External/
  原则: 每个程序编译后运行, 输出与参考结果比对
        同时收集 compile_time, exec_time, code_size 等指标
        历史上称 "nightly tests", 现在运行频率远高于夜间

Layer 5: 基准测试 (test-suite 中的 Benchmark 子集)
  框架: google-benchmark (MicroBenchmarks/)
  范围: SPEC CPU, MiBench, Rodinia 等标准基准套件
  运行: 夜间或按需, 需要专用硬件
  原则: 函数反复运行直到测量结果统计显著
        通过 TEST_SUITE_BENCHMARKING_ONLY 控制
```

### 2.2 Lit 测试的约束控制

```
REQUIRES:   所有表达式为真才启用该测试   (例: REQUIRES: asserts)
UNSUPPORTED: 任一表达式为真则禁用该测试   (例: UNSUPPORTED: system-windows)
XFAIL:       任一表达式为真则预期失败     (例: XFAIL: target-powerpc)
```

Lit 自动注入的特性: `system-darwin`, `system-linux`, `target-x86_64`, `target-aarch64`, `asan`, `tsan` 等。

### 2.3 测试编写规范

- 每个测试文件测试**一个明确的功能点**
- 用 `update_llc_test_checks.py` / `update_test_checks.py` 自动生成 FileCheck 断言
- 测试输入必须**最小化**：不允许把整个失败程序作为测试用例, 用 `llvm-reduce` 裁剪
- grep 用法已被标记废弃, 必须用 FileCheck 验证输出

---

## 三、GCC 的实践

### 3.1 层级结构

```
Layer 1: 编译测试 (compile.exp)
  范围: 验证代码能否编译通过
  形式: 编译源代码, 检查编译器是否正常退出
  原则: 只检查编译, 不运行生成的可执行文件

Layer 2: 执行测试 (execute.exp)
  范围: 编译后运行, 验证运行时行为
  形式: 编译 → 执行 → 检查输出/返回值
  原则: 用 abort() 标记失败, exit(0) 标记成功

Layer 3: dg 测试 (dg.exp)
  框架: DejaGnu 指令驱动
  形式: 源代码中嵌入 dg- 指令注释
  指令: { dg-do run/compile/link/assemble }
        { dg-options "-O2 -Wall" }
        { dg-output "expected output" }
        { dg-error "expected error message" }
        { dg-warning "expected warning" }
        { dg-bogus "unexpected diagnostic" }
  原则: 指令控制测试行为, DejaGnu 框架解析并执行

Layer 4: Torture 测试 (多优化组合)
  范围: 同一程序以大量优化选项组合反复编译执行
  形式: RUNTESTFLAGS="--target_board=unix/\{-O0,-O1,-O2,-O3,-Os\}"
  原则: 确保同一程序在所有优化级别下行为一致
        这是编译器特有的测试手段

Layer 5: Bootstrap + 全量回归
  范围: 用上一个版本的GCC编译当前版本, 反复三轮
  形式: make bootstrap && make check
  原则: 确保编译器自身在所有优化级别下行为一致
        结果发送至 gcc-testresults@ 邮件列表
```

### 3.2 测试选择与运行

```
# 全局
make check

# 语言层
make check-gcc     check-c    check-c++    check-fortran    check-lto

# 按测试类别过滤
make check-gcc RUNTESTFLAGS="compile.exp"
make check-gcc RUNTESTFLAGS="execute.exp=20050121*"

# 并行多组合
make -j3 check-gcc//sh-hms-sim/{-m1,-m2,-m3}/{,-nofpu}
```

### 3.3 结果文件

```
.sum 文件: 汇总各测试状态码 (PASS / XPASS / FAIL / XFAIL / UNSUPPORTED / ERROR / WARNING)
.log 文件: 编译器调用详情与输出明细
```

---

## 四、Triton 的实践

### 4.1 层级结构

```
Layer 1: 单元测试 (unit/)
  框架: pytest
  范围: 单一函数/模块的细粒度测试
  运行: CI 每次 commit, 高频快速执行

Layer 2: 子系统测试
  范围:
    backend/     — GPU后端接入与行为验证
    gluon/       — Gluon子系统功能验证
    gsan/        — GSAN子系统功能验证
  形式: pytest + shared fixtures (conftest.py)
  运行: 每次 PR

Layer 3: 正确性比对 (kernel_comparison/)
  范围: Triton kernel vs 参考实现 (如 cuBLAS) 的数值结果比对
  原则: 确保 Triton 生成的代码在数值上与业界标准实现一致

Layer 4: 性能微基准 (microbenchmark/)
  框架: 自定义性能测量
  范围: Kernel 执行时间、内存带宽等性能指标
  原则: 性能回归检测, 量化优化效果

Layer 5: 回归测试 (regression/)
  范围: 历史 bug 场景的复测用例
  原则: 独立管理, 确保修复不被后续变更破坏
```

### 4.2 Triton 测试策略特点

```
正确性 + 性能 双轨制
  正确性: unit → subsystem → kernel_comparison
  性能:   microbenchmark (独立轨道)

不同频率分层执行
  高频 (每次 push):  unit / backend / regression
  中频 (每次 PR):    kernel_comparison (小规模)
  低频 (夜间):       全量 kernel_comparison + microbenchmark
```

### 4.3 共享 Fixtures 模式

`conftest.py` 提供共享 fixtures: 设备初始化、环境参数、测试数据生成 等, 避免重复配置。

---

## 五、IREE 的实践

### 5.1 层级结构

```
Layer 1: Lit 回归测试 (tests/ 各子目录)
  框架: Lit + FileCheck (与 LLVM 相同体系)
  范围: 编译器各阶段 IR 变换的正确性

Layer 2: 编译器驱动测试 (compiler_driver/)
  范围: 编译器入口/驱动逻辑的专项测试
  性质: 单元测试与集成测试之间

Layer 3: Transform Dialect 测试 (transform_dialect/)
  范围: MLIR Transform Dialect 相关功能的正确性

Layer 4: 平台专项测试 (riscv32/ 等)
  范围: 特定后端架构的正确性验证
  原则: 跨平台兼容性保障

Layer 5: 端到端测试 (e2e/)
  范围: 完整编译 + 执行链路
  原则: 从源码到运行结果的完整正确性验证
```

### 5.2 IREE 测试策略特点

```
与 LLVM 生态深度集成
  - 使用 lit 作为测试运行器
  - 测试配置文件: lit.cfg.py
  - 构建系统: CMakeLists.txt / BUILD.bazel 双构建

CI 自定义
  - ci-skip:    跳过指定作业
  - ci-extra:   增加额外作业
  - ci-exactly: 精确指定作业集
  - skip-ci:    跳过所有作业

外部测试套件集成
  - tests/external/iree-test-suites/
  - 用于兼容性验证
```

---

## 六、四项目对照表

### 6.1 各层使用的框架

| 层次       | LLVM              | GCC              | Triton          | IREE            |
| ---------- | ----------------- | ---------------- | --------------- | --------------- |
| 单元测试   | Google Test       | (无独立框架)     | pytest          | (lit/cc_test)   |
| 回归测试   | Lit + FileCheck   | DejaGnu + dg     | pytest          | Lit + FileCheck |
| 集成测试   | Lit (多Pass链)    | dg + execute     | subsystem pytest| Lit (多阶段)    |
| 端到端     | test-suite        | Bootstrap + 执行 | kernel_compare  | e2e/            |
| 性能基准   | test-suite + SPEC | (无内置)         | microbenchmark  | 外部测试套件    |

### 6.2 测试规模与耗时对比

| 层次       | 数量级 (LLVM)      | 数量级 (GCC)     | 运行频率            |
| ---------- | ------------------ | ---------------- | ------------------- |
| 单元测试   | ~6,000             | —                | 每次 commit         |
| 回归测试   | ~40,000+           | ~200,000+        | 每次 commit         |
| 集成测试   | 同回归层, 更长链    | ~10,000 (execute)| 每次 PR             |
| 端到端     | ~2,000 (test-suite)| Bootstrap 3轮    | PR合并前 / 部分夜间 |
| 性能基准   | ~200 (SPEC子集)    | —                | 夜间 / 版本发布前   |

---

## 七、规则化原则

### 原则 1: 每一层有明确的守卫边界

```
回归测试守卫 IR 层正确性   →  集成测试守卫 Pipeline 正确性  →  E2E 守卫最终产物正确性
```

- 下层失败: 问题在本模块, 容易定位
- 上层失败但下层全绿: 问题在模块间交互, 回溯集成链路
- **禁止**跳过下层直接在顶层加测试 (会导致问题定位成本暴增)

### 原则 2: 测试输入必须最小化

```
单个测试文件 = 复现一个明确场景所需的最小输入

做法:  截取失败代码的 IR dump → 用工具裁剪 → 手工精简 → FileCheck断言
工具:  llvm-reduce, bugpoint, creduce, cvise
反例:  把整个 5000 行的 C 文件作为测试输入
```

**来源**: LLVM "It is not acceptable to place an entire failing program into llvm/test" + GCC 测试命名规范 (每个测试文件名对应一个PR编号)

### 原则 3: 自动化生成断言

```
手工 FileCheck / dg 断言  →  容易遗漏边界  →  维护成本高

推荐:
  LLVM:    update_test_checks.py, update_llc_test_checks.py
  GCC:     dg 指令覆盖诊断输出 (error/warning/bogus)
  Triton:  pytest parametrize + hypothesis (属性测试)
  IREE:    lit 自动生成检查行
```

### 原则 4: 条件测试覆盖多平台/多配置

```
不是为每个平台写单独的测试, 而是让同一测试知道在什么条件下该跑/该跳过

LLVM/IREE lit:
  REQUIRES: asserts           # 只在 assertion 构建下运行
  UNSUPPORTED: system-windows # 在 Windows 上跳过
  XFAIL: target-powerpc       # PowerPC 上已知失败

GCC DejaGnu:
  { dg-do run { target native } }
  { dg-require-effective-target vect_int }

Triton pytest:
  @pytest.mark.skipif(not torch.cuda.is_available(), reason="CUDA required")
```

### 原则 5: 测试失败必须阻断合并

```
CI Pipeline 检查顺序:
  1. 格式检查 (clang-format, black, lint)    → 30s 内
  2. 单元测试 + 回归测试                       → 5min 内
  3. 集成测试                                  → 15min 内
  4. E2E / test-suite                          → 1h 内
  5. 性能基准 (仅夜间/版本发布前)              → 数小时

合并条件: 1-4 全部绿色 (5 的信息不阻塞合并, 但需记录退化趋势)
```

### 原则 6: Torture 测试是编译器特有的正确性保障

```
同一程序 × 全部优化组合 → 输出必须一致

GCC:  --target_board=unix/\{-O0,-O1,-O2,-O3,-Os\}
LLVM: 不同 Pass Pipeline 配置的 test-suite 批量运行
Triton: 不同后端 (CUDA/HIP) 上 kernel_comparison 结果一致

核心理念: 编译器优化不能改变程序语义。任何优化组合下的行为差异 = bug。
```

### 原则 7: 回归测试独立管理

```
Bug 发现
  → 复现 → 最小化
  → 写入 regression/ 目录 (或 test/ 中标注 PR 编号)
  → 验证修复通过
  → 合并后该测试永久保留

四个项目皆有独立的 regression 目录或命名约定:
  LLVM:  test/ 中归入对应组件目录, PR 编号在 commit message 中引用
  GCC:   gcc/testsuite/ 中按语言组织, 文件名 = PR编号
  Triton: python/test/regression/
  IREE:  lit 测试中标注关联 issue
```

### 原则 8: 性能测试不阻塞合并, 但需追踪趋势

```
性能回归检测策略:
  按需运行 → CI 不自动跑
  夜间批量 → 生成趋势报告
  版本发布前 → 全量基准比对
  使用专用硬件 → 确保测量一致性

退化处理:
  在允许误差范围内的退化 → 记录, 后续观察
  显著退化 (>3% / >5%)     → 创建 issue, 评估是否回退
```

---

## 八、核查表 (Testing Checklist)

每个新功能/Bugfix 提交前:

```
□ 1. 是否已在最底层添加了测试? (优先回归测试, 其次单元测试)
□ 2. 测试输入是否已最小化? (用裁剪工具, 不允许整个程序当测试用例)
□ 3. 断言是否自动生成? (update_test_checks / dg指令 / pytest parametrize)
□ 4. 是否标注了平台条件? (REQUIRES / UNSUPPORTED / XFAIL / skipif)
□ 5. Bugfix 是否包含回归测试? (且文件名或commit message引用了PR/issue编号)
□ 6. 是否以不同优化级别/配置组合验证过? (Torture 思维)
□ 7. 非功能性改动是否标记 [NFC] 且不引入新测试?
□ 8. 性能敏感改动是否已在 microbenchmark/test-suite 上对比过?
□ 9. 所有测试层是否绿色? (按频率要求逐层检查)
```

---

## 九、参考来源

- [LLVM Testing Guide](https://llvm.org/docs/TestingGuide.html)
- [LLVM Test-Suite Guide](https://llvm.org/docs/TestSuiteGuide.html)
- [LLVM lit Command Guide](https://llvm.org/docs/CommandGuide/lit.html)
- [GCC Installing: Testing](https://gcc.gnu.org/install/test.html)
- [GCC Coding Conventions](https://gcc.gnu.org/codingconventions.html)
- [IREE Contributing](https://iree.dev/developers/general/contributing/)
- [Triton CONTRIBUTING](https://github.com/triton-lang/triton/blob/main/CONTRIBUTING.md)
