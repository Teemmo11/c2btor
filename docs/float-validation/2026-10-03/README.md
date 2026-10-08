# 浮点、内存编码与 UNSAT 判定接续验证

本次在 release 工作区继续实现；验证基线 HEAD 为 `638c1b35ef8eea39de917e2b56b1b7429839ae1a`。
验证时保留已有修改及 stash，尚未提交、推送或切换 main/dev。完整本地产物为
`/tmp/c2btor-float-complete-cl47driw`；[summary.json](summary.json) 保存结果、配置和哈希。
先前失败运行保留在该目录中，不能将它们计作通过。

## 实现与职责

- `ieee754_arith` 共用五种模式的舍入、渐进下溢、溢出选择和零符号；整数转换向零截断。
  `ieee754_math.cpp` 提供整数开方/余数 sticky 的 sqrt、完整乘积后单次舍入的 FMA，
  以及精确 remainder/fmod。库函数 lowering 独立于电路和普通 CBMC symex，保留同名用户函数。
- 模型注释及 map 保存所有 `proof_obligations`，包括隐藏的模型边界和内存有效性。
  批量工具使用包含所有源 bad 和义务的单一 OR 查询；witness runner 在源 UNSAT 后补查
  未覆盖属性/边界。旧模型缺少契约时不报告源 SAFE，BMC 无反例保持有界结果。
  多属性补查发现 SAT 而没有原模型属性编号时保留 `additional_property_violation`，不伪造 witness。
- Object/BV 标量访问一次读取/更新 word；`object_copy.cpp` 原子复制区间并传播初始化标记。
  数据来自前状态，目的区间外内容及其他对象保持；memcpy 非重叠检查和生命周期检查仍共用。
  其他编码保留字节循环。容量分析使用目标布局偏移、单值机器运算；不可达声明保留零容量。

## 证据层次

| 检查 | 结果与范围 |
| --- | --- |
| 构建 | `cmake --build build --target cbmc -j8` 成功；CMake/Make 均登记新模块 |
| 浮点数值 | 42539 项 binary32/binary64，对实际 BTOR2 经官方 catbtor 和 BtorSim 逐位对照；0 mismatch，错误 oracle 被检测 |
| C 浮点 | 104 项通过（96 项正常完成、8 项模型边界），覆盖 32/64 目标、动态模式、sqrt/FMA/remainder/fmod、同名用户函数、fesetround/fegetround 和非法模式/整数域；long double 算术明确拒绝 |
| 符号浮点 | 两个加法模型得到预期 IC3 UNSAT/SAT；SAT 经 BtorSim 回放 |
| 自动容量 | 当前 54 项、76 次具体执行回放通过，含 int[10000]、VLA、alias、机器 wrap、分配和显式超限 |
| 字节复制 | 32/64 目标共 48 项，在 object/BV、object/array、global/BV 下完成 parser 和具体执行；含双向重叠、非对齐别名、零长度、calloc、未确定字节、只读和 memcpy 重叠 |
| 判定契约 | 四个实际 C/solver 例及零 bad、多 bad 查询通过；隐藏容量/浮点边界和非法访问正确分类，真正安全的模型对照得到 model_unsat |
| 完整 runner | `run_witness_svcomp.execute` 的四个真实流程得到 model_limit、model_limit、memory_validity、model_unsat；带墙钟/RSS 监控 |
| 批量适配器 | 五个 stub 检查验证 rIC3/Pono 对旧模型和有界结果不会生成 SAFE 标签；这些是适配器检查，不是 Pono 求解证据 |
| Witness | 标准反例 → BtorSim → C YAML 的 33 项回归通过；两节点链表由 CPAchecker 得到 FALSE 且 confirmed，位向量/PRINCESS/无浮点 |

四种宿主舍入模式使用 volatile native arithmetic、真正的 fused `std::fma` 和 sqrt；
nearest ties-away 四则使用 CBMC 任意精度值算术，FMA 使用精确整数乘积加和后一次舍入。
NaN 只检查分类，其余逐位比较；没有使用 SoftFloat 或宣称全输入 IEEE 等价证明。
`float-5/` 保留数值电路与 oracle，`float-final/` 保留加入 C fenv 调用的当前集成检查。

本机默认 clang sysroot 指向不存在的 MacOSX27 SDK。涉及库函数的测试显式使用现有
`/Library/Developer/CommandLineTools/SDKs/MacOSX26.sdk/usr/include`，没有修改机器配置。

## 堆性能观察与支持边界

30/60 秒是本次实验的资源预算，不是实现验收条件。合理模型在预算内未完成求解，
记录 TIMEOUT 即可；不能据此推断编码有错或某项实现尚未修复。实现缺陷、支持边界、
求解资源不足分别判断，管线不得把 TIMEOUT/UNKNOWN 当作 SAFE。

在相同 ILP32、object/BV、成功分配预算 2、rIC3 1.5.6 word BMC、bound 240 和
30 秒预算下，realloc 完成对照模型由 4639 节点/170 位置减到 3466 节点/143 位置。
两次求解均超时，因此不能声称已消除该预算下的超时。另一批 60 秒检查中 realloc 终点
SAT 且 BtorSim 回放完成，memmove 的 UNSAT/终点对照也完成；copy-padding、realloc
和失败分支的若干无界检查仍未完成。两节点链表终点已解出，但无界源/边界证明仍未完成。

零长度 memmove 曾生成 2.6 GB 模型，原因是不可达库内数组被当作未知大小；修复后为
约 41 KB。偏移复制另有单值无符号负数/加法被过度放宽的问题，机器算术折叠修复后三种
编码均完成相关具体执行。这些修复没有添加新的 assume，也没有改变成功分配次数预算。

本机 rIC3 1.5.2 的 word 引擎缺 Bitwuzla，另有 bit 后端 panic；已有 1.5.6 的 wl-kind
因缺 get_ctrl 实现崩溃，portfolio 也观察到证书构造 panic。保留日志，不把崩溃计作验证通过。
`c2btor_solver.py` 兼容 --witness/--cex 的接口差异，没有修改外部 solver 源码或替换安装。
Pono 二进制缺 libpono.dylib，本轮未取得实际 Pono 判定。

尚未实现扩展浮点格式、异常标志/陷阱、完整 fenv、NaN payload/sign 传播契约；本轮
浮点 C witness 尚未经 CPAchecker。无界堆、递归调用栈和完整 C provenance 等原边界仍在。
上述回归和模型 UNSAT 都不构成完整 C 语义保持证明。任意外部工具自行丢弃契约或只选
源属性后得到的原始 UNSAT，仍不能直接解释成 C SAFE。

复现主要检查（在仓库根目录、每次使用新目录）：

```sh
run=$(mktemp -d /tmp/c2btor-checks.XXXXXX)
python3 regression/goto-btor2/check_float_lowering.py \
  --cbmc build/bin/cbmc --catbtor "checker/macOS arm64/catbtor" \
  --btorsim "checker/macOS arm64/btorsim" --ric3 "checker/macOS arm64/ric3" \
  --include-dir /Library/Developer/CommandLineTools/SDKs/MacOSX26.sdk/usr/include \
  --out "$run/float"
python3 regression/goto-btor2/check_model_contract.py --output "$run/contract"
python3 regression/goto-btor2/check_object_copy.py --target 64 \
  --include-dir /Library/Developer/CommandLineTools/SDKs/MacOSX26.sdk/usr/include \
  --output "$run/copy64"
python3 regression/goto-btor2/check_auto_capacity.py --output "$run/capacity"
```

数值、模型有效性、规则回归、backend 结果、回放和 C witness 确认分别记录；超时、
unsupported、缺工具和崩溃均保持独立状态。
