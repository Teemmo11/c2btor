# 自动容量验证记录（2026-10-03）

本页保留自动容量阶段的历史快照；后续多舍入/数学函数、UNSAT 守门、区间复制和
CPAchecker 确认的当前结果见[接续记录](../../float-validation/2026-10-03/README.md)。

本轮在 release 工作区实现默认 `--memory object --array bv` 与自动容量规划。
验证基线 HEAD 为 `638c1b35ef8eea39de917e2b56b1b7429839ae1a`，当时代码尚未提交；保留已有修改和 stash。
容量规划与运行时内存更新分开，规则和表示边界见[自动容量说明](../../c2btor-memory-capacity.md)。

| 检查 | 结果与范围 |
| --- | --- |
| 构建与 CLI | cbmc 构建成功；help 显示 object、bv、automatic 默认值 |
| 自动容量 | ILP32/LP64 共 54 项，官方 catbtor 检查及 76 次具体输入 BtorSim 回放通过 |
| 大数组 | 固定 `int a[10000]` 与运行时 `n=10000; int a[n]` 保留 40000 字节；VLA 覆盖四种编码组合 |
| 类型与控制流 | assume、分支、mask、narrowing、wrap、循环 widening、字段/别名写入、大小锁存、布尔转换 |
| 分配与边界 | malloc/calloc/realloc、calloc 溢出、显式容量超限、实际对象越界；辅助属性独立分类 |
| 源属性兼容 | 30 项属性回归通过，包含空源 bad、多 bad、属性合并、选择与 unsupported 拒绝 |
| 浮点 | 未改变的 IEEE 电路共 18315 项原生 oracle 对拍，0 mismatch；当前 binary 的 64 项 C 集成/parser/BtorSim 通过，56 项正常完成、8 项模型边界 |

容量回归的正常路径必须到达专门的完成断言，并且不能先触发数值或辅助 bad。
故障路径只允许到达预期的 `model_limit` 或 `memory_validity` 类别。
整数转 `_Bool` 的回归覆盖 0、2、256、UINT_MAX，验证非零值正规化为 1，
避免位宽截断留下错误布尔值并造成容量分析与实际执行不一致。

自动容量回归的成功分配预算为 1，realloc 用例为 2；产品默认预算仍为 32。
基本 instrumentation 配置为 `--inline --no-standard-checks --no-pointer-check
--no-bounds-check --no-built-in-assertions`。完整命令、工具/源码哈希和逐例结果见
[summary.json](summary.json)。浮点电路哈希与对拍运行一致；64 项当前 C 模型也与该运行逐字节相同。

本地完整产物根目录：`/tmp/c2btor-auto-capacity-eyjas53q`。
其中 `capacity-final-3/`、`float-current/`、`properties-final-2/` 为当前 binary 的结果；
`float-final/` 保留数值对拍及两个符号加法 backend 对照（UNSAT/SAT，SAT 已 BtorSim 回放）。
生成模型、二进制、轨迹和大日志未加入源码目录。

## 尚未完成的验证

较早容量快照的 9 项堆 rIC3 检查中 4 项通过，5 项超时/未完成；ILP32、预算 2（case 17 为 1）、
BMC bound 160、单项 60 秒。这些超时不能记为通过或 SAFE。
另有 6 项堆具体执行/parser/BtorSim 对照及 2 项 export→BtorSim→C violation YAML 候选导出通过，
但候选使用选定的输入轨迹，未由 CPAchecker 确认。完整 witness 回归在 heap-linked 求解处超时，
因此没有完成整套回归。

这些观察不构成区间分析健全性证明、完整 C 语义保持证明或表示上限处的实用求解能力保证。
未知/未确定字节、重复自动对象生命周期、成功分配预算、指针/格式位宽等原有模型边界仍在。
隐藏 `model_limit` 后把源 UNSAT 直接判作 SAFE 的外部管线问题尚未修复。

复现当前容量检查：

```sh
cmake --build build --target cbmc -j8
capacity_run=$(mktemp -d /tmp/c2btor-capacity.XXXXXX)
python3 regression/goto-btor2/check_auto_capacity.py --output "$capacity_run/checks"
```
