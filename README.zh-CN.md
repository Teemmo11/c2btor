# C2BTOR 2.0

[English](README.md) · [简体中文](README.zh-CN.md)

C2BTOR 将 C 程序转换为位精确的 [BTOR2](https://github.com/hwmcc/btor2tools) 状态迁移系统。你可以将生成的模型交给外部模型检测器，检查断言是否可能失败、错误函数是否可达；也可以将标准 BTOR2 反例映射回 C 程序，生成供独立验证的 violation witness。

项目复用 [CBMC](https://github.com/diffblue/cbmc) 的 C 前端、目标机器配置、GotoIR 和程序变换。构建目标与可执行文件均为 **`c2btor`**，产品版本为 **`2.0.0`**。本次源码发布面向 **Linux x86-64**，外部 checker 和预编译二进制需自行安装。

## 从源码构建

需要支持 C++17 的编译器、CMake 3.8 或更新版本、Make 或 Ninja、Flex、Bison、Bash 和 `patch`。Ubuntu/Debian 示例：

```sh
sudo apt-get update
sudo apt-get install build-essential cmake ninja-build flex bison patch git
git clone https://github.com/westtide/c2btor.git
cd c2btor
cmake -S . -B build -G Ninja -DWITH_JBMC=OFF -DCMAKE_BUILD_TYPE=Release
cmake --build build --target c2btor -j4
build/bin/c2btor --version
```

默认配置会下载并打补丁构建 MiniSat，需要网络。已有系统 MiniSat 时的配置及排错方法见 [COMPILING.md](COMPILING.md)。本仓库未包含 JBMC。请明确构建 `c2btor` 目标；保留的上游源码还定义了其他 CBMC 辅助工具。

## 先转换一个小程序

在仓库根目录执行以下命令，创建带一条断言的示例：

```sh
mkdir -p work
cat > work/example.c <<'EOF'
extern int __VERIFIER_nondet_int(void);
int main(void) {
  int x = __VERIFIER_nondet_int();
  __CPROVER_assume(x >= 0 && x <= 10);
  __CPROVER_assert(x < 10, "x must be below 10");
  return 0;
}
EOF
build/bin/c2btor work/example.c --goto-btor2 --inline --64 \
  --goto-btor2-out work/example.btor2 \
  --goto-btor2-map-out work/example.map.json \
  --no-standard-checks --no-pointer-check \
  --no-bounds-check --no-built-in-assertions
```

命令生成模型和源码映射。`x == 10` 时断言会失败，但转换命令本身不判定该状态是否可达。安装 Btor2Tools 后，可运行 `catbtor work/example.btor2` 检查模型语法和类型，再交给所选 BTOR2 模型检测器。parser 通过只表示模型格式有效。

示例关闭 CBMC 的标准安全检查插桩，主要检查程序自身的断言；C2BTOR 自身的内存有效性和模型边界诊断仍保留。`--64` 选择 Linux LP64，ILP32 任务使用 `--32`；这些选项不会把所有整数统一改成 64 位或 32 位。

## 按用途选择参数

参数值用空格传入，例如 `--memory object`。以下转换参数需配合 `--goto-btor2` 使用。

| 参数 | 含义与适用场景 |
| --- | --- |
| `--goto-btor2` | 生成 BTOR2 状态系统，进入转换流程。 |
| `--32` / `--64` | 选择 C 程序的目标数据模型：ILP32 或 Linux LP64。按程序/任务选择，不要仅按本机位数选择。 |
| `--no-standard-checks` | 关闭 CBMC 默认安全检查插桩；不会删除源断言或 C2BTOR 自身的诊断。 |
| `--goto-btor2-out FILE` | 将模型写入文件；省略时输出到 stdout。 |
| `--goto-btor2-map-out FILE` | 保存模型节点、GotoIR 与 C 源位置之间的对应关系，以及属性和建模信息。映射文件与模型文件必须不同。 |
| `--inline` | 转换前内联普通函数调用；有辅助函数的程序通常需要它。递归调用栈尚不支持。 |
| `--memory object` / `--memory global` | 选择每个对象独立存储，或使用统一的全局存储。默认 `object`。 |
| `--array bv` / `--array array` | 选择位向量或 BTOR2 数组存储。默认 `bv`。改变的是表示方式，预期内存语义应保持一致。 |
| `--memory-object-max-bytes N` | 在 object 模式中显式限制运行时对象存储容量，便于控制模型大小；未指定时从最终 GotoIR 推导。超出上限属于模型边界。 |
| `--array-bv-max-object-bytes N` | 限制 BV 存储的运行时对象容量；object/BV 模式下若同时指定两个容量参数，数值必须相同。 |
| `--goto-btor2-heap-objects K` | 限制总成功动态分配次数，默认 `32`。`free` 不归还这份对象身份预算。 |
| `--goto-btor2-checks` | 转换前插入所选 CBMC 安全检查。例如配合 `--no-standard-checks --bounds-check`，只请求数组越界检查，而不启用全部标准检查。 |
| `--goto-btor2-error-function NAME` | 在内联前将指定错误函数（如 `reach_error`）的调用转成 unreach-call 属性；配合 `--inline`。 |
| `--goto-btor2-reach-only` | 只选择该错误函数对应的源属性；必须同时指定 `--goto-btor2-error-function NAME`。 |
| `--goto-btor2-merge-bads` | 将选中的源属性用 OR 合并为一个 bad，供需要单属性的后端使用；别名为 `--goto-btor2-merge-properties`。 |
| `--goto-btor2-no-heap-guards` | 隐藏辅助内存/模型边界 bad 报告；路径冻结和验证义务仍然存在。仅使用此选项不能据此宣称 C 程序 SAFE。 |

再创建一个完整的错误调用示例：当 `x == 10` 时，程序会调用 `reach_error()`。

```sh
cat > work/reach.c <<'EOF'
extern int __VERIFIER_nondet_int(void);
void reach_error(void) { __CPROVER_assert(0, "error reached"); }
int main(void) {
  int x = __VERIFIER_nondet_int();
  if(x == 10) reach_error();
  return 0;
}
EOF
build/bin/c2btor work/reach.c --goto-btor2 --inline --64 \
  --goto-btor2-error-function reach_error --goto-btor2-reach-only \
  --goto-btor2-merge-bads --goto-btor2-out work/reach.btor2 \
  --no-standard-checks --no-pointer-check \
  --no-bounds-check --no-built-in-assertions
```

`--goto-btor2-error-function` 应与程序的实际错误函数名一致。需要包含 malloc 返回 NULL 的失败分支时，添加 `--malloc-may-fail --malloc-fail-null`。完整接口可用 `build/bin/c2btor --help` 查看。

## 属性、源码映射与反例

BTOR2 的 `bad` 表示违反性质的条件。源断言、`memory_validity` 内存诊断、`model_limit` 模型边界分别解释：容量不足不是 C 源程序中错误函数可达的反例。有界搜索未发现反例也不等于无界 SAFE；要给出 C SAFE 结论，还需检查相关验证义务和支持边界。

直接 CLI 生成的映射用于查看属性和源码位置。导出 witness 时，使用维护的 wrapper 绑定模型与源码哈希：

```sh
cat > work/reach.prp <<'EOF'
CHECK( init(main()), LTL(G ! call(reach_error())) )
EOF
python3 scripts/c2btor_witness.py export \
  --c2btor build/bin/c2btor --program work/reach.c \
  --spec work/reach.prp --output-dir work/witness-export -- --64
```

这里复用上一节的 `work/reach.c`，只导出模型和哈希绑定的映射；
此命令尚未运行求解器，也没有生成反例。最终输出目录必须尚不存在。wrapper 默认检查 `__VERIFIER_error` 的可达性，其他错误函数可通过 `--spec property.prp` 指定。维护流程为 BTOR2 反例 → BtorSim 回放 → SV-COMP violation YAML 2.0 → CPAchecker 验证。原始 CLI map 不能替代哈希绑定后的 witness bundle。回译、验证、依赖及支持的非确定性取值见 [witness 文档](docs/c2btor-witness.md)。

## 支持范围与限制

主要入口为带 `main` 的顺序 C 程序，支持机器整数、控制流、有限对象内存及已实现的 binary32/binary64 浮点操作。默认 `--memory object --array bv`：固定对象按准确大小存储，运行时对象的容量在生成模型时规划。

递归调用栈、并发和无界堆尚未实现。位域内存访问、指针/整数地址转换、重复声明同一取地址自动对象、读取未确定字节等仍有限制；完整 C provenance/effective-type 规则和浮点异常环境也尚未实现。C witness 当前面向 unreach-call，支持可定位的标量整数/布尔非确定性返回值；SAFE 不生成 invariant witness。超出这些范围时，请先查阅详细文档。

## 文档与回归

| 文档 | 内容 |
| --- | --- |
| [对象内存](docs/c2btor-object-memory.md) | 对象身份、别名、字节存储与生命周期。 |
| [数组编码](docs/c2btor-array-encoding.md) | 位向量与数组存储的区别。 |
| [容量规划](docs/c2btor-memory-capacity.md) | 运行时对象容量及模型边界。 |
| [浮点设计](docs/ieee754_kratos_analysis.md) | IEEE-754 lowering 与支持操作。 |
| [回归说明](regression/goto-btor2/README.md) | 使用独立安装的工具运行转换回归。 |
| [发布整理记录](RELEASE_PREPARATION.md) | 2.0 来源提交、清理范围和保留的构建依赖。 |

`src/` 包含 C2BTOR 与复用的 CBMC 构建依赖，`scripts/` 包含 witness 工具，`regression/goto-btor2/` 包含转换回归。上游测试、版权头和许可声明均保留。`unit/goto-programs/` 中的 5 个二进制格式文件
是解析器测试输入，为保持测试完整而保留，不是发布的转换器/checker 工具。带日期的验证文档是历史证据，不能当成本次 checkout 的新测试结果。问题请提交到 [westtide/c2btor Issues](https://github.com/westtide/c2btor/issues)。

## 致谢与许可

C2BTOR 基于 CBMC/CProver。This product includes software developed by Daniel Kroening, Edmund Clarke, Computer Science Department, University of Oxford, Computer Science Department, Carnegie Mellon University.

迁入源码保留[上游 4-clause BSD 许可](LICENSE)及各文件自己的声明。[LICENSE.c2btor](LICENSE.c2btor) 保存目标仓库原有 BSD 3-Clause 声明，不替代迁入源码的许可条件。[Btor2Tools](https://github.com/hwmcc/btor2tools)、[Boolector/BtorMC](https://github.com/Boolector/boolector)、[CPAchecker](https://github.com/sosy-lab/cpachecker) 等外部工具需单独安装，并遵循各自许可。
