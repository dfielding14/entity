# AGENTS.md

## Scope
This file covers `/cmake` only.

Use with:
- `/Users/dbf75/Work/Research/AthenaK/entity/AGENTS.md`

If instructions conflict, follow the closest `AGENTS.md` to the edited file.

---

## What `cmake/` owns
`/cmake` defines build configuration policy and build-mode wiring used by
`/Users/dbf75/Work/Research/AthenaK/entity/CMakeLists.txt`:
1. Defaults and option validation.
2. Dependency discovery/fetch/submodule fallback.
3. Tests and benchmark build graph wiring.
4. Configure-time report formatting.

This directory is build-control code. Small changes can affect all build modes.

---

## Directory map
- `/Users/dbf75/Work/Research/AthenaK/entity/cmake/defaults.cmake`
  Default values, including env-var-driven defaults (`Entity_ENABLE_*`).
- `/Users/dbf75/Work/Research/AthenaK/entity/cmake/config.cmake`
  Option helpers: precision, shape-order validation, pgen resolution and
  include path wiring.
- `/Users/dbf75/Work/Research/AthenaK/entity/cmake/dependencies.cmake`
  `find_or_fetch_dependency(...)` and online/offline behavior.
- `/Users/dbf75/Work/Research/AthenaK/entity/cmake/kokkosConfig.cmake`
  Kokkos build toggles that depend on debug/testing mode.
- `/Users/dbf75/Work/Research/AthenaK/entity/cmake/adios2Config.cmake`
  ADIOS2 options used when ADIOS2 is fetched/built.
- `/Users/dbf75/Work/Research/AthenaK/entity/cmake/tests.cmake`
  CTest enablement and module test subdirectory selection.
- `/Users/dbf75/Work/Research/AthenaK/entity/cmake/benchmark.cmake`
  Benchmark target wiring.
- `/Users/dbf75/Work/Research/AthenaK/entity/cmake/styling.cmake`
  Color/format helper functions used by configure report.
- `/Users/dbf75/Work/Research/AthenaK/entity/cmake/report.cmake`
  Human-readable configure summary and compiler/dependency notes.

---

## Configure flow (high level)
Main configure pipeline is in `/Users/dbf75/Work/Research/AthenaK/entity/CMakeLists.txt`:
1. Load styling helpers and defaults.
2. Materialize cache options (`DEBUG`, `precision`, `deposit`, `shape_order`,
   `pgen`, `gui`, `output`, `mpi`, `gpu_aware_mpi`).
3. Apply validators/helpers from `config.cmake`.
4. Resolve dependencies via `find_or_fetch_dependency(...)`.
5. Set compile macros from feature/device choices (`MPI_ENABLED`,
   `OUTPUT_ENABLED`, `DEVICE_ENABLED`, `CUDA_ENABLED`, `HIP_ENABLED`,
   `SYCL_ENABLED`, `GPU_AWARE_MPI`).
6. Select build mode:
   - `TESTS` -> include `tests.cmake`
   - `BENCHMARK` -> include `benchmark.cmake`
   - otherwise build main executable with `set_problem_generator(...)` + `src/`
7. Emit configure report via `report.cmake`.

---

## Option semantics and constraints

### Core cache options
- `precision=single|double`
  `single` adds `-DSINGLE_PRECISION`.
- `deposit=zigzag|esirkepov`
  Governs shape-order handling.
- `shape_order`
  Applied only for `deposit=esirkepov` (`-DSHAPE_ORDER=<n>`).
- `pgen`
  Required for normal executable builds (must not be `"."`).
- `output=ON|OFF`
  Enables ADIOS2 paths and `-D OUTPUT_ENABLED`.
- `mpi=ON|OFF`
  Enables MPI and `-D MPI_ENABLED`.
- `gpu_aware_mpi=ON|OFF`
  Relevant only when MPI + device backend are enabled.
- `gui=ON|OFF`
  Optional GUI dependency path.
- `TESTS`, `BENCHMARK`
  Mode switches; `TESTS` branch takes priority over `BENCHMARK`.

### Env-var defaults
`defaults.cmake` reads these when present:
- `Entity_ENABLE_DEBUG`
- `Entity_ENABLE_OUTPUT`
- `Entity_ENABLE_GUI`
- `Entity_ENABLE_MPI`
- `Entity_ENABLE_GPU_AWARE_MPI`

### Non-obvious behavior
- With `deposit=zigzag`, `shape_order` is reset to default behavior.
- `set_shape_order(...)` enforces `shape_order <= 11` for `esirkepov`.
- `set_problem_generator(...)` supports:
  - in-tree pgens (`pgens/<name>`)
  - external pgens (`-D pgen=pgens/<name>` -> `extern/entity-pgens/<name>`)
  - explicit filesystem path.

---

## Dependency resolution contracts
Defined in `/Users/dbf75/Work/Research/AthenaK/entity/cmake/dependencies.cmake`:
- `check_internet_connection()`:
  - honors `-D OFFLINE=ON`
  - otherwise pings `8.8.8.8` and sets `FETCHCONTENT_FULLY_DISCONNECTED`.
- `find_or_fetch_dependency(package, header_only, mode)`:
  1. `find_package(...)` (unless `header_only`).
  2. If missing and online, `FetchContent` from repository (pinned tags for
     Kokkos and ADIOS2).
  3. If missing and offline/disconnected, fallback to `extern/<package>`
     subdirectory.

Keep this fallback order intact unless changing reproducibility policy.

---

## Tests and benchmark wiring

### Tests (`tests.cmake`)
- Always adds core libraries from `src/*`.
- Test subdirectories are mode-gated:
  - non-MPI: global, metrics, kernels, archetypes, framework, output
  - MPI+output: framework, output
  - output tests are always appended in current wiring.

### Benchmark (`benchmark.cmake`)
- Builds `benchmark.xc` and links core libs.
- When `output=ON`, benchmark wiring includes additional output-related
  subdirectories.
- Keep benchmark subdirectory paths aligned with current source layout when
  editing this file.

---

## Report and styling
- `styling.cmake` defines color and formatting helpers used by configure output.
- `report.cmake` prints selected options, compiler info, Kokkos/ADIOS versions,
  and test hints.
- If you add user-facing CMake options, update report output so configuration is
  inspectable at configure time.

---

## Change guidance

### Safe change points
1. Add/adjust default values in `defaults.cmake`.
2. Extend option validation in `config.cmake`.
3. Add dependency handling in `dependencies.cmake`.
4. Adjust mode wiring in `tests.cmake`/`benchmark.cmake`.

### High-risk edits
- Renaming cache options consumed by code/docs/CI.
- Changing dependency fallback order (find/fetch/submodule).
- Changing feature macro emission in root `CMakeLists.txt`.
- Modifying pgen resolution semantics.

---

## Recommended validation
Run at least one configure per mode after CMake edits:
```bash
cmake -B build -D pgen=streaming -D output=ON -D mpi=OFF
```

```bash
cmake -B build -D TESTS=ON -D output=ON -D mpi=OFF
```

```bash
cmake -B build -D BENCHMARK=ON -D output=OFF
```

For dependency-path changes, also test offline behavior:
```bash
cmake -B build -D OFFLINE=ON -D pgen=streaming
```

---

## Common failure modes
- Forgetting `-D pgen=...` for normal (non-TESTS, non-BENCHMARK) builds.
- Assuming `shape_order` applies when `deposit=zigzag`.
- Enabling `output=ON` without ADIOS2 availability and without initialized
  externals/fetch path.
- Breaking parent-scope propagation for values used by the root configure flow.
- Updating tests/benchmark wiring without matching module directory reality.
