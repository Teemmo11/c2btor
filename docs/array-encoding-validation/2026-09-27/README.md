# 数组编码验证记录（2026-09-27）

> 本记录保存最初版本的验证结果。其中“未提供容量则拒绝”已由后续的
> 1024 字节默认容量替代；原始结果不作改写。后续检查见 [默认容量验证](capacity-default.md)。

已实现 `--array array`（默认）、`--array bv` 和显式运行时对象容量
`--array-bv-max-object-bytes N`。BV 输出连内部元数据也不含 Array Sort。
完整接口、表示关系、数组特性和限制见 [使用说明](../../c2btor-array-encoding.md)。

| 检查 | 结果 | 证据 |
| --- | --- | --- |
| 64 位数组语义及正负对照 | 56/56 | [结果](rules-64.json) |
| 32 位代表用例 | 29/29 | [结果](rules-32.json) |
| 大端字节别名 | 4/4 | [结果](rules-big-endian.json) |
| BV calloc、释放后访问、memcpy、realloc、分配次数限制 | 8/8 | [结果](heap-bv.json) |
| 既有 witness 回归 | 33/33 | [结果](witness-regression.json) |
| 默认模式、参数拒绝、缺失容量、rIC3 | 11/11 | [结果](cli-backend.json) |
| BV source witness 回放/导出与 model_limit 拒绝导出 | 2/2 | [结果](bv-witness.json) |
| 最终容量检查针对性复测 | 14/14 | [结果](final-capacity.json) |
| 极大容量拒绝与二维子数组越界 instrumentation | 3/3 | [结果](final-boundaries-corrected.json) |
| 最终二进制生成模型一致性 | 89/89，逐字节相同 | [结果](final-model-consistency.json) |

这些计数是检查次数，不是独立 benchmark 数量。数组测试采用同一 C 程序、
目标机器与检查选项，分别生成两种表示；正例同时检查终点可达性，避免空路径通过。
每个成功模型都通过官方 catbtor parser/type 检查，BV 模式额外检查不存在
`sort array`、`read`、`write`。

数组规则测试的 BMC 界限通常为 250 步，复制用例为 600 步；结果明确记为
`bounded_no_violation`。旧堆规则中的 memcpy 用例使用至少 1000 步。
这些结果不替代无界安全证明或完整语义保持定理。

另对两个小模型，把所有原始 bad 条件合成一个析取属性，再交给本地 rIC3-IC3：
正确版本得到 UNSAT，故意修改断言的版本得到 SAT。合成只改变属性的组织方式，
不改变状态、转移、输入或约束。这样不依赖 backend 默认选择哪个 bad 属性。
BV 源反例通过 BtorSim 回放并导出 violation witness；本轮未运行 CPAchecker。

对 VLA/分配容量超限，单独确认报告 `model_limit`，不输出源错误 witness。
二维数组跨行访问使用保留的 `array bounds` 断言，两种编码均能检出。
初次探查携带 `--no-built-in-assertions` 时，该参数移除了自动生成的边界断言；
去掉它后复查通过。这是检查配置差异，不能把根对象边界当作完整的 C 子数组边界。

最终构建成功，`git diff --check` 通过。构建二进制、相关源码和工具哈希见
[汇总](summary.json) 及各 manifest。完整命令、BTOR2、source map 和日志保存在
`/tmp/c2btor-array-encoding-20260927/`，其中最终一致性复核目录为
`final-model-consistency-repo-env/`。按仓库根目录运行使用说明中的回归命令即可重建。
从另一项目执行环境复核时曾遇到 macOS SDK 路径导致的 `string.h` 缺失；
回到本次测试使用的 CBMC 项目执行环境后，最终 89 个模型全部与已求解模型逐字节一致。

尚未进行全量 SV-COMP、性能消融或通用语义保持证明。相关实现仍受使用说明列出的
对象生命周期、初始化和 C 内存模型边界约束。
