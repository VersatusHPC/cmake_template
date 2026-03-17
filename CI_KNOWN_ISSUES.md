# CI Known Issues

This document tracks CI limitations introduced by the migration from
CPM/FetchContent to Conan 2.0. The upstream template
([cpp-best-practices/cmake_template](https://github.com/cpp-best-practices/cmake_template))
does not have these issues because it uses FetchContent, which builds all
dependencies from source within the build tree.

## clang-tidy and Conan include paths (RESOLVED)

**Status:** Fixed by removing the `-p` flag from `StaticAnalyzers.cmake`.

**Root cause:** The `-p` flag in `CMAKE_CXX_CLANG_TIDY` options told CMake
to have clang-tidy use `compile_commands.json` instead of passing the full
compile command directly. This broke include resolution for Conan packages
whose headers live in external `-isystem` paths (`~/.conan2/p/...`). Without
`-p`, CMake appends `-- <full compile command>` to clang-tidy, which includes
all `-isystem` paths and works reliably with Conan.

## macOS + GCC excluded from CI

**Status:** Excluded via matrix exclude in `ci.yml`.

**Root cause:** GCC 14 on macOS uses `libstdc++`, but Conan's default macOS
profile builds packages with Apple's `libc++`. This causes ABI mismatches
at link time (undefined Catch2 symbols). With FetchContent, dependencies are
built with the same compiler and standard library, so no mismatch occurs.

**Workaround in place:** macOS CI validates Clang only. GCC is validated on
Linux (ubuntu-latest) and in the devcontainer (UBI 10).

**Proper fix:** Create a macOS-specific Conan profile for GCC that forces
`compiler.libcxx=libstdc++11` and builds all packages from source:

```ini
[settings]
compiler=gcc
compiler.version=14
compiler.libcxx=libstdc++11
```

Then pass `--profile:host=macos-gcc` to `conan install`.

## GCC coverage disabled on macOS

**Status:** Skipped in `cmake/Tests.cmake` when `APPLE AND GNU`.

**Root cause:** GCC's `--coverage` flag links against `libgcov`, which
Apple's linker (`ld`) can't find on ARM. This may be resolvable once the
macOS + GCC ABI issue is fixed (the linker error might be masking the
real problem).

**Proper fix:** Depends on the macOS + GCC fix above. If linking works
after the ABI fix, coverage may work too. Otherwise, GCC coverage on
macOS ARM is a known GCC limitation.

## codecov fail_ci_if_error set to false

**Status:** Set to `false` in `ci.yml`.

**Root cause:** The `CODECOV_TOKEN` secret is not configured on the
VersatusHPC repository.

**Proper fix:** Add `CODECOV_TOKEN` to the repository secrets at
`Settings > Secrets and variables > Actions`, then set
`fail_ci_if_error: true` back in `ci.yml`. The token can be obtained
from [codecov.io](https://codecov.io) after linking the repository.

## Intel ICX coverage skipped

**Status:** gcovr skipped when `matrix.compiler == intel` in `ci.yml`.

**Root cause:** Intel ICX produces coverage data in a format incompatible
with `gcov`. Tests still run and pass; only the coverage report is skipped.

**Proper fix:** Use Intel's own coverage tools (`llvm-cov` from the oneAPI
toolkit) or `llvm-profdata` to process ICX coverage data. This is a
nice-to-have, not a blocker.
