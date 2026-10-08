# C2Btor 浮点与内存编码专项（2026-10-03 会话记录）

记录当日问题定位、修复、进度与剩余缺陷。第 1–3 节保留此前实验背景；
第 4–7 节按 release 合并后的实际源码和本轮验证更新。配套分析见
[ieee754_kratos_analysis.md](ieee754_kratos_analysis.md)。

## 1. 起因：11 个 FT 误判（label=false 判 unsat）

集中在两个源 case（ILP32 unreach-call，label=false）：

1. `array-industry-pattern/check_removal_from_set_after_insertion.c`（7 次，
   lab2/AIGER 上 simplecar fcar/bcar/ic3 与 ric3/ic3 判 unsat）
2. `loop-floats-scientific-comp/loop2-1.c`（4 次，simplecar bcar/fcar）

## 2. Case 1 根因与修复

**根因**：`const int SIZE=100000; int set[SIZE]` 在 C 语义是 VLA，落入
"runtime-sized object capacity: 1024 字节/对象" 边界；global_bv 模型把整个
内存截断为 2×1024 字节。首个初始化循环在越界访问处触发 `__memory_halted`
冻结，`--goto-btor2-no-heap-guards` 又抹掉了 model_limit 报告 → 断言不可达
→ 求解器在截断模型上"正确"证明 UNSAT → 管线当成 SAFE。btorsim 重放证实
第 7 步 halted 置位、pc 永久冻结。

**修复**（`memory_access.{h,cpp}`）：const 传播折叠数组大小——
`collect_const_bindings()` 扫描函数内唯一常量赋值的 const 局部变量，
`fold_size_expr()` 代入大小表达式并对静态 const 全局查符号表初值；
对象注册改用折叠后大小。实测 Case 1 两数组按真实 400000 字节打包
（原 1024），btorsim 60 步 halted 不再置位。

**遗留策略问题**：真 VLA/malloc 超容量仍会静默截断；管线缺"模型含容量截断
时禁止输出 UNSAT=SAFE"的守门规则（待办）。

## 3. Case 2 根因与修复

**根因**：浮点 lowering 走 `float_utilst` + `btor2_propt` 适配器，适配器的
`set_equal()` 生成 BTOR2 `constraint`（全局恒真），电路依赖 `__floatbv_bit$N`
自由位且锁定不完整——运行期浮点除法输出 0（btorsim 重放 `pi/3`、`1.5/3.0`
均为 0），assume 守卫失败 → abort 冻结 → 断言不可达 → 假 UNSAT。
分析确认 BTOR2 标准无浮点能力，Kratos2 靠 IR 层实数语义 + 自包含位级电路
（详见 [ieee754_kratos_analysis.md](ieee754_kratos_analysis.md)）。

**修复**（本次核心改动）：

- 新增 `src/goto-btor2/ieee754_arith.{h,cpp}`：参数化 float/double 的自包含
  IEEE-754 电路（扩展定点 + 单次 RNE 舍入 + 显式特殊值分支）；
- 重写 `expr_to_btor2_float.cpp`：删除 `btor2_propt`，typecast/op/relation/
  predicate/unary 全部入口改走新电路；舍入模式显式检查（算术 RNE、向整数
  RTZ），不支持即拒绝；移除 `floatbv_bit_counter`；
- `CMakeLists.txt`/`Makefile` 注册新文件。

## 4. 本轮验证状态（release `638c1b35ef` + 未提交修复）

旧记录的“184 组全通过，因此排除加法电路”不能作为当前正确性证据：
旧 `add()` 把 `diff == 0` 同时应用于同号加法和异号减法，实际电路
`1.5f + 1.5f` 的结果为 0。保留旧源码的独立 BTOR2/BtorSim 重放已复现该错误。
问题来自加法电路，不是模型组装层。此前 14 个微用例的 10/14 结果是历史
进度，本轮采用下列可复现回归替代其验收口径。

| 验证层 | 本轮结果及范围 |
|---|---|
| 构建 | `cmake --build build --target cbmc -j8` 成功 |
| 数值电路 | 18,315 项，binary32/binary64 四则、比较、分类、一元操作、格式/整数转换及整数转换定义域；原生 IEEE RNE/RTZ oracle 对实际 BTOR2 电路重放，零不一致 |
| parser/type | 144 批数值模型、错误 oracle 对照、64 个 C 集成模型及两个符号模型均通过官方 catbtor |
| C 集成及重放 | 28 个正常 case × 32/64 目标 = 56 项；数值断言不失败，完成标记均可达 |
| 支持边界 | 4 个 case × 32/64 目标 = 8 项；运行时非 RNE、整数越界、负数转 unsigned、NaN 转整数触发 `model_limit`，完成标记不可达 |
| 格式拒绝 | long double 算术转换失败，未产出可用模型 |
| 错误对照 | 把 `+0 + +0` 的 oracle 改为最小正非正规数，重放检测到不一致 |
| backend | ric3/IC3：`1 <= a <= 2` 下 `2 <= a+a <= 4` 为 UNSAT；`a == 1.5` 下 `a+a < 3` 为 SAT，标准 BTOR2 反例经 BtorSim 检查通过 |
| 合并兼容回归 | 30 项 property/parser 回归通过 |
| 源 witness/CPAchecker | 本轮浮点修复未做 C witness 导出与 CPAchecker 确认；此前合并验证的完整 witness 回归在 ric3 堆模型 frontend 崩溃处受阻 |

数值 oracle 禁止 fast-math/FMA contraction，运行时设置 FE_TONEAREST；固定
seed `20261003`，包含正负零、NaN/Inf、最小/最大非正规数、normal 边界、
舍入 tie/sticky、上溢、渐进下溢和整数转换端点。NaN 结果只比较 NaN 分类，
其他结果逐位比较（包含零的符号）。这是规则回归，不是全输入语义等价证明；
两个 backend 结论只针对所列源程序和配置。

复现入口：

```sh
cmake --build build --target cbmc -j8
python3 regression/goto-btor2/check_float_lowering.py \
  --cbmc build/bin/cbmc \
  --catbtor 'checker/macOS arm64/catbtor' \
  --btorsim 'checker/macOS arm64/btorsim' \
  --ric3 'checker/macOS arm64/ric3' \
  --out /tmp/c2btor-float-NEW_RUN_ID
```

`--out` 必须是新的目录；平台工具路径须按实际安装替换。`--ric3` 可省略，
此时明确不执行 backend 检查。目录保存源程序、模型、oracle manifest、VCD、
标准反例、命令/输出、源码及工具哈希；`summary.json`/`integration.json`/
`backend.json` 分别给出各层结果。

本轮本地产物：
`/private/var/folders/mr/g44lm5251r39hy7cv4y8t2680000gn/T/c2btor-float-fix-a65fljy7/`，
其中 `final/` 是最终回归，`before-add.vcd` 是旧加法错误的重放，
`build-final.log` 是构建记录，`properties-final/` 是兼容回归。

## 5. 本轮修正与剩余边界

### 已修正

- 加法/减法按完整幅值排序；抵消仅作用于异号，左移规格化按实际距离减阶码。
- `shift_right_jam()` 保留移出位的 sticky；`normalize()` 共用 normal/subnormal
  输入规格化，`round_pack()` 共用一次舍入与渐进下溢，避免双重舍入。
- NaN/Inf/零分支顺序修正；正负零比较相等，零结果符号遵循所建模的 RNE 运算。
- 除法保留商的额外精度并在规格化后合并余数 sticky。
- int→fp 使用舍入返回值与进位；短整数先扩展，避免 int32→double 的切片越界。
- fp→int 按源浮点宽度解码、正确选择移位方向；NaN/Inf/越界不再变成成功的零值或回绕值。
  定义域检查复用当前指令的路径守卫及冻结机制，以 `model_limit` 单独报告。
- fp↔fp 修正有效数对齐和重偏置工作宽度，包含非正规数。
- 舍入模式读取运行期状态；静态不支持的 mode/format 拒绝转换，动态不支持的
  mode 报告模型边界。常量直接复用 `convert_constant()`，移除重复逐位打包适配。

旧记录中“ASSIGN 转换失败会静默跳过”也不符合当前代码：
`goto_btor2_instructions.cpp` 已调用 `conversion_error()` 阻止成功模型输出；
本轮未修改普通 ASSIGN 流程。

### 前一阶段待补项（接续实现见第 9 节）

1. 算术/向浮点转换仅 RNE；向整数转换 RTZ。IEEE remainder、sqrt/FMA、其他
   舍入模式及 binary32/binary64 之外的算术格式未覆盖。
2. 算术 NaN 采用 canonical quiet NaN；不承诺 payload/sign 传播，也未建模
   signaling NaN、浮点异常标志/陷阱或完整 fenv。
3. 尚无全输入电路等价证明和本轮浮点 C witness/CPAchecker 验证；原大型
   loop/array benchmark 未在本轮重跑，历史结果不算当前版本通过。
4. `--goto-btor2-no-heap-guards` 会隐藏辅助边界 bad，冻结仍存在；本轮新增
   浮点 `model_limit` 也受这个开关影响。选择源属性或隐藏辅助属性后，不能
   仅凭 UNSAT 宣称源程序 SAFE；实验管线的统一边界守门规则仍待实现。

转换定义域遵循 C11 草案 §6.3.1.4：有限浮点值转整数先向零截断，截断后的
整数须可表示；例如 `-0.75f → unsigned` 的结果是 0。
参考 [WG14 N1570](https://www.open-std.org/jtc1/sc22/wg14/www/docs/n1570.pdf)。
IEEE 舍入模式与异常状态的区分参考
[Berkeley SoftFloat 文档](https://www.jhauser.us/arithmetic/SoftFloat-3/doc/SoftFloat.html)；
本实现未引入 SoftFloat 依赖，也不把原生 oracle 对拍当成 SoftFloat 差分验证。

## 6. 最新运行参数

基本命令（基准口径，参数**仅支持空格分隔**，`--memory=object` 会报
Unknown option；`--memory/--array*` 必须与 `--goto-btor2` 同用）：

```sh
build/bin/cbmc [--32|--64] --inline --goto-btor2 \
  --memory global|object --array bv|array \
  [--memory-object-max-bytes n] [--array-bv-max-object-bytes n] \
  --no-standard-checks --no-pointer-check --no-bounds-check --no-built-in-assertions \
  [--goto-btor2-error-function reach_error] \
  --goto-btor2-out model.btor2 \
  [--goto-btor2-map-out model.map.json] \
  program.c
```

合并后 release 已包含 `--goto-btor2-no-heap-guards`、
`--goto-btor2-merge-properties`，并保留本地 `--goto-btor2-reach-only`、
`--goto-btor2-merge-bads`；后两个 merge 开关按当前统一实现合并所选属性。
默认保留 `memory_validity`/`model_limit` 辅助 bad；按 test/AGENTS.md 的
结果解释规则单独分类，不得并入 unreach-call 结论。

浮点修复完成时的旧默认值为 `--memory global --array array`、运行时容量 1024 字节/对象。
后续按用户要求已改为 `--memory object --array bv`，容量默认自动推导；
见[自动容量](c2btor-memory-capacity.md)。上述旧实验保持原配置，不能当作新编码结果。
新电路后浮点模型特征：无 `__floatbv_bit$N` 自由位、无
`floatbv_bitblast_equal` constraint，含 `add/mul/udiv` 位级算术节点。

## 7. 本轮浮点变更文件

- `src/goto-btor2/ieee754_arith.{h,cpp}`：共用规格化、jam、舍入/打包及各运算修复。
- `src/goto-btor2/expr_to_btor2_float.cpp`：格式/舍入契约、源宽度与整数定义域检查。
- `src/goto-btor2/expr_to_btor2.h`：入口声明与电路职责注释。
- `regression/goto-btor2/float_circuits.cpp`、`check_float_lowering.py`：可复现分层回归。
- 本文及 `docs/ieee754_kratos_analysis.md`：更新已证实根因、证据和支持边界。

本节验证时仍在 release 工作区，保留合并前本地修改和 stash 备份；当时未提交、推送或切换
main/dev。产品重命名与上游同步方案属于后续单独迭代。

## 8. 默认编码与自动容量迭代

按后续用户要求，默认值已改为 `--memory object --array bv`，运行时对象容量从固定
1024 字节改为最终 GotoIR 区间分析得到的上界，并受指针偏移及 BTOR2 位宽限制。
容量在模型生成时确定；运行期 BV 不扩容，也没有新增 assume 来缩小 C 输入范围。
`n=10000; int a[n]` 与 `int a[10000]` 均保留 40000 字节，实际大小仍在声明时锁存。

最后审查补上 C `_Bool` 的表示范围，并修复标量转布尔时直接截断位宽的问题。
`(_Bool)2` 等非零转换现在生成正规化的 1；该问题的回归覆盖两种目标机器。

当前 binary 的容量回归共 54 项、76 次具体输入回放，parser/BtorSim 通过；
源属性兼容回归 30 项通过。IEEE 电路源码未再改变，18315 项数值对拍保持 0 mismatch；
当前 binary 重新生成和检查的 64 项浮点 C 集成也通过，包含正常完成及预期模型边界。
完整结果、哈希、命令与未完成检查见[验证记录](memory-capacity-validation/2026-10-03/README.md)。

较早容量快照的部分堆 rIC3 检查及完整 witness 回归仍超时，不能算通过。
尚无本轮 CPAchecker 确认或全输入语义证明；隐藏 `model_limit` 后的 UNSAT→SAFE
管线问题在该容量阶段尚未修复；后续实现和验证见第 9 节。

## 9. 接续实现：多舍入、数学函数与 UNSAT 守门

接续代码已实现 binary32/binary64 的五种 CBMC IEEE 舍入模式，sqrt、单次舍入
FMA、IEEE remainder 和 fmod。共用规格化/舍入保持在 `ieee754_arith`，数学电路放在
`ieee754_math.cpp`；库函数 lowering 在 inline 前完成，保留同名用户函数及普通 symex。
整数转换仍向零截断；非法动态模式和越界整数转换仍是独立模型边界。

模型及 map 始终保存 `proof_obligations`。维护的批量工具在解释 UNSAT 前检查所有
源属性及边界；旧模型没有契约时保留未验证结果，BMC 无反例保持有界结果。隐藏容量、
非法访问和非法浮点模式的实际求解及完整 runner 都得到正确边界分类；零 bad 与多 bad
的单一 OR 查询也已验证。

Object/BV 标量访问改成每对象一次 word 访问；字节复制改成原子区间更新，同时传播
数据和初始化标记，保持别名/重叠/生命周期/区间外内容。容量分析补齐不可达声明、
常量布局偏移与单值机器算术，修复零长度 memmove 的库内对象被放大到表示上限的问题。

浮点 42539 项数值对照、104 项 C/parser/BtorSim 集成及两项符号 backend 对照通过。
标准 witness 回归 33 项通过，两节点链表 witness 已由 CPAchecker 得到 FALSE/confirmed。
部分堆查询在实验预算内超时，保留 TIMEOUT；这是资源结果，不作为实现未修复的证据。
模型缩小不能替代求解结论。完整逐层证据、命令、哈希及支持边界见
[接续验证记录](float-validation/2026-10-03/README.md)。验证时工作区仍在 release，尚未提交或推送。
