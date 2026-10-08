# C2Btor release 分支修改总结（2026-09-23）

本次沿用 CBMC GotoIR → BTOR2 路线，引入统一对象/字节堆模型与经过独立验证的
C 级 violation witness 流程。不改变普通 CBMC symex 的 malloc/free 实现。

## 修复的 bug

1. **字段与根对象存储不一致**：原先 heap 根数组和字段投影可能独立更新，内部
   指针、数组元素或另一个类型视图读不到实际写入。现在统一为对象号＋字节偏移，
   所有合法视图读取同一存储。
2. **循环 malloc 合并对象**：原先静态调用点可能复用单个 heap 对象，破坏链表、
   树等多节点对象的独立性。现在每次成功分配使用新身份，并检查显式容量。
3. **大小及初始化语义错误**：分配大小在事件发生时锁存；malloc 不再默认清零；
   calloc 按字节清零并保留乘法溢出判断。CBMC 溢出结果的逻辑临时 pair 单独建模。
4. **释放后仍可普通访问**：不再仅依赖随机选择的 deallocated 监视变量；保留真实
   释放事件并更新 live，识别释放后访问、重复释放和内部指针释放。
5. **复制丢失初始化信息**：字节与初始化标记一起复制，未初始化 padding 可作为
   对象表示复制；不能因目标来自 calloc 就把源未初始化字节变成确定零值。
6. **不完整翻译仍输出可用模型**：无法处理的指针/字段/指令不再静默引入任意
   输入后当作正常转换；转换失败时不输出可供验证的模型。
7. **循环展开与界限混淆**：修正展开指令顺序、循环常量/更新条件分析；有限
   --unwind 使用 ASSERT_ASSUME，展开不足的断言不能冒充 C 源违例。
8. **后端 SAT 被混同为 C 反例**：区分源属性、memory_validity、model_limit；
   旧无轨迹运行器遇到含辅助内存属性的 SAT 返回 unknown，防止直接记作 TF/FF。
9. **数组初始化轨迹兼容性**：BtorMC 完整轨迹的初始化数组行可能被 BtorSim 拒绝；
   译器省略可由 init 决定的第 0 帧数组行，保留原轨迹并重新模拟，不伪造输入值。

## 新增的功能

- `heap_model`、`memory_access`、`lower_heap_operations` 三个模块，管理对象、布局、
  字节内容、生命周期与库原语来源。
- `--goto-btor2-heap-objects K`，默认 32 次总成功分配；超限作为独立模型边界。
- 源码映射导出：BTOR 节点/PC → GotoIR 指令 → C 位置，同时记录分配与属性类别。
- `scripts/c2btor_witness.py`：标准 BTOR2 witness → BtorSim → SV-COMP YAML 2.0
  → CPAchecker；校验模型、源码与 map 的哈希绑定。
- `scripts/run_witness_svcomp.py`：有时间/内存限制的批量导出、求解、翻译与验证。
- 内存规则回归、链表反例及支持范围文档；移除旧 BTOR2-to-GraphML/YAML C++
  回放入口，不再维护两套 witness 路径。

## 完善的功能

- 标量、结构体、成员数组、结构体数组、多维数组、柔性数组与多级指针的统一访问。
- 全局对象、字符串初始化，目标字节序和 ABI 成员偏移，取地址变量与指针别名一致性。
- free(NULL)、memcpy、memmove、memset、realloc 的前缀复制及成功/失败控制流。
- nondet 返回值使用执行后的状态，保留重复调用；错误函数在内联前保留原始调用位置。
- SAFE 不生成 invariant witness；未知、不支持、超时与已确认的违例分别记录。

## 验证结果与仍有限制

- 最终源码构建成功；相关 Python 语法检查及 `git diff --check` 通过。
- 内存规则共 41 项：40 项符合预期，1 项为允许 malloc 失败的 realloc 求解超时
  （400 步、60 秒）。最终核心路径复查 7/7；witness 回归 33/33。
- 两节点链表反例通过 BtorSim、YAML 2.0 与 CPAchecker/PRINCESS 位向量验证。
- 历史 TF 名单 57 个：52 个转换且官方 BTOR2 parser 通过；5 个 c_dsa 仍因递归
  明确拒绝。这不是“52 个已经证明 SAFE”或“所有 TF 已消除”。
- 第一版有界身份不复用，重复的取地址局部对象生命周期、位域内存访问、指针整数
  地址表示转换等仍有限制；完整 C provenance/effective-type/trap 规则尚未完成。
  界内无反例及 parser 通过均不构成完整语义证明。

详细证据见 [堆模型实施及验证记录](heap-validation/2026-09-23/README.md)。
2026-09-22 witness 统计作为历史快照单独保存，不与新版结果合并计算。
