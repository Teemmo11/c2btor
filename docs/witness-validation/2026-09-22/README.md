# C2Btor violation-witness validation — 2026-09-22

在 **109 个不同的 SV-COMP 原始任务、16 个目录类别**上完成了 164 次流水线尝试。
其中 **55 个任务**完成了 BTOR2 反例求解、BtorSim 检查、YAML 2.0 回译，
并获得 CPAchecker witness validator 的 `FALSE` 确认及进程退出码 0。
另有 7 个最小程序组成的集成回归，38 项检查全部通过。

这里的 `FALSE` 表示 validator 确认了 witness 约束下的 C 错误执行。
在 witness-validation 模式下，`TRUE` 只表示该 witness 没有得到确认，
**不能据此把原任务归类为 SAFE**。生成了 BTOR2 或 YAML 文件也不算验证成功。

## 本次代码修改

- 删除旧 `--witness-to-graphml` 及其附带的 YAML 入口、专用 C++ parser/replay
  和构建项。CBMC 本身的独立 `--graphml-witness` 功能保留。
- 使用 `--goto-btor2-map-out` 导出实际转换过程中的 PC、GotoIR、源位置、
  状态变量、nondet 返回值和 bad property 映射；Python 包装器绑定源码和模型哈希。
- `--goto-btor2-error-function` 在 inline 前标记属性指定的错误函数调用，
  支持原始 SV-COMP `reach_error()`，保留调用点而非被调用函数内部断言位置。
- 回译通过 BtorSim 重建状态；返回值来自执行调用后的状态，保留循环中重复调用。
  对同一行多个调用，新增直接返回值到目标变量的数据流信息，支持多次内联的 helper。
- malloc/calloc 库模型内部 nondet 保留在电路 trace 中，不生成不存在的源码调用 waypoint。
- CPAchecker 显式使用 predicate witness validator；关闭会触发本机 PRINCESS
  formula-dump 崩溃的附加文件导出，仍然要求分析成功、退出码 0 且结果为 `FALSE`。
- 新增读取 SV-COMP task YAML 的批量脚本，保存任务清单、命令、日志、各阶段耗时
  和采样 RSS，区分转换、求解、回译、验证阶段失败。

SAFE 不生成 correctness/invariant witness。主用法见
[c2btor-witness.md](../../c2btor-witness.md)。

## 选样和运行条件

输入来自本机 `/Users/west/Developer/sv-benchmarks/c`，没有重写原始 C 程序。
只选择 task YAML 中 `unreach-call.prp` 的 `expected_verdict: false`、单输入文件任务；
使用任务各自的 ILP32/LP64 数据模型。

第一批按源码大小，从默认 14 个目录各取最多 5 个任务，共 58 个。
第二批用 BtorMC 重试其中 36 个 rIC3 求解失败/超时任务。
第三批补充 51 个未出现过的任务，包括数组、链表、循环和 LDV 程序。
第四批在修复 validator 导出问题后重试 17 个任务；第五批重试两个同一行多调用任务。
每次选样清单在运行前保存；失败尝试完整保留。
这是一组偏向小程序的功能样本，**不代表 SV-COMP 全集通过率或工具性能比较**。

工具及限制：

| 项目 | 设置 |
|---|---|
| CBMC/C2Btor | 本地修改后构建，CBMC 6.7.1 |
| 后端 | rIC3 1.5.2 BMC；BtorMC 3.2.1 |
| 电路检查 | 本地 Btor2Tools `btorsim --states` |
| C validator | CPAchecker 4.2.2-1141-gd28fc1d549+，Java 21 |
| SMT | PRINCESS，精确 bitvector；浮点编码显式禁用 |
| 转换 | 20 秒；默认 inline 和项目规定的检查关闭参数 |
| 求解 | BMC 上限 400；rIC3 内部 15 秒；外层最多 20 秒 |
| 回放 | 20 秒，外层最多 25 秒 |
| 验证 | 首轮 25 秒 CPU；17 个复测任务提高至 60 秒；同一行调用复测 35 秒 |
| 资源 | 同时最多 2 个任务；每任务进程树 RSS 上限 4 GiB；Java heap 2 GiB |

PRINCESS 此处未使用 integer/rational approximation。
4 GiB 是按 0.2 秒间隔采样执行的限制；工具自带 CPU 限制和外层 wall-time 限制分别保留。
[tools.json](tools.json) 是最终构建/工具二进制的 SHA-256 快照；早期批次使用的 C2Btor
缺少后续补充的调用目标元数据，不能把最终二进制哈希当成每次历史运行的哈希。

## 结果

每个任务按最后一次尝试归类。本次复测没有让已确认的任务退化。

| 最终状态 | 任务数 | 含义 |
|---|---:|---|
| `confirmed` | 55 | CPAchecker 退出 0 且 `FALSE` |
| `solver_unknown` | 27 | 后端未取得反例；包括达到有限 BMC 上限 |
| `solver_timeout` | 7 | 后端时间限制 |
| `conversion_error` | 6 | 真实递归不支持，转换器明确拒绝 |
| `translation_error` | 5 | 浮点 nondet 返回值尚不支持回译 |
| `validation_rejected` | 4 | CPAchecker 返回 `TRUE`，未确认该反例 |
| `validation_error` | 3 | 浮点模式不支持 1 个；PRINCESS/validator 错误 2 个 |
| `validation_unknown` | 2 | 提高到 60 秒后仍因 CPU 限制返回 UNKNOWN |
| 合计 | 109 | 原始任务去重数 |

阶段数量：103 个任务生成模型，69 个取得并通过 BtorSim 检查的电路反例，
64 个生成 YAML witness，55 个获 C validator 确认。不能将 55/64 当作全集通过率。
第一批曾有 31 个 rIC3 进程错误；所有这些任务都保留在历史结果中，之后用 BtorMC 重试。

| SV-COMP 目录 | 原始任务数 | CPAchecker 确认 |
|---|---:|---:|
| array-examples | 5 | 0 |
| array-fpi | 15 | 11 |
| array-industry-pattern | 6 | 0 |
| array-programs | 2 | 0 |
| bitvector | 5 | 0 |
| bitvector-regression | 6 | 4 |
| floats-cdfpl | 5 | 0 |
| ldv-regression | 17 | 16 |
| list-properties | 3 | 3 |
| loop-acceleration | 15 | 6 |
| loop-crafted | 1 | 1 |
| loop-invariants | 1 | 1 |
| loop-invgen | 1 | 1 |
| loops | 17 | 12 |
| loops-crafted-1 | 5 | 0 |
| recursive-simple | 5 | 0 |

## 尚未解决的问题

1. **四个电路反例未获 C 层确认。** `loops/linear_search` 的 nondet 输入为 2，
   原程序 `SIZE = input / 2 + 1` 得到 2；按原程序有效内存语义，搜索应找到已写入的 3。
   电路 trace 却返回未找到。`array-fpi/s1iff`、`s2iff`、`s3iff` 都选择 `N=1`，
   原 C 中该输入下断言应成立。源返回值回译与电路值一致，说明仍需排查前端的
   heap/alias/数组建模；本次没有把未确认的反例包装成成功。这里只定位到模型与
   C 执行不一致，尚未完成具体编码规则的根因修复。
2. **浮点支持不完整。** `floats-cdfpl` 的五个任务已取得电路反例，但 `floatbv`
   nondet 返回值不能用当前整数返回约束回译。`bitvector-regression/implicitfloatconversion`
   的混合浮点验证也不被本次 PRINCESS 配置支持。
3. **Validator 限制。** `array-fpi/condnf` 的 CPAchecker 路径重放不一致并报错；
   `loops/invert_string-1` 的 PRINCESS 插值遇到未知 sort。
   没有打开 `allowImpreciseCounterexamples` 绕过它们。
   `ldv-regression/test24-2`、`loop-acceleration/diamond_1-2` 验证超时。
4. **后端/长路径。** rIC3 在部分 array/witness 导出上崩溃；BtorMC 交叉复测解决了
   部分任务，其余有限展开不足或超时。UNKNOWN 不能作为 SAFE 证据。

## 复现及证据

- [results.csv](results.csv)：109 个任务的最后状态、数据模型、源码哈希和原始产物路径。
- [results.json](results.json)：完整 164 次尝试及每任务尝试历史，保留原始状态分类。
- [tasks.txt](tasks.txt)：109 个去重任务，可传给批量脚本 `--tasks-file`。
- `evidence.zip`（完整本地归档，不随源码提交推送）：模型、映射、原始 backend witness、BtorSim 日志、
  C witness、CPAchecker 命令/输出/结果、原始 C/task/property 文件，以及 38 项回归证据。
  已检查 ZIP 全部条目的 CRC。为控制体积，未打包 CPAchecker 自动生成的 HTML/CFA 图等附加输出。
  原始绝对路径仍记录在映射和 witness 中；迁移机器后应重新 export。

示例（仓库根目录执行，工具路径按本机调整，输出目录必须不存在）：

```sh
python3 scripts/run_witness_svcomp.py \
  --benchmarks /Users/west/Developer/sv-benchmarks/c \
  --cbmc build/bin/cbmc \
  --ric3 'checker/macOS arm64/ric3' \
  --btorsim 'checker/macOS arm64/btorsim' \
  --backend btormc \
  --btormc /Users/west/Developer/all/simple_CAR/btor2/deps/btor2tools/deps/install/bin/btormc \
  --cpachecker /Users/west/Developer/cpachecker/bin/cpachecker \
  --tasks-file docs/witness-validation/2026-09-22/tasks.txt \
  --output-dir /tmp/c2btor-witness-new-run
```

批量脚本需要 PyYAML 和读取进程表的权限。命令中的工具路径、限时和源码哈希也可从
每次运行的 JSON/日志核对。回归包含故意修改返回值的自检，仅用于确认验证链路会识别
不成立的约束；它不计入 109 个原始任务或 55 个成功任务。
