# C2BTOR 2.0 source preparation / 源码整理记录

## Provenance / 版本来源

- Source repository: `westtide/cbmc`, tag `2.0`.
- Exact source commit: `4b94c0095bfe08bfed221cb7665a225a28d18eb8`.
- Source commit subject: `release: publish C2BTOR 2.0 product interface and documentation`.
- Product version: `2.0.0`; executable and build target: `c2btor`.
- Destination: `westtide/c2btor`, a source snapshot on `release/2.0-prep`.
- Release platform: Linux x86-64. No prebuilt checker or converter is bundled.

The source snapshot comes from the tag's Git tree, not from a local ZIP or
the latest upstream branch. The destination retains its own Git history.
This preparation does not change the C-to-BTOR2 conversion implementation.

源码直接提取自 2.0 的 Git 树，保留目标仓库自己的 Git 历史。此次整理未改变转换语义。

## Cleanup / 清理范围

Omitted from the destination source snapshot:

- `checker/`: external prebuilt tools, including macOS binaries.
- `frontier/` and `frontier.zip`: independent experimental material.
- `sv-log/`, copied `test/loop_examples/` benchmarks, local analysis/notes,
  generated corner-case models, logs, witnesses, and Python caches.
- LaTeX `.aux`, `.log`, `.toc` and generated PDF; the `.tex` source remains.
- Inherited CBMC/Java/multi-platform publishing workflows, private ownership
  configuration, hooks, release planning notes and the old Dockerfile.
- `.gitmodules`: the referenced JBMC Java subtree is not present in this tag.

以上内容从待发布快照中排除，原始源仓库及历史实验文件仍保留，便于追溯。

Retained: all `src/` modules, build scripts and patches, upstream regression
and unit fixtures, C2BTOR conversion regression scripts, witness tools,
curated dated evidence, technical documentation, and copyright/license
notices. `src/goto-checker` is a linked dependency and is retained. Five binary-format input fixtures (`Hello.class`, `hello_elf`,
`hello_elf_with_gbf`, `hello_fat_macho`, `hello_fat_macho_with_gbf`)
are retained in `unit/goto-programs/` because the binary-reader tests
reference them. They are not prebuilt converter or checker releases.

## Preparation changes / 整理改动

- English README plus linked Chinese README, runnable examples and option explanations.
- Linux build instructions and a conversion regression guide.
- Normalized CRLF shell scripts to LF and preserved Linux checkout rules.
- Materialized two upstream man-page symlinks as ordinary files for Windows portability.
- Scoped ignore rules for builds, caches and local runs; test sources stay visible.
- JBMC defaults to OFF because its source is absent; a clear diagnostic explains an ON request.
- Removed hard-coded bundled checker paths from regression defaults; use installed tools.
- Component installation includes both READMEs and both license notices.
- Updated product issue contact to the destination repository.

The upstream 4-clause BSD notice remains at `LICENSE`; the destination's
original 3-clause notice remains at `LICENSE.c2btor`. File-specific notices
are retained. This is not a relicensing of upstream code.

Original executable modes are preserved for Linux checkouts. Validation
results are reported separately; historical evidence is not treated as
a current build or test pass.
