# CI/CD Configuration

The CI pipeline is configurable via `.github/ci-config.yml`. Users can
toggle which compilers are tested on each platform without editing
workflow files.

## Configuration file

Edit `.github/ci-config.yml` to enable or disable compilers:

```yaml
linux:
  gcc-14: true       # System GCC (UBI 10)
  gcc-15: true       # GCC Toolset 15
  clang: true        # System Clang (version depends on UBI minor release)
  intel: true        # Intel ICX/ICPX (oneAPI)

macos:
  llvm: true         # LLVM/Clang via setup-cpp
  gcc-14: true       # Homebrew GCC 14

windows:
  msvc: true         # Visual Studio 17 2022
  llvm: true         # Clang-cl via LLVM
```

Set a compiler to `false` to disable it. Disabling all compilers on a
platform skips that platform entirely (the job won't run).

## How it works

The `setup` job in `ci.yml` reads the config file and produces dynamic
matrix values using `yq` and `jq`. Each platform job uses
`fromJSON()` to build its matrix from the setup outputs.

Per-compiler settings (CC, CXX, gcov executable, IPO) are resolved in
each job's "Set compiler environment" step using a `case` statement,
not matrix `include` entries. This ensures disabling a compiler
cleanly removes it from the matrix without creating partial entries.

### Linux

Runs inside UBI 10 container images from GHCR. The matrix is a
cross-product of `compiler x build_type x packaging_maintainer_mode`.
A separate `Linux-Make` job tests Unix Makefiles with gcc-14 (only
runs when gcc-14 is enabled).

### macOS

Uses bare `macos-latest` runners with `aminya/setup-cpp`. The LLVM
version is controlled by the `CLANG_TIDY_VERSION` env var in `ci.yml`
(not the config file). The config maps `llvm` to `llvm-<version>`
automatically.

### Windows

Uses bare `windows-latest` runners with `aminya/setup-cpp`. The
Windows matrix is generated as a full JSON array of include objects
in the setup job, so each entry carries its generator, IPO setting,
and package generator. This avoids the GitHub Actions limitation
where matrix includes for disabled compilers create partial entries.

## Adding a new compiler

1. Add the toggle to `.github/ci-config.yml`
2. Add a `case` branch in the "Set compiler environment" step of the
   relevant platform job in `ci.yml`
3. For Windows, add the matrix entries in the setup job's jq script
4. For Linux, if the compiler needs a different container image, update
   the container selection logic

## Container images

Linux CI jobs run inside pre-built UBI 10 containers:

- `ghcr.io/versatushpc/cmake_template/ci:latest` — GCC 14/15, Clang,
  cmake, ninja, ccache, conan, cppcheck, IWYU, Bloaty, gcovr, lizard
- `ghcr.io/versatushpc/cmake_template/ci-intel:latest` — extends
  ci:latest with Intel oneAPI DPC++/C++ compiler

By default, derived repos use the template's pre-built images. To use
custom images, uncomment the per-repo image computation in the setup
job and re-enable the `build-ci-image.yml` workflow.

## Known issues

- **macOS GCC coverage** ([#5](https://github.com/VersatusHPC/cmake_template/issues/5)):
  Apple ARM linker can't find libgcov. Tests pass, coverage skipped.
- **Doxygen not in CI** ([#9](https://github.com/VersatusHPC/cmake_template/issues/9)):
  Not available in UBI 10 repos. Only needed for the `docs` target.
- **IWYU on GCC/Intel**: Disabled. IWYU is Clang-based and rejects
  GCC-specific warning flags with `-Werror`. Enabled on Clang only.
- **IPO/LTO on Linux GCC**: Disabled. Conan-built dependencies lack
  LTO objects, causing link failures.
