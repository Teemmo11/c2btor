# C2BTOR 2.0

[English](README.md) · [简体中文](README.zh-CN.md)

C2BTOR translates C programs into bit-precise [BTOR2](https://github.com/hwmcc/btor2tools) transition systems. An external model checker can then search for assertion failures or reachable error calls. C2BTOR can also map a standard BTOR2 counterexample back to a C violation witness for independent validation.

The converter reuses the [CBMC](https://github.com/diffblue/cbmc) C frontend, target machine configuration, GotoIR and program transformations. Its executable is **`c2btor`**, version **`2.0.0`**. This source release targets **Linux x86-64**; external checkers and prebuilt binaries are not bundled.

## Build from source

You need a C++17 compiler, CMake 3.8 or newer, Make or Ninja, Flex, Bison, Bash and `patch`. On Ubuntu/Debian:

```sh
sudo apt-get update
sudo apt-get install build-essential cmake ninja-build flex bison patch git
git clone https://github.com/westtide/c2btor.git
cd c2btor
cmake -S . -B build -G Ninja -DWITH_JBMC=OFF -DCMAKE_BUILD_TYPE=Release
cmake --build build --target c2btor -j4
build/bin/c2btor --version
```

CMake downloads and patches MiniSat during configuration, so the default build needs network access. For an installed MiniSat alternative and troubleshooting, see [COMPILING.md](COMPILING.md). JBMC is not included. Build the `c2btor` target explicitly; the retained upstream source tree also defines auxiliary CBMC tools.

## Try a small program

From the repository root, create a program with one assertion:

```sh
mkdir -p work
cat > work/example.c <<'EOF'
extern int __VERIFIER_nondet_int(void);
int main(void) {
  int x = __VERIFIER_nondet_int();
  __CPROVER_assume(x >= 0 && x <= 10);
  __CPROVER_assert(x < 10, "x must be below 10");
  return 0;
}
EOF
build/bin/c2btor work/example.c --goto-btor2 --inline --64 \
  --goto-btor2-out work/example.btor2 \
  --goto-btor2-map-out work/example.map.json \
  --no-standard-checks --no-pointer-check \
  --no-bounds-check --no-built-in-assertions
```

This produces a model and a source map. The assertion fails when `x == 10`; conversion itself does not report whether it is reachable. If you have installed Btor2Tools, run `catbtor work/example.btor2` to check the model's syntax and types. Then pass the model to your chosen BTOR2 model checker. A parser success is not a verification result.

The example selects source assertions and disables CBMC's standard safety instrumentation. C2BTOR's own memory validity and model limit diagnostics remain. `--64` selects Linux LP64; use `--32` for ILP32 tasks. Neither option makes every integer 64 or 32 bits.

## Choose the options you need

Pass values with a space, for example `--memory object`. The conversion options below require `--goto-btor2`.

| Option | Meaning and when to use it |
| --- | --- |
| `--goto-btor2` | Generate a BTOR2 transition system instead of running ordinary CBMC verification. |
| `--32` / `--64` | Select the C target data model: ILP32 or Linux LP64. Match the program/task rather than the build machine. |
| `--no-standard-checks` | Disable CBMC's default safety instrumentation. It does not remove source assertions or C2BTOR's internal diagnostics. |
| `--goto-btor2-out FILE` | Save the model to a file. Without it, the model is written to stdout. |
| `--goto-btor2-map-out FILE` | Save the connection between model nodes, GotoIR and C source locations, including property and model metadata. Use a different file from the model. |
| `--inline` | Inline ordinary function calls before conversion. Use it for programs that call helper functions; recursive call stacks are unsupported. |
| `--memory object` / `--memory global` | Choose separate storage for each object or one global storage model. Default: `object`. |
| `--array bv` / `--array array` | Encode storage as bitvectors or BTOR2 arrays. Default: `bv`. This changes the representation, not the intended memory semantics. |
| `--memory-object-max-bytes N` | Cap runtime object storage in object mode when you need a smaller, explicit model boundary. Without a cap, capacity is inferred from the final GotoIR. Exceeding it is a model limit. |
| `--array-bv-max-object-bytes N` | Set the runtime object cap for BV storage. In object/BV mode, if both capacity options are supplied, they must agree. |
| `--goto-btor2-heap-objects K` | Bound the total number of successful dynamic allocations. Default: `32`. Freeing memory does not return an allocation identity to this budget. |
| `--goto-btor2-checks` | Insert the selected CBMC safety checks before conversion. For example, combine `--no-standard-checks --bounds-check --goto-btor2-checks` to request array bounds checks without enabling all standard checks. |
| `--goto-btor2-error-function NAME` | Turn calls to an error function, such as `reach_error`, into unreach-call properties before inlining. Combine with `--inline`. |
| `--goto-btor2-reach-only` | Select only the source properties for that error function. Requires `--goto-btor2-error-function NAME`. |
| `--goto-btor2-merge-bads` | OR the selected source properties into one bad state for backends expecting one property. Alias: `--goto-btor2-merge-properties`. |
| `--goto-btor2-no-heap-guards` | Hide auxiliary memory/model-limit bad reports. Path freezing and proof obligations remain; this option alone does not justify a C SAFE result. |

For a complete error-call example, create a second program. Its
`reach_error()` call is reachable when `x == 10`:

```sh
cat > work/reach.c <<'EOF'
extern int __VERIFIER_nondet_int(void);
void reach_error(void) { __CPROVER_assert(0, "error reached"); }
int main(void) {
  int x = __VERIFIER_nondet_int();
  if(x == 10) reach_error();
  return 0;
}
EOF
build/bin/c2btor work/reach.c --goto-btor2 --inline --64 \
  --goto-btor2-error-function reach_error --goto-btor2-reach-only \
  --goto-btor2-merge-bads --goto-btor2-out work/reach.btor2 \
  --no-standard-checks --no-pointer-check \
  --no-bounds-check --no-built-in-assertions
```

Keep `--goto-btor2-error-function` equal to the actual error function name. To include malloc's NULL failure branch, add `--malloc-may-fail --malloc-fail-null`. Run `build/bin/c2btor --help` for the complete interface.

## Properties, source maps and witnesses

A BTOR2 `bad` describes a violation condition. Source assertions, `memory_validity` diagnostics and `model_limit` diagnostics have different meanings: a storage limit is not a source-level error-call counterexample. A bounded search without a counterexample does not prove unbounded safety. A C SAFE claim also requires checking the relevant model obligations and support boundaries.

The CLI source map helps inspect properties and C locations. For witness export, use the maintained wrapper to bind the source and model hashes:

```sh
cat > work/reach.prp <<'EOF'
CHECK( init(main()), LTL(G ! call(reach_error())) )
EOF
python3 scripts/c2btor_witness.py export \
  --c2btor build/bin/c2btor --program work/reach.c \
  --spec work/reach.prp --output-dir work/witness-export -- --64
```

This reuses `work/reach.c` from the preceding example and exports a model
and hash-bound map; it does not run a solver or produce a counterexample.
The final output directory must be new. The wrapper's default property is reachability of `__VERIFIER_error`; another error function can be selected through `--spec property.prp`. The maintained witness path is BTOR2 counterexample → BtorSim replay → SV-COMP violation YAML 2.0 → CPAchecker validation. A raw CLI map is not a hash-bound witness bundle. See [the witness guide](docs/c2btor-witness.md) for translation, validation, dependencies and supported nondeterministic values.

## Support and limitations

The main entry point is sequential C with `main`. The converter models machine integers, control flow, finite object memory and supported binary32/binary64 floating-point operations. Default storage is `--memory object --array bv`; fixed objects use their exact size, while runtime object capacity is planned at model generation time.

Recursive call stacks, concurrency and unbounded heaps are unsupported. Some bitfield memory accesses, pointer/integer address conversions, repeated declarations of address-taken automatic objects and reads of undetermined bytes are limited. The full C provenance/effective-type rules and floating-point exception environment are not implemented. C witness translation currently covers unreach-call with supported scalar integer/Boolean nondeterministic values; SAFE invariant witnesses are not generated. See the detailed documents before drawing conclusions beyond these bounds.

## Documentation and regression checks

| Resource | What it explains |
| --- | --- |
| [Object memory](docs/c2btor-object-memory.md) | Object identity, aliases, byte storage and lifetime. |
| [Array encoding](docs/c2btor-array-encoding.md) | Bitvector versus array storage. |
| [Capacity planning](docs/c2btor-memory-capacity.md) | Runtime storage capacity and model boundaries. |
| [Floating-point design](docs/ieee754_kratos_analysis.md) | IEEE-754 lowering and supported operations. |
| [Regression guide](regression/goto-btor2/README.md) | How to run the retained conversion tests with separately installed tools. |
| [Release preparation](RELEASE_PREPARATION.md) | The exact 2.0 source commit, cleanup scope and retained dependencies. |

`src/` contains C2BTOR and the reused CBMC build dependencies; `scripts/` contains witness tools; `regression/goto-btor2/` contains conversion tests. Upstream tests, copyright headers and license notices are retained.
Five binary-format files in `unit/goto-programs/` are parser test inputs;
they are retained to keep those tests complete and are not shipped converter/checker tools. Dated validation documents are historical evidence, not fresh results for your checkout. Report issues at [westtide/c2btor](https://github.com/westtide/c2btor/issues).

## Acknowledgements and license

C2BTOR builds on CBMC/CProver. This product includes software developed by Daniel Kroening, Edmund Clarke, Computer Science Department, University of Oxford, Computer Science Department, Carnegie Mellon University.

The imported source retains the [upstream 4-clause BSD license](LICENSE) and file-specific notices. [LICENSE.c2btor](LICENSE.c2btor) preserves the destination repository's original BSD 3-Clause notice; it does not replace the imported source's terms. External tools such as [Btor2Tools](https://github.com/hwmcc/btor2tools), [Boolector/BtorMC](https://github.com/Boolector/boolector) and [CPAchecker](https://github.com/sosy-lab/cpachecker) are installed separately under their own licenses.
