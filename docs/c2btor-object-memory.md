# C2Btor object-local memory

`--memory object` selects separate storage for each GOTO-IR root object.
`--memory object --array bv` is the default. `--memory global` selects the
previous shared byte memory. Both memory organizations support `--array array`
and `--array bv`.
These are C2Btor options and require `--goto-btor2`.

```sh
build/bin/cbmc program.c --inline --goto-btor2 --memory object --array array \
  --goto-btor2-out program.object-array.btor2
build/bin/cbmc program.c --inline --goto-btor2 --memory object --array bv \
  --goto-btor2-out program.object-bv.btor2
build/bin/cbmc program.c --inline --goto-btor2 --memory global --array array \
  --goto-btor2-out program.global-array.btor2
```

This is a storage alternative inspired by an object-based memory model. It
reuses C2Btor's existing C/GOTO lowering, pointer representation and validity
rules; it does not claim to reproduce all of CBMC symex's memory semantics.

## Representation

Non-address-taken scalars remain ordinary BV state variables. Arrays,
aggregates and address-taken scalars have one backing object per root:

| Root | Object + Array Sort | Object + BV |
| --- | --- | --- |
| `int x` when address-taken | BV32 | BV32 |
| `int a[3]` | Array(BV2, BV32) | BV96 |
| `int a[2][3]` | Array(BV3, BV32), flattened target layout | BV192 |
| struct / union / arrays of aggregates | object-local byte array | object-local packed bytes |
| malloc slot | object-local byte array | object-local packed bytes |

The examples assume an 8-bit C byte and 32-bit `int`, as in both ILP32 and
LP64. Pointer/long elements use the configured target widths. An array's
index sort is at least one bit and otherwise `ceil(log2(number of cells))`.
A one-element array still has a BV1 index. Spare indices are not C elements.

C pointers retain `(object id, signed byte offset)`. A symbolic object id
dispatches reads/writes to registered objects. Each write has an object-id
match guard; all other objects retain their contents. Dispatch currently
considers all objects; points-to narrowing is a future optimization.

Typed arrays and character-pointer views share **one** backing. A byte read
selects an element and extracts its byte lane. A byte write replaces that
lane and preserves the other lanes and elements. Lane order follows target
endianness. There is no parallel typed copy that could diverge from memcpy,
union views or pointer accesses. Structures and unions use their target
layout through existing GOTO lowering and the existing address layer.
Complex byte access does not require a global-heap fallback in this version.

Each root also has local `written` and `defined` BV bitmaps and scalar BV
`size`, `live`, `zeroed` and `readonly` metadata. The bitmaps track **bytes**:
`int a[3]` has 12 bits in each bitmap on an 8-bit-byte target. Three bits would
lose partial-write information for `((char *)a)[i]` and byte copies. This first
implementation retains the metadata uniformly; eliminating provably redundant
flags is a separate optimization. Metadata zero initialization is a BV zero,
never a constant array. Data starts arbitrary; zeroed objects return zero
for unwritten bytes under the existing initialization policy.

For little-endian Object + BV, scalar reads/writes dispatch once per root and
extract/replace the whole word. Byte writtenness still selects zero for
unwritten calloc bytes; definedness is checked for every observed byte.
`object_copy.cpp` copies a byte interval in one atomic state update: source
data/flags are taken from the pre-state, unwritten zeroed bytes become defined
zero, and indeterminate source bytes remain indeterminate. A dynamic range
mask changes data/written/defined together while preserving all other bytes
and objects. This replaces the temporary snapshot and two byte loops for
array-copy/replace intrinsics in that encoding. Other encodings keep the loop
implementation; memcpy's non-overlap and memory-validity checks remain shared.

## Capacity and error classification

Fixed-size objects reserve their exact size; their sizes do not depend on a
capacity flag. VLA, runtime copy snapshots and malloc slots use automatic
capacities inferred from the final GotoIR; there is no default 1024-byte cap.
This applies to both Array Sort and BV data because local BV metadata still
needs a fixed width. See [automatic capacity](c2btor-memory-capacity.md) for
the analysis, representation ceilings, resource tradeoffs and regressions.
No extra parameter is mandatory.

```sh
# Override runtime-sized object capacity, independently of loop bounds.
build/bin/cbmc program.c --inline --goto-btor2 --memory object --array array \
  --memory-object-max-bytes 4096 --goto-btor2-heap-objects 8
```

In `--memory object --array bv`, either `--memory-object-max-bytes` or the
existing `--array-bv-max-object-bytes` sets the same effective capacity. If both
are supplied, their values must agree. The latter still requires `--array bv`.
Zero, malformed or unrepresentable capacities are rejected. Data cell rounding
may allocate a few spare bytes; the full byte offset and exact capacity are
checked before narrowing the array index or packed shift.

- Runtime size is retained separately from reserved capacity.
- Declaration/allocation exceeding capacity raises `model_limit`; it does not
  truncate, assume the path away, return NULL, or become a source assertion.
- Access outside actual object size, NULL access, use after free and invalid
  free use the shared `memory_validity` rules.
- Successful allocation count remains independently bounded by
  `--goto-btor2-heap-objects` (default 32); identities are not reused after free.
- Faulting paths halt ordinary execution. Source assertions, memory validity
  and model limits remain distinct properties and witness classifications.

The source map contains `memory.memory_encoding` and `memory.object_storage`:
object tag, capacity, cell width/count, data encoding, and all local state IDs.
Object mode has no single `memory_state` or global packed interval layout.

## Intended representation relation

For each registered object o and byte offset j in its reserved capacity,
relate the old global byte at `(id(o), j)` to the byte decoded from o's local
backing. Equate written/defined bits at that location and equate the object's
size, liveness, zero status and readonly status. Non-memory variables, PC,
fresh-object counter and pointer values agree. Unwritten, nonzero-initialized
data is unobservable under the current indeterminate-read boundary.

Under matching target layout, frontend options, allocation budget, supported
operations and in-capacity runtime sizes:

1. Initialization relates metadata and every initialized byte.
2. Reads select the same byte, subject to the same validity/definedness rule.
3. Writes replace the same byte; the object-id and offset guards give the
   frame condition for all other bytes/objects.
4. Allocation and lifetime changes update the same object metadata and counter.
5. Faults and source properties consequently correspond at the same GOTO step.

This describes proof obligations and the intended finite representation
correspondence, **not a completed implementation soundness proof**. Proving a
source assertion alone is insufficient if a model-limit property is reachable.
Array/BV comparisons must check the internal properties too.

## Operation coverage and retained boundaries

The regression fixtures cover scalar aliases; static/local arrays; partial,
designated and string initialization; symbolic indices; matrices and pointer
arrays; struct assignment/fields; union character views; pointer arithmetic,
equality, ordering/difference within one array; malloc/calloc/realloc/free;
memcpy/memmove/memset/memcmp; byte casts; NULL/OOB/UAF/scope exit; and capacity
classification. Positive checks include reachable completion controls.

The following shared limitations are not repaired by changing storage:

- integer/pointer conversions other than supported null forms, bitfield memory
  access and unknown layouts are rejected;
- repeated activation of the same automatic-object slot and indeterminate typed
  reads remain `model_limit`; these are narrower than general CBMC semantics;
- root bounds alone do not implement all subobject/dimension bounds or C UB
  checks; appropriate frontend instrumentation is still needed;
- no complete effective-type/provenance, concurrency or unbounded-allocation
  model is claimed;
- the previously observed `w_ok` readonly discrepancy, cross-object pointer
  ordering/invalid-pointer policy, and ILP32 index-narrowing gap remain separate
  semantic audit items. Object storage does not establish CBMC equivalence.

Pono compatibility depends on engine and SMT backend. Removing constant-array
metadata fixes that representation trigger; it is not a guarantee that every
array/interpolation algorithm handles the remaining formulas.

## Regression commands

```sh
cmake --build build --target cbmc -j8
python3 regression/goto-btor2/check_array_encoding.py --memory object \
  --btormc /path/to/btormc --output /tmp/object-array-new
python3 regression/goto-btor2/check_object_memory.py \
  --btormc /path/to/btormc --output /tmp/object-alias-new
python3 regression/goto-btor2/check_heap_rules.py --memory object --array bv \
  --memory-object-max-bytes 64 --heap-objects 3 \
  --btormc /path/to/btormc --output /tmp/object-heap-new
```

The runners accept `--include-dir` for environments with a stale SDK path.
Use new output directories. Each runner keeps commands/models/maps/logs and
result classes; BMC absence is only `bounded_no_violation`. Array tests also
support `--target 32` and `--big-endian`. The object runner verifies independent
backings, BV-only metadata, and absence of constant-array initializers.
