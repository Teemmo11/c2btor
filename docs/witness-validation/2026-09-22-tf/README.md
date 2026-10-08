# RQ1 / RQ2 TF witness 专项检查

对用户列出的 **57 个历史 TF case**进行了当前版本检查：RQ1 lab1 为 20 个，
RQ2 lab2 为 37 个。本次生成 **23 个通过 BtorSim 检查的 YAML 候选 witness**，
**0 个获得 CPAchecker 的 C violation 确认**。

| 集合 | 指定任务 | 后端 SAT 反例 | BtorSim 通过且生成 YAML | CPAchecker 确认 |
|---|---:|---:|---:|---:|
| RQ1 lab1 | 20 | 18 | 17 | 0 |
| RQ2 lab2 | 37 | 6 | 6 | 0 |
| 合计 | 57 | 24 | 23 | 0 |

这是“可以产出候选，但不等于存在真实 C 错误”的实测例子。
TF 的定义是预期安全而工具报错；若标签、任务语义和 validator 都正确，
真正的 TF 不应有可被可靠确认的 C violation witness。
候选验证失败也不能自动证明程序 SAFE。

## 每种结果的含义

23 个候选中：

- 13 个由 CPAchecker 返回 `TRUE`，未确认提交的 violation witness。
- 8 个在 25 秒 CPU 限制下返回 `UNKNOWN`。
- 2 个遭遇 PRINCESS/JavaSMT 错误：`array-fpi/pcomp.c` 的 bitvector 类型错误，
  `list-simple/dll2n_update_all.i` 的 arithmetic atom 错误。

其余 34 个：

- `array-fpi/sqm.c` 有后端 SAT 输出，但 BtorSim 报告 frame 80 的 array state
  assignment 不兼容（state ordinal 18，node 57），未生成 YAML。
  尚未区分后端 trace 导出与 simulator 兼容性问题。
- `loops/nec40.c` 在当前模型和 rIC3 portfolio 下返回 `UNSAT`，不生成反例 witness。
  不能将历史 TF 标签直接当作当前工具状态。
- 27 个在本次界限内未取得反例，包含 UNKNOWN 和超时；不是 SAFE 证明。
- 五个 `c_dsa` 程序全部因真实递归被当前前端明确拒绝，未进入求解阶段。
  没有采用“忽略递归后继续生成模型”的做法。

## 一个可以复查的 TF 例子

`array-fpi/eqn1.c` 的电路反例选择 `N=1`，共 84 个回放帧。
BtorSim 成功检查，YAML 包含 `\result == 1` 和 `reach_error` target。
CPAchecker 返回 `TRUE`，所以它是未确认的候选。
在该输入的有效内存执行中，源码写入 `b[0]=1`，对应断言要求 `b[0]==1`。
模型包含 `__runtime_ptr_read_default` 自由输入，后续可以沿指针读取默认分支定位
与源码语义的差异。本次尚未完成具体错误编码规则的根因证明或修复。

## 复杂程序的正例与边界

之前的独立 unsafe 样例测试中，下列反例已经通过 CPAchecker；它们不属于本次 TF 集：

| 任务 | 程序结构 | 电路回放帧数 | 结果 |
|---|---|---:|---|
| `list-properties/list-2.i` | 循环分配动态链表、指针连接和遍历 | 130 | C witness 确认 |
| `loops/matrix-2-2.c` | 二维变长数组、嵌套循环、nondet 数据 | 44 | C witness 确认 |
| `ldv-regression/recursive_list.i` | 多次分配节点、两级 next 指针访问；此例并非递归调用 | 104 | C witness 确认 |

证据见[前一批 109 个任务的报告](../2026-09-22/README.md)。
这些程序具有较复杂的数据结构，但反例仍短；预处理后的 `.i` 文件行数还包含大量声明。
不能据此宣称已经覆盖大型驱动、深递归或很长的错误路径。
当前 witness 对 scalar integer/bool nondet 调用生成约束；没有源码 nondet 调用的程序
可能仅有 target waypoint，完整电路 trace 另存。生成 YAML 不代表已经导出了全部堆约束。

## 实验配置与可复现边界

- 52 个 SV-COMP 源文件来自 `/Users/west/Developer/sv-benchmarks/c`；
  5 个 `c_dsa` 源文件来自 `artifact-for-submission/benchmarks/exp2`。
  57 个输入均与提交包对应源码逐字节一致，所有选中属性标签为 true，模型为 ILP32。
- 使用当前构建的 C2Btor 和 source map，BtorMC 3.2.1 作为首轮后端。
  对 29 个首轮没有可回放反例的任务，再用 rIC3 1.5.2 portfolio 交叉检查。
  28 个重导出的模型逐字节相同；`dll2c_prepend_equal` 的对象编号顺序发生变化，
  因此又直接用首轮原 BTOR2 复跑 rIC3，该次达到 4 GiB RSS 限制而被终止。
  额外复跑的命令、模型哈希和内存记录已保留；不把它记为求解结论。
- 转换 20 秒，BMC 界限 400，后端内部 15 秒/外层最多 20 秒；
  BtorSim 20 秒；CPAchecker 25 秒 CPU；每任务进程树 RSS 限制 4 GiB；每批最多 2 个任务。
- CPAchecker 4.2.2-1141-gd28fc1d549+，Java 21，显式 predicate witness validation，
  PRINCESS 精确 bitvector，禁用浮点，没有 integer/rational 近似。
- 本机默认 SDK 路径失效；5 个 `c_dsa` 任务显式加入已安装 SDK 的 include 路径，
  解析通过后仍因递归被拒绝。没有修改源码或全局系统设置。
- 本次未使用论文当时的工具二进制和历史 BTOR2，未宣称复现原 RQ1/RQ2 运行。
  它是对给定历史 TF 名单的当前 witness 能力审计。

## 文件

- [results.csv](results.csv)：57 个任务、用户原名称、源码哈希、候选生成状态和各次尝试。
- [results.json](results.json)：完整阶段结果、候选路径和 CPAchecker verdict。
- [selection.json](selection.json)：输入与提交包对应路径、逐字节比对结果。
- `evidence.zip`（完整本地归档，不随源码提交推送）：模型、map、backend witness、回放输出、YAML、验证日志、
  原始源码与脚本快照。ZIP CRC 检查通过；绝对路径仍记录原机器位置。
- `sv-tasks.txt` / `custom-tasks.txt` / `ric3-tasks.txt`：各批次选样清单。

使用 `scripts/run_witness_svcomp.py --expected-verdict true --tasks-file ...`
可以复测此类任务。SAFE 不生成不变量 witness。
