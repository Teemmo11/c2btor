# 对象与字节堆模型：实现及验证记录

日期：2026-09-23。对应 [实施方案](../../c2btor-heap-implementation-plan.md)。
这是新的基准实现及支持范围记录，不是完整 C 内存语义的正确性证明。

## 本次实际修改

- `heap_model.h/.cpp`：对象号、字节偏移、对象大小与 live 状态；一份权威字节存储；
  written / defined / zeroed 元数据；分配、释放、字节复制和独立边界属性。
- `memory_access.h/.cpp`：CBMC 目标字节宽度、字节序、成员布局和对齐；结构体、
  数组、成员数组、多级指针、取地址标量共用地址与读写规则。C 全局对象和字符串
  初始化也进入这一存储。字符串写入触发内存有效性属性。
- `goto_btor2_collect.cpp` / `goto_btor2_assign.cpp`：删除按第一次指针类型猜测 heap
  布局、根数组和独立字段状态并存的路径。只有不取地址的标量保留独立状态。
  CBMC 溢出计算的 value / overflow 逻辑临时值单独保存，不冒充 C 内存对象。
- `lower_heap_operations.h/.cpp`：内联前保留来自 CBMC 库的释放事件，以及
  memcpy / memmove 的边界和重叠条件；用户自定义同名函数按其真实函数体翻译。
- `lower_array_operations.cpp`：复制参数和长度锁存；使用源快照及字节复制步骤。
  初始化元数据随字节复制，允许复制尚未初始化的 padding；memmove 保持重叠
  情况下的源内容。realloc 复用库的 allocate → 公共前缀 copy → free 控制流。
- `c2btor_witness.py`：区分源属性、memory_validity、model_limit；后两者不导出
  unreach-call witness。BtorMC 完整轨迹在第 0 帧提供的已初始化数组行可省略，
  由 BtorSim 根据模型 init 计算；保留原轨迹、未初始化数组、所有输入及后续帧。
- `run_witness_svcomp.py` 保留上述结果分类。旧 `test/tools/run_ric3.py` 不请求轨迹，
  因而对带辅助内存属性的模型，不能仅凭 SAT 标成 TF / FF，改记 unknown。

## 规则与更新关系

| 操作 | 已实现的更新与保持关系 |
|---|---|
| malloc | 成功分支锁存 size；每次执行使用新对象号；已有对象内容、大小不变；失败分支来自 CBMC 库 |
| calloc | 同一分配规则，加全零字节默认值；乘法溢出使用 CBMC value / overflow pair |
| 字段、元素、别名 | 按对象号和偏移读取同一份存储；写入仅改变相应字节和 defined 标记 |
| 指针差 | 同一活对象、合法范围内的字节差除以元素跨度；检查结果位宽 |
| free | NULL 不操作；否则要求活的动态对象首地址；live 置假，指针变量本身不置零 |
| memcpy / memmove | 检查范围，memcpy 另外检查不重叠；快照复制字节和初始化标记 |
| realloc | 失败保留原对象；成功复制 min(old_size,new_size) 前缀后释放原对象；扩展区不设为零 |
| 无效访问 / 模型超限 | 独立 bad，冻结后续普通执行，禁止同一路径继续触发源属性 |

复制未定义字节时，不将其标记成已定义；复制到 calloc 对象也不能因目标原先清零
而掩盖源字节未初始化。类型化读取未确定字节目前触发 model_limit，避免武断地
将所有这类 C 行为判为 UB，或产生可以冒充真实反例的固定零值。

## 明确的范围与边界

1. `--goto-btor2-heap-objects K` 默认为 32，含义是总成功分配次数。释放不归还身份。
   第 K+1 次成功分配触发 model_limit，不能解释为 malloc 返回 NULL 或程序 UNSAFE。
   对象号、偏移可表示性也有容量检查。映射文件记录预算、分配位置、大小节点等。
2. malloc 失败策略保留 CBMC 前端配置。仓库默认 `--no-standard-checks` 不等于
   启用分配失败；要包含返回 NULL 分支，需要 `--malloc-may-fail --malloc-fail-null`。
3. 同一取地址的自动对象声明重复执行时，目前触发 model_limit，避免旧生命周期的
   指针别名和旧内容被静默复用。复制 lowering 的局部快照也受此边界约束。
4. 大小不定或超过 4096 个元素的整体数组赋值需要另行 lowering；当前明确拒绝。
   位域内存访问、不能确定的布局、普通指针与整数之间的地址表示转换明确拒绝。
5. 根对象大小、存活状态及解引用对齐已检查；完整子对象 provenance、effective type、
   trap representation、所有 union 重解释规则尚未完成。根对象范围检查不能称为
   完整 C UB 检查。没有以本次回归代替模拟关系证明。
6. 没有实现递归调用栈、并发或无界堆抽象。零大小分配及 realloc(p,0) 沿用当前
   CBMC 库的实现选择。SAFE 不生成不变量 witness。

## 实际验证及局限

- 最终 `cmake --build build --target cbmc -j8` 成功；BTOR2 官方 parser 检查通过。
- `heap_rules.c` 的 25 类规则场景，加 16 个终点可达性对照，共 41 项：40 项满足
  预期，1 项求解超时。一般 BMC 界为 400，结构体 padding 复制的终点对照使用 1000。
  正例的“界内未发现违反”和到达终点的对照分别记录，均不称为无界 SAFE。
- 唯一未完成项是 CASE=16：允许 malloc 失败的 realloc 场景，在 400 步、60 秒
  求解限制下超时；该例的终点可达性对照成功。保留超时，不改报通过。
- 最终核心修改另复查循环分配、free 后读取、realloc 前缀、容量和真实源错误，7/7
  项满足预期。既有 witness 回归及新增链表反例共 33 项通过。
- [heap_linked_witness.c](../../../regression/goto-btor2/heap_linked_witness.c) 的两个
  动态节点与 next 指针反例已通过 BtorSim → YAML 2.0 → CPAchecker/PRINCESS
  验证，CPAchecker 返回 FALSE，即确认 witness 中的违例执行。使用位向量编码，
  关闭浮点编码。证据见 `linked-witness.yml` 和 `linked-validation.json`。
- 同一程序的 USE_AFTER_FREE 和 K=1 变体分别归为 memory_validity / model_limit，
  未生成 C violation witness。
- 用户给出的 57 个历史 TF：52 个转换且 parser 通过；5 个 c_dsa 因真实递归
  明确拒绝。这里只复核转换覆盖，未重新求解全部任务，也不说明 TF 已全部消除。

`rule-results.json`、`tf-conversion-results.json` 保存逐项结论、边界和产物路径。
完整运行产物保留在 `/tmp/c2btor-heap-rules-run3`、`/tmp/c2btor-heap-rules-run4`、
`/tmp/c2btor-heap-rules-final`、`/tmp/c2btor-tf-memory-final`、
`/tmp/c2btor-memory-witness-final`；旧实验目录未覆盖。

## 复现

```sh
cmake --build build --target cbmc -j8
python3 regression/goto-btor2/check_heap_rules.py \
  --btormc /path/to/btormc --catbtor /path/to/catbtor \
  --output /tmp/new-heap-rule-run --bound 400 --timeout 60
```

macOS SDK 配置过期时可额外传 `--include-dir` 指向实际 SDK 的 `usr/include`。
脚本复制当前 CBMC 二进制到独立输出目录，避免编译与运行交叉时修改被执行文件。
规则集出现超时返回非零，并在结果 JSON 中保留 incomplete；不为了绿色结果跳过它。
