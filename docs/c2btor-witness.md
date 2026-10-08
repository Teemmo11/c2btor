# C2Btor: BTOR2 counterexample to SV-COMP violation witness

The supported submission path is:

```text
C -> CBMC GotoIR -> BTOR2 + translation map
                           |
                       model checker
                           |
                    standard BTOR2 witness
                           |
                  BtorSim checking + replay
                           |
                  source-mapped call events
                           |
                  SV-COMP YAML 2.0 witness
                           |
                  CPAchecker validation
```

This follows CPV's use of circuit simulation and source-level nondeterministic
call events. CPV's implementation tracks instrumented call counters and results;
C2Btor can identify each executed GotoIR instruction from the simulated PC.
There is no new counterexample format and no correctness-witness generation for
SAFE results.

## Export, solve, translate, validate

Run from the repository root. Use an SV-COMP C input containing direct
`__VERIFIER_nondet_*()` calls and `__VERIFIER_error()` for the initial supported
reachability subset. All output directories must be new; historical results
are never overwritten by the wrapper.

```sh
witness_run=$(mktemp -d /tmp/c2btor-witness.XXXXXX)

python3 scripts/c2btor_witness.py export \
  --c2btor build/bin/c2btor --program program.c \
  --output-dir "$witness_run/export" -- --32

ric3 check --witness \
  "$witness_run/export/model.btor2" bmc > "$witness_run/backend.witness"

python3 scripts/c2btor_witness.py translate \
  --model "$witness_run/export/model.btor2" \
  --map "$witness_run/export/model.map.json" \
  --witness "$witness_run/backend.witness" \
  --btorsim btorsim \
  --output-dir "$witness_run/translation"

python3 scripts/c2btor_witness.py validate \
  --witness "$witness_run/translation/witness.yml" \
  --cpachecker /path/to/CPAchecker/bin/cpachecker \
  --output-dir "$witness_run/validation"
```

`export` uses the repository's default inlining/check-disable flags. Additional
CBMC options follow `--`. `--32` and `--64` are preserved in the command record;
the witness's ILP32/LP64 field is derived from the actual exported target widths.
Use `export --spec property.prp` for a different unreach-call error function,
including SV-COMP's `reach_error()`. The wrapper passes
`--goto-btor2-error-function` to CBMC, which marks the violating call before
inlining so its original call-site location is retained. Other property types
are rejected.

The `.yml` file uses JSON syntax, which is valid YAML 1.2 and is accepted by the
tested CPAchecker YAML witness parser. Python requires no third-party packages.
The exporter includes SHA-256 input hashes, UTC creation time, UUID, source
coordinates, task language, data model, and property specification.

`translate` also saves `trace.json`, the normalized backend witness, and the
simulator command/stdout/stderr. `validate` saves its generated configuration,
command, logs, and `result.json`. A simulator or validator failure never becomes
a success simply because an output file exists. CPAchecker `FALSE` confirms an
error execution constrained by the witness; `TRUE` or `UNKNOWN` does not confirm
the submitted violation witness.

## Translation map

The native export option is `--goto-btor2-map-out model.map.json`. It writes
metadata from the actual converter, after inlining/lowering/unwinding, including:

- PC state node and C target widths.
- Each PC value, GotoIR location number, instruction kind/text, source file,
  line, column, function, and property ID.
- The instruction's PC-guard node and range of nodes created while processing
  it; the exact `bad` node for each assertion.
- State-node to full GotoIR symbol mapping.
- Nondeterministic call identifier, return-state node, return type, and the
  destination variable for a directly consumed return value.

The Python `export` wrapper binds this map to the model and mapped source files
with SHA-256 and records the task input and specification. `translate` requires
this bound map and rejects changed model/source files. A raw native map remains
useful for inspecting the translation but is not accepted as a bound bundle.

This is provenance, not a claim that every BTOR operator has exactly one C
statement. Shared expressions, initialization, and final `next` ITE chains can
serve several instructions. Instruction node ranges record where nodes were
created; PC guards and bad nodes identify execution/violation locations exactly.

## Back-translation semantics

BTOR2 witness indices are zero-based **state/input ordinals**, not BTOR node IDs.
`#t` and `@t` belong to the same frame. Backend logs are stripped at exact `sat`
headers; `UNSAT`, truncated witnesses, and invalid property indices are rejected.

BtorSim checks the witness against the circuit and reconstructs missing internal
states. Its scalar state/input values are retained in `trace.json`. Array rows
remain in the simulator output and are not misread as scalar values. Unknown
scalar bits are rejected rather than silently replaced with zero.

At frame `t`, `pc[t]` identifies the instruction about to execute. If it executes
a nondeterministic call, its result is read from **state `t+1`** and emitted as:

```yaml
type: function_return
action: follow
constraint:
  format: acsl_expression
  value: '\result == -3'
location:
  file_name: program.c
  line: 5
  column: 35 # the closing parenthesis of the call, resolved from the source
```

Repeated call events are preserved even when their returned values are equal.
When a line contains multiple calls, the direct GotoIR result-to-variable flow
can distinguish source assignments/initializers. This also preserves columns
when a helper function is inlined several times. Calls with ambiguous locations
still fail; the translator never guesses their order from C argument evaluation.
Signed results are decoded using their actual bit width. The final simulated PC
must map to a claimed bad property and a source error call. Earlier visits to an
assertion are not automatically treated as violations. A target is never invented
from the last assignment. General post-assignment values are not exported as
pre-statement assumptions.

## Validation and supported boundary

Initial scope: unreach-call, ILP32/LP64, scalar integer/bool nondet returns,
unambiguous direct nullary calls, and standard BTOR2 safety counterexamples.
Macros or ambiguous source call locations, pointer/float nondet returns, other
property types, and unsupported machine models fail explicitly. Array programs
can be simulated, but this stage does not reconstruct array/pointer assumptions.
Uninitialized C values are not exported as additional source assumptions.

rIC3 and BtorMC standard witness output are tested. AVR output conforming to the
same standard can be fed to the simulator, but has not been independently tested.
Raw ABC/AIGER or backend-specific traces first require the backend's inverse
bit-blasting/encoding map; renaming a raw AIGER trace to BTOR2 is insufficient.

Compiler-library nondeterministic choices, such as malloc success/failure,
remain in the circuit trace but do not create fictitious C function-return
waypoints. The C validator supplies the original library semantics.

The validator wrapper explicitly selects CPAchecker's predicate witness
validator. In the tested local installation, the default restart portfolio could
fall back after missing SMT libraries and accept a deliberately contradictory
return-value witness. The explicit predicate configuration passes the positive
and negative controls and avoids that fallback.

For integer/array/pointer examples on installations without native SMT libraries,
the tested configuration retains exact bitvectors:

```sh
python3 scripts/c2btor_witness.py validate ... \
  --solver PRINCESS --no-floats
```

Actual floating-point formulas fail explicitly in this mode. Optional
`--solver SMTINTERPOL --integer-encoding` uses integer/rational approximation
and records it in the result; it was **not** used for the expanded SV-COMP batch.
That approximation is not evidence for general bit-vector overflow or floats.
Likewise, one validated witness confirms an execution, not a semantics-preservation
theorem for all C2Btor translations.

The wrapper disables CPAchecker's optional output-file generation to avoid a
PRINCESS formula-dump crash after validation. Commands, stdout/stderr, and the
verdict remain saved by the wrapper. A successful process exit and `FALSE` are
both required; analysis/replay failures are not suppressed.

The old BTOR2-to-GraphML/YAML entry points, C++ parsers, and GotoIR replay code
have been removed. Use the Python workflow above. CBMC's independent native
GraphML witness support for its own verifier is unaffected.

## Regressions

```sh
python3 regression/goto-btor2/check_witness_translation.py \
  --c2btor build/bin/c2btor --ric3 ric3 \
  --btorsim btorsim \
  --cpachecker /path/to/CPAchecker/bin/cpachecker

```

The integration suite checks conditional reachability, repeated signed nondet
events in a loop, same-line calls in a twice-inlined helper, `reach_error` call
sites, compiler-library malloc events, a second bad property, an initial violation, sparse witnesses,
changed source/model rejection, truncated/UNSAT rejection, a real SAFE circuit,
and rejection of a contradictory return-value witness by CPAchecker. Omitting
`--cpachecker` runs circuit/translation checks only. Optional `--solver`,
`--integer-encoding`, and `--no-floats` are forwarded to validation. Use
`--btormc /path/to/btormc` for the heap-model regression on rIC3 installations
with array/witness crashes.

The [2026-09-22 validation report](witness-validation/2026-09-22/README.md)
contains all 109 original-task outcomes, failed attempts, resource limits, and
the retained evidence bundle.

`scripts/run_witness_svcomp.py` runs a bounded batch of original SV-COMP tasks.
It reads expected verdicts, data models, inputs, and properties from task YAML,
records selection in `manifest.json`, and produces `results.json` and
`summary.csv` with per-stage logs, elapsed time, and sampled peak RSS. The default
selection takes five smallest unsafe unreach-call inputs from each requested
family; this is a functional sample, not a representative performance benchmark.
The runner requires PyYAML and process-table access for its memory limit.
Use `--backend btormc --btormc /path/to/btormc` to run BtorMC instead of rIC3.
Both backends produce the same standard witness input for back-translation.

```sh
python3 scripts/run_witness_svcomp.py \
  --benchmarks /path/to/sv-benchmarks/c \
  --c2btor build/bin/c2btor --ric3 ric3 \
  --btorsim btorsim \
  --cpachecker /path/to/CPAchecker/bin/cpachecker \
  --output-dir /tmp/new-svcomp-witness-run --per-family 5 --workers 2
```

Timeout, memory limit, conversion error, solver UNKNOWN, unexpected UNSAT,
translation error, and validation failure are reported separately. No SAFE
invariant witnesses are emitted. `--tasks-file` accepts relative task YAML paths
for rerunning a specified subset in a new output directory.

For a false-alarm audit, add `--expected-verdict true`. This selects tasks marked
safe without changing the converter or witness semantics. A safe label does not
prevent exporting a candidate from a circuit SAT result. Only subsequent C
validation can confirm that candidate; validator TRUE/UNKNOWN/error is not a
confirmed violation. `model_unsat` records a backend UNSAT result for a safe
task and does not generate a witness. `--engine portfolio` enables rIC3 portfolio
cross-checks. For local header configuration, repeat `--cbmc-arg=-I/path` as
needed; these arguments are saved with each conversion command.

References: [SV-COMP YAML violation witness 2.0 specification](https://sosy-lab.gitlab.io/benchmarking/sv-witnesses/yaml/violation-witnesses.html),
[Btor2Tools/BtorSim](https://github.com/Boolector/btor2tools),
[CPAchecker witness validation](https://www.sosy-lab.org/research/pub/2025-TACAS.CPAchecker_4.0_as_Witness_Validator.pdf).
