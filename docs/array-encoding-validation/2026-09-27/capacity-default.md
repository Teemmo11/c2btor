# BV 默认容量验证（2026-09-27）

运行时大小对象现在默认预留 1024 个目标 C 字节；
`--array-bv-max-object-bytes N` 仅用于覆盖默认值。固定大小对象仍精确编码。
有效容量同时写入日志和 source map。原先未提供容量即拒绝的策略已取消。

| 检查 | 结果 |
| --- | --- |
| ILP32/LP64 下 VLA 与 malloc 的容量和访问边界 | 26/26 |
| 默认 Array Sort、参数校验与默认堆分配预算 | 8/8 |

26 项检查涵盖默认值与显式 1024 的模型字节一致性、恰好达到容量、
超过默认容量、显式较小容量及超限、实际对象越界，以及固定数组大于显式容量。
所有模型通过 catbtor parser 检查且不含 Array Sort/read/write。
每例通过 btormc 在 150 步内到达预期属性：有效访问到达故意设置的终点断言，
容量超限到达 model_limit，实际对象越界到达 memory_validity。
这些可达性检查不是无界安全证明。

CLI 检查确认：未指定 --array 仍等价于 --array array；0、负数、格式错误的容量
以及在 Array Sort 模式使用容量参数仍被拒绝；默认堆预算仍为 32 次成功分配。
仅含动态堆的探针预留 32 × 1024 = 32768 字节，说明默认容量按对象计算，
不是整个程序的总内存界限。该堆预算探针进行了转换和元数据核对。

构建及 git diff --check 通过。结构化结果和二进制哈希见
[capacity-default.json](capacity-default.json)。原始模型、命令及日志保存在
`/tmp/c2btor-array-default-20260927/`。

复现容量检查：

```sh
python3 regression/goto-btor2/check_array_capacity.py \
  --cbmc "$PWD/build/bin/cbmc" \
  --catbtor "$PWD/checker/macOS arm64/catbtor" \
  --btormc /Users/west/Developer/simple_CAR/btor2/deps/btor2tools/deps/install/bin/btormc \
  --output /tmp/c2btor-capacity-new-run
```
