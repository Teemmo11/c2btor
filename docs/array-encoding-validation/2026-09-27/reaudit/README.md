# 数组双编码复查（2026-09-27）

本轮重新检查统一字节内存、packed storage 读写、对象映射、初始化元数据、
复制、容量检查及故障分类，并使用当前二进制复测。未修改生产编码逻辑。

**结论：已检查的容量范围内常见操作未发现 Array Sort/BV 行为不一致；
不能宣布全部数组操作正确或历史 wrong results 已结案。**
新发现 ILP32 下前端先窄化超宽下标导致两种编码共同漏报；分配失败分支的
realloc 安全检查在两种编码下均超时。失败原始结果保留，不改记为通过。

## 位宽的准确表述

`int a[N]` 在 ILP32/LP64 下均有 32 位元素，数组内容占 32N 位，即 BV(32N)。
抽象上也可以视为 N 个 BV32。当前实现将数组放入统一字节存储，因此并非每个
C 数组独立生成一个 BV 状态节点；物理拼接不改变元素类型和读取结果的宽度。
字节顺序由目标端序确定。本文的容量按目标 C 字节计，常用配置下每字节 8 位。

## 本轮实际检查

| 检查 | 结果 | 解释 |
| --- | --- | --- |
| 64 位，原 16 类数组规则 | 56/56 | 含正常、终点和故意错误对照 |
| 32 位，完整 16 类数组规则 | 54/56 | CASE=11 在两种编码均漏报，不是超时 |
| 64 位大端，元素类型/矩阵/指针数组/结构体/字节别名 | 20/20 | CASE=3,4,5,6,16 |
| 16 类堆/复制/生命周期规则，Array Sort | 24/25 | CASE=16 的无违反检查超时 |
| 同一批规则，BV | 24/25 | 同一 CASE=16 超时 |
| 新增符号写入、frame、二维 VLA、插删、未初始化复制、条件读、只读对象 | 16/20 | 条件式中的直接数组退化在前端被拒绝，4 项未进入模型检查 |
| 条件读改用显式指针表达式 | 4/4 | 不覆盖原始前端拒绝结论 |
| 二维跨行访问，开启 bounds instrumentation | 两种编码均检出 | property_class 为 array bounds |
| 超宽下标，开启 conversion instrumentation | 两种编码均检出 | property_class 为 overflow；不是默认路径修复 |
| 数学 read/store 关系的 24 组有限位宽 SMT 检查 | 全部 UNSAT | 独立公式检查，不是 C++ 实现证明 |

这些是检查次数，不是独立 benchmark 数量。正常数组规则 BMC 界限 250，复制
用例 600；扩展规则 400，堆 padding 复制 1000；每次求解 45 秒。
正例的 bounded_no_violation 不作为无界 SAFE；终点对照只证明终点可达。
本轮成功转换的回归模型经过 catbtor parser；数组差分脚本另外检查 BV 输出
无 sort array/read/write。堆脚本不重复执行该结构断言。

## 逐类操作的审查结论

| 操作 | 代码与验证 | 保留边界 |
| --- | --- | --- |
| 读写、查询、符号下标 | packed shift/extract、mask update；已有 CASE=1 与新增任意 64 位数据、i/j 下标测试 | 前端宽下标窄化缺口见下 |
| 未修改元素及其他对象保持 | 新增源断言独立检查 read-after-write 和 frame；SMT 检查数学读写关系 | 不是任意程序的整体证明 |
| 初始化 | 静态、零、部分、指定、字符串初始化；未初始化类型读报告 model_limit | 没有把未定义字节改成零 |
| 多维数组 | 固定矩阵、行指针、二维 VLA；改变原长度变量后检查锁存的大小与跨度 | 逐维边界依赖对应 instrumentation |
| 元素类型 | signed/unsigned、short/int/long long、_Bool、float、指针、结构体代表用例 | 内存位域明确拒绝；并非所有 C 类型 |
| 别名 | 直接数组/元素指针/指针数组/结构体字段/unsigned char 视图共用对象字节 | 完整 provenance/effective type 不在已证明范围 |
| 赋值和复制 | 含数组结构体赋值、memcpy、memmove、memset、realloc 前缀；未定义源复制到 calloc 目标仍为未定义 | realloc 允许失败的无违反检查超时 |
| 插入删除 | C 数组本身不扩容；新增通过移动元素实现的插入删除检查通过 | 动态扩容另属 realloc 语义 |
| 大小与容量 | 实际大小独立于预留容量；本轮 VLA/分配超限仍为 model_limit | 默认 1024 字节测试见同目录上层 capacity-default.md |
| 生命周期和错误分类 | free(NULL)、free 后访问、零大小分配、预算耗尽、源错误、位域拒绝均符合对应预期 | 递归、重复自动对象生命周期等已有边界仍存在 |

## O1：ILP32 超宽下标在前端被窄化——未解决

输入：

```c
int a[2] = {1, 2};
unsigned long long i = 1ULL << 32;
a[i] = 9;
```

当前前端将最后一行转换成：

```text
ASSIGN a[cast(i, signedbv[32])] := 9
```

转换结果的下标为零，Array Sort 和 BV 都没有报告原始超大下标；本轮两项
预期 model_limit 的检查实际得到 bounded_no_violation。旧的 32 位代表子集
未包含 CASE=11，因此旧 29/29 不能覆盖这个缺口。

责任位置：`src/ansi-c/c_typecheck_expr.cpp:1324` 的 make_index_type 使用
c_index_type；后端 `memory_access.cpp:250` 接收到的是已窄化的表达式。
packed storage 对它接收到的完整下标进行检查，无法自动恢复此前丢失的信息。

启用 `--goto-btor2-checks --conversion-check` 并保留 built-in assertions
可检出此探针，但默认检查关闭时问题仍在。不能把这个额外 overflow 属性当作
unreach-call 的源反例，也不能声称打开它就完成了 C 下标语义证明。

本轮没有靠删除所有整数 cast 或把窄化全部判为源错误来修补：源码显式整数
转换有自己的语义；前端插入的下标归一化需要可识别的来源及相应保持条件。
该缺口须单独修复并加入“显式转换合法 / 隐式窄化丢信息”的成对回归。

## O2：含分配失败分支的 realloc——未确认

`heap_rules.c CASE=16` 在 Array Sort/BV 下都达到 45 秒限制，未得到 400 步内
全部属性无违反的结论。终点可达性对照通过，不能代替安全检查；也不能由超时
断定实现有错。普通 realloc 前缀复制测试通过。

## O3：条件表达式中的数组退化——前端拒绝

合法 C 形式 `choose ? a : 0` 在当前前端报 array 与 int 条件操作类型错误，
两种编码都没有输出成功模型。显式指针形式 `choose ? &a[0] : (int *)0`
的正常与终点测试均通过，确认已支持路径中的条件读取不会执行未选分支。
保留 extra-original.c 与初次结果，不把修改后的例子当作原表达式已经支持。

## 历史 wrong results 的状态

仓库此前对 57 个历史 TF 的记录不是逐项根因闭环：2026-09-23 记录了 52 个
转换/parser 成功、5 个递归拒绝，并明确未重新求解全部任务。
2026-09-22 witness 审查记录 23 个可回放候选、0 个获 CPAchecker 确认；
候选未获确认同样不能反推 SAFE。这些是历史记录，本轮未重新运行 57 个任务。

本轮能确认读写、别名、容量及分类的具体证据，不能把这些结果写成
“历史 wrong results 已全部定位或修复”。需要另外保留逐任务：原始输入与版本、
错误属性及 trace、根因规则、最小复现、修复后结果和 witness 确认的审查表。

## 表示关系与证据

见 [representation.md](representation.md)。其结论是容量条件下的存储表示
保持论证草案，不是完整 C → GOTO-IR → BTOR2 soundness 定理。

[summary.json](summary.json) 保存当前二进制哈希、各批结果路径及未通过项；
本目录还保存各批结果 JSON、补充 C 源码和数学检查脚本。原始模型/命令/日志位于
`/tmp/c2btor-array-reaudit-20260927*` 各目录。源码和原始结果未覆盖。

`run_extra.py` 使用 extra.c；若复现首次条件式拒绝，使用 extra-original.c 内容。
正常回归脚本位于 `regression/goto-btor2/check_array_encoding.py` 与
`check_heap_rules.py`，完整参数保存在每个原始用例目录的 command.json 中。
