# Kratos2 IEEE-754 到 BTOR2 的表示方案分析（2026-10-03）

本文记录对 Kratos2（https://kratos.fbk.eu/）浮点到 BTOR2 翻译方案的逆向分析，
以及 C2Btor 借鉴该方案重写浮点 lowering 的设计。实验产物与 14 个微用例见
`docs/2026-10-03-c2btor-float-session.md`。第 1–4 节是原会话的实验记录，
本轮未重复执行 Kratos2；第 5–6 节按本轮 C2Btor 源码和回归更新。

## 1. 实验方法

- Kratos2 v2.2/2.3 Linux x64 发行版；C 前端 `c2kratos.py`（依赖 pycparser）；
  翻译引擎 `kratos -stage=trans -trans_output_format=btor`。
- 对照生成器：C2Btor（本仓库 release 系代码）。
- 微用例：float/double 四则、一元负号、6 种比较、fp↔int/unsigned/long long、
  fp↔double、溢出special，每个运算独立成文件以便隔离电路。

## 2. Kratos2 的浮点表示链

| 层 | 表示 |
|---|---|
| C → K2（c2kratos） | `float`/`double` → `real`（数学实数）；`--no-floats-as-reals` 需配合 `--bitvectors` 才产出 `(fp E M)`；所测微用例的两种模式 BTOR2 **逐字节相同** |
| K2 → BTOR2（常量） | 编码期按实数语义求值后舍入为 IEEE 位模式（如 π/3 → `constd 1065749131` = 0x3F860A92） |
| K2 → BTOR2（变量值） | 32/64 位 bitvector，符号名 `fpasbv__8_23_32`（float-as-bitvector），用 `slice` 拆 sign/exp/mantissa |
| 浮点比较 | NaN 排除 + 同号正数直接 `ult` 位模式比较 + 符号分支 |
| 浮点除法（变量） | 48 位定点尾数 `udiv` + 指数相减 + `srl` 规格化 + `urem` 余数做 guard/sticky 舍入 |
| 断言 | 编译为常量位模式区间比较（如 0xC0000000 = -2.0f） |

关键结论：**BTOR2 标准格式没有任何浮点 sort/操作**（btor2tools parser 全部
108 个操作码仅整数位向量运算），Kratos2 的浮点能力来自"IR 层实数语义 +
落到 BTOR2 时自包含位级电路"，不是 BTOR2 原生支持。

## 3. Kratos2 除法电路结构（修复 C2Btor 的参考模板）

以 `y = x / n`（x∈(1,2), n∈(1,4)）为例，Kratos2 输出 479 行自包含电路：

1. 被除数有效数 `{1,m}` 左移扩展为 2P 位定点（P=24/53）；
2. `udiv` 得 25 位商，`urem` 得余数（余数非零 = sticky）；
3. 指数相减并加 bias，商的最高位决定是否再减 1；
4. guard/sticky 做 round-to-nearest-even；
5. NaN/Inf/零分支显式编码。

乘法同构：`{1,m}×{1,m}` 2P 位 `mul`，高 P 位 + guard + sticky 舍入。
加/减法：指数对齐移位 + 有效位加法 + 前导 1 规格化 + 舍入。

## 4. C2Btor 旧实现的缺陷

旧 lowering 复用 CBMC `float_utilst`，经 `btor2_propt`（propt 适配器）落到
BTOR2。适配器的 `set_equal()` 把 CNF 语境的 Tseitin 等式生成为 BTOR2 的
`constraint`——而 BTOR2 的 constraint 是**全局恒真假设**。后果：

- 电路正确性依赖大量 `__floatbv_bit$N` 自由位与 constraint 的锁定关系；
- constraint 缺失或冲突时，运算输出退化为自由位取值（实测 `pi/3` → 0.0、
  `1.5/3.0` → 0.0，赋值语义完全丢失）；
- 下游求解器在"锁不紧"的模型上证明 UNSAT，产生假 SAFE（见 session 文档
  Case 2）。

`constraint` 本身可以合法约束辅助变量；不能仅凭模型含有 constraint 就判定
翻译错误。上述问题来自旧适配器关系不完整/冲突的实验现象。自包含电路使
每个结果直接依赖操作数，避免将运算正确性交给辅助自由位的约束完整性。

## 5. C2Btor 新实现（本仓库 `src/goto-btor2/ieee754_arith.{h,cpp}`）

自包含 IEEE-754 位级电路，参数化 binary32(P=24,E=8)/binary64(P=53,E=11)：

```text
IEEE 编码 → normal/subnormal 规格化 → 扩展有效数运算
         → 保留 sticky 的移位 → 按当前模式单次舍入 + 渐进下溢 → IEEE 编码
```

- `normalize()` 将非正规数的前导 1 移到有效数顶端，同时调整阶码。
- `shift_right_jam()` 将所有移出位是否非零保存在低位；加法对齐、进位
  规格化和下溢共用此步骤。
- `round_pack()` 先移入非正规数区间再舍入，支持 RNE、向负无穷、向正无穷、
  RTZ、nearest ties-away；溢出按模式选择 Inf 或最大有限值，抵消的零符号遵循模式。
- 加法按完整幅值排序，仅异号且有效数之差为零才抵消；减法按真实左移距离
  修正阶码。旧电路对同号 `a+a` 也应用抵消，是历史假 SAT 的已证实根因。
- 乘法使用 2P 位精确乘积；除法使用 2P+4 位分子、P+4 位商和余数 sticky。
- 特殊值分支最后按优先级覆盖普通结果，NaN 及 invalid 运算不会被 Inf/zero 分支覆盖。
- fp→int 选择左/右移向零截断，独立检查截断值是否可表示；int→fp 共用
  舍入并处理进位；fp↔fp 共用输入规格化、重偏置及舍入/打包。
- 比较排除 NaN、按符号及幅值排序；正负零相等。一元负号/abs 只操作符号位。
- `ieee754_math.cpp` 实现整数开方加余数 sticky 的 sqrt、完整 2P 位乘积后只舍入
  一次的 FMA，以及按模 2B 保留商奇偶性的精确 remainder/fmod。余数不依赖当前模式。
- `lower_float_operations` 在 inline 前替换 CBMC 库的这些数学函数，保留同名用户
  函数和普通 CBMC symex；未用库中的近似 remainder 或扩展精度 FMA 代替标准运算。

电路结果不依赖浮点辅助自由位或全局等式 constraint。入口层负责契约检查：
静态不支持的 format/mode 拒绝转换；动态 rounding mode 和整数转换定义域
检查使用当前指令/分支路径的 `model_limit`，复用已有状态冻结及映射机制。
普通 C assume 仍可产生合法 constraint。

## 6. 当前验证与边界

- 42,539 项 binary32/binary64 oracle 与实际 BTOR2/BtorSim 对拍无不一致，
  NaN 仅比较分类，其他结果逐位比较。错误 oracle 对照能被检测。
- 96 项正常 C 集成检查（32/64 目标）完成；8 项支持边界检查报告 `model_limit`；
  long double 算术拒绝输出成功模型。所有成功转换经过 catbtor。
- 两个符号加法模型经 ric3/IC3 分别得到预期 UNSAT/SAT；SAT 标准反例经
  BtorSim 重放确认。命令、哈希、源文件、模型及日志见 session 文档及回归脚本。
- 撤回旧“184 组全通过故排除加法”的推断；也不把当前样本回归当作全输入
  IEEE-754 语义保持证明。

支持五种模式的算术/向浮点转换和整数转换 RTZ，包括非正规数、渐进下溢、sqrt、
FMA、IEEE remainder/fmod。尚未覆盖扩展格式算术、异常标志/陷阱、完整 fenv 或
NaN payload/sign 传播契约。RNA 四则使用 CBMC 值算术对照，RNA FMA 使用精确
整数乘积加和后单次舍入对照；四种宿主模式使用 native fenv。NaN 只比较分类。
隐藏边界后的 UNSAT 守门已实现并实测；本轮浮点 C witness/CPAchecker 和大型
benchmark 尚未重跑。完整证据与仍超时的堆检查见[本轮记录](float-validation/2026-10-03/README.md)。

语义参考：[C11 草案 §6.3.1.4](https://www.open-std.org/jtc1/sc22/wg14/www/docs/n1570.pdf)、
[Berkeley SoftFloat 的舍入及异常接口说明](https://www.jhauser.us/arithmetic/SoftFloat-3/doc/SoftFloat.html)。
本轮 oracle 是宿主原生 IEEE 运算，未调用 SoftFloat。
