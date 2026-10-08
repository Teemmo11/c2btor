# Building C2BTOR on Linux x86-64

[English quick start](README.md) · [中文快速开始](README.zh-CN.md)

The supported release platform is Linux x86-64. Retained upstream platform
code and tests do not imply additional supported release platforms.

## Dependencies and default build

Use a C++17 compiler, CMake >= 3.8, Flex, Bison, Bash, patch and Make or Ninja.
On Ubuntu/Debian:

```sh
sudo apt-get update
sudo apt-get install build-essential cmake ninja-build flex bison patch git
cmake -S . -B build -G Ninja -DWITH_JBMC=OFF -DCMAKE_BUILD_TYPE=Release
cmake --build build --target c2btor -j4
build/bin/c2btor --version
```

Run these commands from the repository root. Use an out-of-source build:
configuring directly in the source directory is prohibited by CMake.
Lower `-j4` if the machine has limited RAM.

The default `sat_impl=minisat2` downloads MiniSat and applies the bundled
patch. Keep `scripts/minisat-2.2.1-patch` and `scripts/minisat2_CMakeLists.txt`.
The converter links the CBMC solver libraries even when an external BTOR2
backend is used later. Network/proxy errors during dependency download do
not mean the C2BTOR source is incomplete.

## Use a system MiniSat installation

To avoid the configure-time dependency download, first install the system
headers and library. On Ubuntu 24.04 these are supplied by `minisat`:

```sh
sudo apt-get install minisat
cmake -S . -B build-system -G Ninja -DWITH_JBMC=OFF \
  -DCMAKE_BUILD_TYPE=Release -Dsat_impl=system-minisat2
cmake --build build-system --target c2btor -j4
```

For a custom installation, use `CMAKE_PREFIX_PATH`. CMake expects
`minisat/simp/SimpSolver.h` and a linkable MiniSat library. Use a new build
directory when changing solver or compiler configurations.

## Optional witness and regression tools

Basic conversion does not require an external model checker or Python.
The single-task `scripts/c2btor_witness.py` wrapper uses only the Python
standard library. The batch runner `scripts/run_witness_svcomp.py` also
requires PyYAML (`python3 -m pip install PyYAML` in your Python environment).
Use Python 3.10+ for the maintained Python tools.
Install Btor2Tools (`catbtor`, `btorsim`), your chosen backend and CPAchecker
separately. Supply tool paths explicitly or put the tools on PATH; no
`checker/` directory is shipped. See [the regression guide](regression/goto-btor2/README.md).

JBMC and its Java submodule are not included; `WITH_JBMC` defaults to OFF.
The retained CBMC regression/unit infrastructure is separate from the
Python C2BTOR regression scripts. This release keeps its build dependencies
rather than removing libraries solely because they have a CBMC name.

## Optional local installation

```sh
cmake --install build --component c2btor --prefix /path/to/install
```

This installs the converter, man page, Bash completion, bilingual README,
license notices and witness scripts. Building/installing locally does not
add binaries to the Git source release.
