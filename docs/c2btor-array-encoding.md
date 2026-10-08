# C2Btor array storage encodings

This document describes the optional `--memory global` representation.
The default is `--memory object --array bv`.
For separate object backings and local BV metadata, see
[object-local memory](c2btor-object-memory.md); all four memory/array combinations are available.

`--array array` and `--array bv` (the default) select the storage representation
of `--goto-btor2`. They do not change C type widths, object identities, pointer
arithmetic, lifetime rules, or the selected source properties. These options
are rejected outside `--goto-btor2`, rather than silently affecting CBMC symex.

```sh
# The existing global Array Sort representation.
build/bin/cbmc program.c --inline --goto-btor2 --memory global --array array \
  --goto-btor2-out program.array.btor2

# Global pure bit-vectors; runtime-sized capacities are inferred.
build/bin/cbmc program.c --inline --goto-btor2 --memory global --array bv \
  --goto-btor2-out program.bv.btor2

# Optionally override capacity for VLA, runtime copy snapshots and malloc.
build/bin/cbmc program.c --inline --goto-btor2 --memory global --array bv \
  --array-bv-max-object-bytes 64 --goto-btor2-heap-objects 2 \
  --goto-btor2-out program.bv.btor2 \
  --goto-btor2-map-out program.bv.map.json
```

Use the same target (`--32`/`--64`, endianness), frontend options, instrumentation,
program and properties when comparing representations. Runtime-sized objects
use inferred capacities by default; `--array-bv-max-object-bytes N` optionally
sets an explicit bound. See [automatic capacity](c2btor-memory-capacity.md). Larger capacities increase packed storage
and may increase solving cost. The effective capacity is printed in the log
and recorded in the source map. Capacity is measured in C bytes of the configured target, not array elements. Fixed-size objects use
their exact size, even when larger than `--array-bv-max-object-bytes`. This option
is a storage capacity for runtime-sized objects, not a bound on loop iterations
or a malloc-failure policy. The existing allocation-count bound is independent.

## Representation and semantic obligations

The current converter already places arrays, aggregates and address-taken
scalars in one authoritative byte memory. Therefore, replacing only native
array expressions would not make a model consumable by a pure-BV backend.
BV mode also removes the arrays for initialized/defined bytes, object size,
liveness, zero initialization and readonly status. A final guard rejects a BV
conversion if any Array Sort remains; it emits no successful partial model.

Native mode keeps `sort array`, `read` and `write`. BV mode compacts the finite
storage into packed bit-vectors. For a storage vector of N elements of w bits,
element i occupies bits `[w*i+w-1 : w*i]`. Symbolic reads use logical shift and
extraction; writes use a shifted mask and preserve every other element. The
full storage index received from GOTO-IR is checked before narrowing a shift
amount. This prevents wrapping inside the packed storage helper, but does not
recover information already lost during frontend index conversion (see below).

C pointers remain `(object id, byte offset)`. They are not replaced with compact
storage indices. A checked dispatch maps each object to a disjoint packed byte
interval. Direct access, pointer access, structure members and byte aliases
therefore use the same bytes and retain the target's endianness and layout.
The source map records `memory.array_encoding`, `packed_bytes`, the effective
capacity and `(tag, byte_start, capacity_bytes)` for each reserved object.

The intended representation relation equates each in-capacity byte and its
initialization flags, and equates all object metadata in the two encodings.
Initialization, read, write, allocation and lifetime operations should preserve
this relation until a declared capacity is exceeded. The regressions below
check representative instances; they are not a completed semantic-preservation
proof for the implementation.

## Capacity and result classification

- A size simplified to a compile-time constant is stored exactly.
- BV mode infers runtime-sized root capacities and a maximum request for
  allocation slots from conservative integer intervals over the final GotoIR.
  Existing assumptions and branch conditions may refine those intervals; no
  new assumptions are added. `--array-bv-max-object-bytes N` sets an explicit
  positive bound. Zero, negative and malformed capacities are rejected.
- Actual VLA/allocation size remains latched in metadata. A declaration or
  allocation exceeding its reserved capacity raises `model_limit` and freezes
  ordinary execution of the affected path. It is not silently truncated,
  assumed away, or converted into malloc returning NULL.
- An access beyond the actual object size remains `memory_validity`, even when
  the reserved capacity is larger. Source assertion failure is a separate class.
- Fixed objects or total packed storage whose widths cannot be represented by
  the BTOR2 implementation are rejected before width arithmetic can overflow.
- `SAT` on `model_limit` is inconclusive for source safety. A proof of the source
  assertion alone must not hide reachable model-limit properties. Check the
  internal properties as well and retain their classes in the result report.

Global Array Sort mode retains all existing model boundaries, including
allocation count and pointer/object limits; it does not imply unbounded memory.

## Array features and remaining boundaries

| Feature | Behavior in this change |
| --- | --- |
| Fixed-size local/global/static arrays | Exact object size; both representations share initialization and lifetime semantics. |
| VLA | Runtime size is latched by the existing lowering; BV reserves inferred or explicit capacity and checks it. |
| Constant/symbolic reads and writes | Same lvalue and byte semantics; BV implements selection/update without array operations. |
| Two-dimensional arrays | Row-major target layout, pointer-to-array stride and byte offsets are retained. Fixed matrices are covered by regression. |
| Element types | Existing scalar/aggregate lowering is reused. Tests cover signed/unsigned widths, `_Bool`, float, pointer and structure elements. No blanket claim of all C types is made. |
| Pointer/alias access | Array decay, interior pointers, pointer arrays and character views share one authoritative object storage. |
| Initialization | Zero/static, partial, designated, string and explicit initializers follow frontend lowering. Uninitialized typed reads retain the existing `model_limit` behavior; they are not changed to zero. |
| Assignment/copy | C has no general `a = b` array assignment. Structure assignment containing arrays, byte copies and overlapping memmove use their existing structural/snapshot lowering. |
| Bounds | Root-object access validity is always tracked. Full per-dimension/subobject C bounds require the appropriate frontend instrumentation, e.g. `--goto-btor2-checks --bounds-check`. Root bounds alone are not all C UB checks. |
| Allocation/free | Allocation identity and count, malloc/calloc initialization, copying and liveness remain shared semantics. |

When checking instrumented array bounds, do not pass
`--no-built-in-assertions`: in the current CBMC pipeline it removes generated
bounds assertions too. A row-crossing access can be inside the root object and
still violate a C array dimension; the regression probe verifies this case
with bounds instrumentation retained in both representations.

This change does not remove existing restrictions on repeated activation of the
same automatic object slot, indeterminate typed values, bitfield memory access,
unknown layouts, full provenance/effective-type rules or concurrency. Aggregate
assignment currently rejects an array extent over 4096 unless handled by loop
lowering. Large fixed arrays and generous dynamic capacities may create very
large BV models; backend compatibility does not imply a speedup.

## Reproducible checks

```sh
cmake --build build --target cbmc -j8
python3 regression/goto-btor2/check_array_encoding.py \
  --btormc /path/to/btormc --catbtor /path/to/catbtor \
  --output /tmp/array-rules-unique

python3 regression/goto-btor2/check_array_encoding.py \
  --btormc /path/to/btormc --catbtor /path/to/catbtor \
  --target 32 --cases 1,3,5,7,9,13,16 --output /tmp/array-rules-32-unique

python3 regression/goto-btor2/check_heap_rules.py \
  --btormc /path/to/btormc --catbtor /path/to/catbtor \
  --array bv --array-bv-max-object-bytes 64 --heap-objects 2 \
  --cases 2,4,13,15,17 --output /tmp/array-heap-unique
```

Each new-array test checks official parser acceptance and, for BV, the absence
of all Array Sort/read/write operations. Both modes run the same C cases,
including deliberate errors, capacity exhaustion, multidimensional/pointer
views, copying, element widths and endianness. Positive cases also have a
reachable completion control. BMC absence is recorded as
`bounded_no_violation`, never as an unbounded safety proof. The runner saves
commands, tool/source hashes, model sizes and property classes.

Before a paper ablation, additionally measure translation time, total BV state
bits, BTOR2 size, backend time, peak memory and end-to-end time on a fixed common
task set. Include unsupported, model limits and timeouts; do not report them as
ordinary source wrong results. Verify the chosen rIC3 version/algorithm: the
local checkout has array-elimination paths, so “rIC3 never supports arrays” is
too broad. Pure BV output removes dependence on that preprocessing succeeding.

## Re-audit: remaining frontend and verification gaps

The 2026-09-27 expanded re-audit found an ILP32 frontend boundary: `a[i]` with
`unsigned long long i = 1ULL << 32` reaches both storage encodings as
`a[(signedbv[32]) i]`. With conversion/bounds instrumentation disabled, both
modes access element zero and fail to report the original oversized index.
This is an unresolved pipeline issue, not evidence of array/BV disagreement.
Explicit `--goto-btor2-checks --conversion-check` detects this probe as an
`overflow` property when built-in assertions are retained; this is not a fix
for the default pipeline or a proof of all index conversions. Distinguishing
frontend-inserted narrowing from explicit source casts needs further work.

The malloc-failure-enabled realloc rule remains incomplete: both encodings
timed out at 400 BMC steps / 45 seconds, although the completion controls passed.
The frontend also rejects the tested `choose ? a : 0` array-decay expression;
the explicit-pointer form `choose ? &a[0] : (int *)0` passes the guarded-load
checks. Rejection is a coverage limit, not a silently emitted model.

Full results, operation coverage and a conditional representation argument are
in [the re-audit](array-encoding-validation/2026-09-27/reaudit/README.md).
These findings supersede any interpretation of previous regression counts as
complete C-array correctness or completion of the historical wrong-result audit.
