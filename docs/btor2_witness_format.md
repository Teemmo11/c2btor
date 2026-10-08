# SV-Witnesses YAML 格式说明 (v2.1)

> 本页是历史格式笔记。C2Btor 当前实现采用 YAML **2.0** 的 violation witness；
> 用法、字段和验证边界以 [c2btor-witness.md](c2btor-witness.md) 及官方规范为准。

> 基于 [sv-witnesses](https://github.com/sosy-lab/sv-witnesses) 官方规范整理。
> 参考文献: Ayaziová et al., *Software Verification Witnesses 2.0*, SPIN 2024.

## 1. 概述

软件验证 Witness 是 YAML 格式的文件，用于在验证器之间交换验证结果。有两种类型：

- **Violation Witness**（违反见证）：证明程序违反了规范（存在 bug）。
- **Correctness Witness**（正确性见证）：证明程序满足规范（无 bug）。

Witness 的核心价值：它不仅声称结果，还提供**可验证的信息**（如具体执行路径、不变量），使验证结果可被独立检查。

## 2. 公共元素

### 2.1 Location（位置）

所有 witness 中通用的位置描述：

```yaml
location:
  file_name: "main.c"    # 必需，文件名
  line: 42               # 必需，行号
  column: 5              # 可选，列号（1-based，tab 算 1 字符）
  function: "foo"        # 可选，所在函数名（便于人类阅读）
```

若省略 `column`，默认取该行中第一个语法上匹配的列位置。

### 2.2 Metadata（元数据）

每个 witness 条目必须包含元数据，描述规范、被验证程序和验证器信息：

```yaml
metadata:
  format_version: "2.1"
  uuid: "..."
  creation_time: "2025-01-01T00:00:00Z"
  producer:
    name: "ric3"
    version: "1.0"
  task:
    input_files:
      - "main.c"
    input_file_hashes: {...}
    specification: "CHECK( init(main()), LTL(G ! call(__VERIFIER_error())) )"
    data_model: "ILP32"
    language: "C"
```

### 2.3 表达式格式

Witness 中的表达式（如约束、不变量）有两种格式：

| 格式 | 说明 | 允许的扩展关键字 |
|------|------|------------------|
| `c_expression` | 标准 C 表达式，基于程序变量求值 | 无 |
| `ext_c_expression` | 扩展 C 表达式，支持 witness 扩展关键字 | 见下表 |

**扩展关键字：**

| 关键字 | 含义 |
|--------|------|
| `\result` | 函数调用的返回值 |
| `\at(x, Old)` | 参数 `x` 在函数调用参数求值完成后的值 |
| `\at(x, Pre)` | 参数或全局变量 `x` 在函数入口处的值 |
| `\at(x, AnyPrev)` | 变量 `x` 在当前求值点的某个先前值 |
| `\at(x, LastPrev)` | 变量 `x` 在当前求值点的上一次值 |

表达式求值要求：
- 无副作用
- 无未定义行为
- 并发程序中原子求值
- 求值结果必须是明确的 `true` 或 `false`

## 3. Violation Witness（违反见证）

### 3.1 结构

Violation witness 由一个 `violation_sequence` 条目组成，包含一系列 **segment**，每个 segment 包含一系列 **waypoint**：

```
violation_sequence
  └── segment [1..*]
        └── waypoint [1..*]
            ├── type: assumption | target | function_enter | function_return | branching
            ├── constraint [可选]
            │     ├── value: "C 表达式"
            │     └── format: c_expression | ext_c_expression
            ├── location
            └── action: follow | avoid | cycle
```

### 3.2 Waypoint 类型

| 类型 | 含义 | 位置指向 | 求值点 | 通过条件 |
|------|------|----------|--------|----------|
| `assumption` | 在某位置断言一个条件 | 语句或声明 | 位置前的 sequence point | `constraint` 求值为 true |
| `branching` | 限制分支走向 | 分支语句 | 控制表达式求值后的 s.p. | 控制表达式值与 `constraint` 匹配 |
| `target` | 标记违规发生位置 | 包含违规的语句 | 不求值 | 永远不会 "通过" |
| `function_enter` | 进入函数调用 | 调用点的右括号 `)` | 参数求值完成后的 s.p. | 无额外条件 |
| `function_return` | 函数返回 | 调用点的右括号 `)` | 函数调用完成后的 s.p. | 返回值满足 `constraint` |

### 3.3 Waypoint Action

| Action | 语义 |
|--------|------|
| `follow` | waypoint 被求值时**必须通过**，且恰好通过一次 |
| `cycle` | waypoint 被求值时**必须通过**，且将**无限次通过**（用于活性/非终止） |
| `avoid` | waypoint **永远不能通过**，但可以被求值任意次（含 0 次） |

### 3.4 Segment 规则

- 一个 segment 包含 1 个或多个 waypoint。
- 最后一个 waypoint 必须是 `follow` 或 `cycle`。
- 一个 segment **恰好**包含 1 个 `follow` 或 1 个 `cycle`。
- 可包含任意数量的 `avoid` waypoint。
- Segment 内 waypoint 的**顺序无关**。

**segment 类型：**
- **follow segment**：含 1 个 `follow` waypoint，无 `cycle`
- **cycle segment**：含 1 个 `cycle` waypoint，无 `follow`
- **final segment**：含 `target` waypoint 的 segment（必须是最后一个 segment）

### 3.5 执行语义

Violation sequence 是一组 segment 的有序序列。执行过程中维护一个"当前 segment"：

1. 初始时，当前 segment 是第一个 segment。
2. 执行与当前 segment 匹配后，推进到下一个 segment。
3. 若最后一个匹配的是 cycle segment，则回到第一个 cycle segment（循环）。
4. Final segment 匹配后无下一个 segment。

一段执行**匹配**一个 normal segment，当且仅当：
- 所有 waypoint 的语义不被违反
- 执行在某个 waypoint 通过时结束

一段执行**匹配**一个 final segment，当且仅当：
- 所有 waypoint 的语义不被违反
- 在 `target` 位置指向的语句求值期间发生了规范违反

### 3.6 完整示例

以下是一个 violation witness 示例，描述一个数组越界 bug 的反例：

```yaml
- entry_type: violation_sequence
  metadata:
    format_version: "2.1"
    uuid: "c0ffee00-dead-beef-1234-567890abcdef"
    creation_time: "2025-04-20T12:00:00Z"
    producer:
      name: "cbmc-btor2"
      version: "6.0.0"
    task:
      input_files:
        - "t03_array_basic_bug.c"
      specification: "CHECK( init(main()), LTL(G ! call(__VERIFIER_error())) )"
      data_model: "ILP32"
    language: "C"
  content:
    - segment:
        - waypoint:
            type: function_enter
            location:
              file_name: "t03_array_basic_bug.c"
              line: 5
              function: "main"
            action: follow
    - segment:
        - waypoint:
            type: assumption
            constraint:
              value: "x == 1"
              format: c_expression
            location:
              file_name: "t03_array_basic_bug.c"
              line: 7
            action: follow
    - segment:
        - waypoint:
            type: assumption
            constraint:
              value: "arr[0] == 3"
              format: c_expression
            location:
              file_name: "t03_array_basic_bug.c"
              line: 8
            action: follow
    - segment:
        - waypoint:
            type: assumption
            constraint:
              value: "arr[2] == 8"
              format: c_expression
            location:
              file_name: "t03_array_basic_bug.c"
              line: 10
            action: follow
    - segment:
        - waypoint:
            type: target
            location:
              file_name: "t03_array_basic_bug.c"
              line: 11
              function: "main"
            action: follow
```

解读：
1. 进入 `main` 函数（line 5）
2. 断言 `x == 1`（line 7 处的赋值结果）
3. 断言 `arr[0] == 3`（line 8 处的赋值结果）
4. 断言 `arr[2] == 8`（line 10 处的赋值结果——这里 8 是 bug 值）
5. 到达 target（line 11 处的 `__VERIFIER_error()` 调用），违规发生

### 3.7 安全规范与活性规范

**安全规范**（如 unreach-call, memory-safety, no-overflow）：
- 不得包含 `cycle` waypoint
- 必须恰好包含 1 个 final segment

**活性规范**（如 no-cycle / 非终止）：
- 结构为：stem（0 个或多个 normal segment）+ cycle（1 个或多个 cycle segment）
- 表示无限执行的循环部分

## 4. Correctness Witness（正确性见证）

### 4.1 结构

正确性见证可以包含两种类型的条目：

```
correctness witness
  ├── invariant_set [可选]
  │     ├── invariant [0..*]
  │     └── contract [0..*]
  └── ghost_instrumentation [可选]
        ├── ghost_variable [0..*]
        └── ghost_update [0..*]
```

### 4.2 Invariant（不变量）

不变量描述在某个位置始终成立的条件：

```yaml
- entry_type: invariant_set
  metadata: { ... }
  content:
    - invariant:
        type: loop_invariant
        location:
          file_name: "main.c"
          line: 10
          function: "foo"
        value: "i >= 0 && i <= n"
        format: c_expression
        labels:
          - inductive
```

**不变量类型：**

| 类型 | 位置指向 | 求值点 | 说明 |
|------|----------|--------|------|
| `location_invariant` | 语句或声明 | 语句前的 s.p. | 在该位置始终成立 |
| `loop_invariant` | 循环语句 | 循环控制表达式求值前的 s.p. | 每次循环迭代都成立 |
| `location_transition_invariant` | 语句或声明 | 语句前的 s.p. | 比较当前状态与所有先前状态 |
| `loop_transition_invariant` | 循环语句 | 循环控制表达式求值前的 s.p. | 循环内的跨迭代关系 |

**`labels`** 字段：可包含 `inductive` 标签，表示这是一个归纳不变量——从满足该不变量的任意状态出发，沿合法路径回到该求值点时，状态仍然满足不变量。

### 4.3 Contract（函数契约）

```yaml
- entry_type: invariant_set
  metadata: { ... }
  content:
    - contract:
        type: function_contract
        location:
          file_name: "main.c"
          line: 3
          function: "abs"
        requires: "x >= -1000 && x <= 1000"
        ensures: "\\result >= 0"
        format: ext_c_expression
        labels:
          - inductive
```

- `requires`：前置条件，在函数调用参数求值完成后、函数体执行前成立。
- `ensures`：后置条件，在函数体最后一个 full expression 求值后成立。
- `ensures` 中可用 `\result` 引用返回值，用 `\at(x, Old)` 引用参数在调用时的值。

### 4.4 Ghost Instrumentation（幽灵变量）

为多线程等场景引入额外辅助变量：

```yaml
- entry_type: ghost_instrumentation
  metadata: { ... }
  content:
    ghost_variables:
      - name: "ghost_locked"
        type: "int"
        scope: global
        initial:
          value: "0"
          format: c_expression
    ghost_updates:
      - location:
          file_name: "main.c"
          line: 15
        updates:
          - variable: "ghost_locked"
            value: "1"
            format: c_expression
```

Ghost 变量规则：
- `name`：必须是有效的 C 标识符，且不与程序中已有标识符冲突。
- `initial.value` 在全局变量初始化后求值，不能引用局部变量。
- Ghost update 在控制流离开指定位置时原子执行。

## 5. 支持的规范 (SV-COMP 2026)

### 单线程

| 规范 | LTL 公式 | Witness 类型 |
|------|----------|-------------|
| unreach-call | `G ! call(func())` | violation / correctness |
| no-overflow | `G ! overflow` | violation / correctness |
| memory-safety | `G valid-free` / `G valid-deref` / `G valid-memtrack` | violation / correctness |
| termination | `F end` | correctness |
| no-cycle (non-termination) | `F end` | violation |

### 多线程 (ConcurrencySafety)

| 规范 | LTL 公式 |
|------|----------|
| unreach-call | `G ! call(func())` |
| no-data-race | `G ! data-race` |
| no-overflow | `G ! overflow` |
| memory-safety | 同上 |

## 6. C2Btor 当前反例流程

旧的 BTOR2 → GotoIR 回放 → GraphML/YAML 代码已删除。
当前使用 [c2btor-witness.md](c2btor-witness.md) 中的 source map、BtorSim 回放、
SV-COMP YAML 2.0 输出和 CPAchecker 验证流程。SAFE 不输出不变量 witness。

回译以实际执行的 nondet 调用为事件，使用执行后的返回值生成
`function_return` waypoint；`target` 指向原程序的错误函数调用点。
不将赋值后的值直接写成赋值前的 `assumption`。

### 对象/字节内存模型的辅助属性

新版 map 的 `memory.properties` 区分 `memory_validity` 和 `model_limit`，二者不能
转换成 unreach-call violation witness。`memory.allocation_sites` 关联分配 PC、大小
节点和对象计数状态；`allocation_budget` 是总成功分配次数上界，不是最大活对象数。

BtorMC 完整轨迹中的第 0 帧已初始化数组行可省略，由 BtorSim 计算 init。译器同时
保存原始规范化轨迹和供模拟的稀疏轨迹，不删除输入、未初始化数组或后续帧。
见 [堆模型实施及验证记录](heap-validation/2026-09-23/README.md)。
