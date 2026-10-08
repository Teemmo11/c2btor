# 默认编码与自动容量

默认转换使用 `--memory object --array bv`。省略这两个开关即选择每对象独立存储和纯位向量编码。
`--memory global`、`--array array` 仍可显式选择。普通 CBMC symex 不受这些转换选项影响。

运行时对象不再默认保留 1024 字节。自动容量在**模型生成时**确定，BTOR2 的 BV sort 位宽
在求解过程中固定，不能运行到一半扩容。对象实际大小仍在声明/成功分配时锁存；容量只是
backing storage 的大小，不改变 `sizeof`、指针偏移、别名、初始化、生命周期或访问有效性规则。

## 容量分析

`memory_capacity.{h,cpp}` 对最终内联和内存 lowering 后的 GotoIR 做前向整数区间分析；
`heap_modelt` 根据结果规划存储，object/global 两种存储实现使用同一规划结果。

- 固定大小对象使用准确大小，即使显式容量参数比它小。
- 每个 VLA/运行时快照使用其声明处可达状态下的大小上界，各对象分别计算。
  在区间分析的保守 CFG 中完全不可达的声明分配零字节容量，避免把零长度复制
  的库内临时数组当作未知大小。常量下标/字段偏移及指针加减使用 CBMC 目标布局。
- 动态分配槽使用所有可达 allocate 请求大小的最大上界；各次分配仍锁存自己的实际大小，
  每次成功分配获得 fresh 身份。成功分配次数预算仍独立，默认 32，free 不回收身份。
- 赋值、结构体字段、分支、已有 assume、整数运算和类型转换可给出较小上界。
  overflow-result 的 value 字段按对应机器算术分析，单值 tuple 用 CBMC 规则折叠。
  指针赋值/复制/转换锁存对象大小与可知的偏移区间，支持复制快照所用 OBJECT_SIZE；
  未知指针或未知偏移仍保守放宽。没有添加新的 assume，
  不把 assert 当作前提。控制流汇合取区间并集，循环回边扩展时 widening 到机器类型范围。
- 可能溢出或截断的非单值整数运算取完整机器类型范围，不能沿用数学整数上界。
  单值转换使用 CBMC 的机器宽度简化规则。未分析的表达式也取完整类型范围。
  未知 C `_Bool` 内存表示覆盖目标机器位宽；正常标量转 `_Bool` 按是否为零生成 0/1，
  不能用整数低位截断替代这一转换。
- 通过指针写入和未知内存操作会丢弃取地址变量、全局变量及聚合内存对象字段的旧区间，防止别名写入后沿用
  过小容量。未取地址的局部标量可保留其区间。

例如，32 位 int、8 位 byte 的目标下：

| 程序中的大小 | 自动存储容量 |
| --- | --- |
| `int a[10000]` | 40000 字节 |
| `unsigned n=10000; int a[n]` | 40000 字节 |
| nondet n，已有 `assume(2<=n && n<=10000)`，`int a[n]` | 40000 字节 |
| 分支令 n 为 10000 或 20000，`int a[n]` | 80000 字节 |
| `malloc(5000)` | 每个分配槽 5000 字节 |

区间分析有意允许精度损失。例如取地址 unsigned short n 经别名修改后，分析可能需要
覆盖 n 的完整类型范围，而不是直接识别后一次写入的常量。因此自动模型不总是最小模型。

## 表示与资源边界

不能推导较小上界时，容量覆盖到大小类型、指针偏移与本实现 BTOR2 位宽允许的范围，
而不是回退到 1024 字节。自动容量取推导上界与表示上限的较小值，typed cell 也须整体容纳。

- 对象大小必须小于 `2^(offset_bits-1)`；offset_bits = pointer_width - object_bits。
- 本实现所用 BTOR2 parser 的 sort width 为 signed 32-bit；数据 BV 和定义性元数据的
  位宽都必须可表示。global+BV 的**总** packed storage 还必须满足这个限制。
- 8 位 byte、默认 8 位 object id 下，ILP32 的 byte 对象上限为 8388607 字节，int VLA
  最大完整容量为 8388604 字节；LP64 的 byte BV 上限为 268435455 字节，int VLA
  为 268435452 字节。这些是表示上限，不是已测试规模或实用求解能力保证。
- 大 BV 会增加 parser、模拟和求解资源消耗。自动容量没有另设固定资源截断；实际可用
  规模取决于内存、时间、对象数量、访问表达式和 backend。超时/内存超限不能当作 SAFE。
  Global+Array 不需要该有限存储规划，但仍保留指针、分配次数及其他模型边界。

要主动控制资源，可显式设置运行时容量：

```sh
build/bin/c2btor program.c --goto-btor2 --inline \
  --memory-object-max-bytes 40000 --goto-btor2-heap-objects 8
```

`--array-bv-max-object-bytes N` 在 BV 模式设置同一限制；object+BV 同时传入两个容量参数时
数值必须相同。显式容量为正整数。超过容量触发 `model_limit` 并冻结普通执行，不能截断对象、
新增 assume 或冒充 malloc 返回 NULL；访问超过实际对象大小仍为 `memory_validity`。

隐藏内部边界属性后，只对源 bad 得到 UNSAT 不能直接解释成源程序 SAFE：冻结路径可能
隐藏后续违例。现在模型注释及 map 始终保留 `proof_obligations`；维护的 rIC3/Pono
批量工具检查包含所有源属性和这些边界的单一 OR 查询，witness runner 在需要时补查
边界。旧模型无证明契约、BMC 有界无反例、边界可达分别保留为未验证、有界结果和
独立边界分类。外部自行选择源属性的命令仍须遵守此契约，不能只看原始 UNSAT。
未确定字节读取、重复自动对象生命周期和成功分配次数等原有模型边界仍保留。

有限存储模式的 map 中 `max_object_bytes: null` 表示自动模式，显式模式记录整数；`capacity_policy`、
`representation_capacity_bytes`、`allocation_capacity_bytes` 和每对象 `capacity_bytes`
记录实际规划。global+array 的 policy 为 `pointer-addressed`，不记录有限存储容量字段。存储布局、source map 及 witness 的对象身份规则保持原有契约。

## 验证

`regression/goto-btor2/check_auto_capacity.py` 覆盖 ILP32/LP64、默认模式、四种编码组合、
大 VLA、独立对象、assume/分支/mask、布尔转换、整数 narrowing/wrap、别名写入（含未显式取地址的聚合根）、循环 widening、
大 malloc/calloc、calloc 溢出、realloc 前缀、锁存大小、显式容量边界与实际越界。它检查容量元数据、官方 parser，再用标准 BTOR2
输入轨迹经 BtorSim 回放；正常路径必须到达专门的完成断言，不能把未运行到终点当作通过。
这是具体执行回归，不是区间分析的形式化健全性证明，也不是完整 C 语义保持证明。

```sh
capacity_run=$(mktemp -d /tmp/c2btor-capacity.XXXXXX)
python3 regression/goto-btor2/check_auto_capacity.py --output "$capacity_run/checks"
```

原有 `check_array_capacity.py` 已更新自动容量预期；`check_object_memory.py` 的 1025 字节
边界用例改为显式设置 1024，以继续验证容量超限的独立分类。

本轮结果与未完成检查见[验证记录](memory-capacity-validation/2026-10-03/README.md)。
