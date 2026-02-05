# AGENTS.md

## Purpose
This file gives local guidance for `src/global/`, the base layer that most
other modules include. Use it when editing global types, enums, portability
aliases, logging/error helpers, and low-level utilities.

---

## Scope of `src/global/`

### What lives here
- `global.h`, `global.cpp`
  Core aliases, flags, macros, and global runtime init/finalize.
- `defaults.h`
  Default simulation parameter values (`ntt::defaults::*`).
- `enums.h`
  String-backed enum wrappers (`Coord`, `Metric`, `SimEngine`, BC/output IDs).
- `arch/`
  Kokkos and MPI abstraction helpers (`kokkos_aliases`, `mpi_aliases`,
  `mpi_tags`, `directions`, compile-time traits).
- `utils/`
  Error/logging, formatting, numeric/comparator helpers, timers/diagnostics,
  command-line args, and parameter container.
- `tests/`
  Unit tests for the module (`GLOBAL::*` CTest names).

### Cross-module role
`src/framework`, `src/engines`, `src/kernels`, and `src/archetypes` depend on
this directory for fundamental types, logging/errors, and architecture wrappers.
Changes here are high-impact and should stay conservative.

---

## Core Contracts

### Global lifecycle
- `ntt::GlobalInitialize(argc, argv)` initializes Kokkos, then MPI (when
  `MPI_ENABLED`).
- `ntt::GlobalFinalize()` finalizes MPI first (when enabled), then Kokkos.
- Do not add secondary init/finalize paths in leaf modules. Keep lifecycle
  centralized through `Simulation` and top-level tests.

### Types and precision
- `real_t` is the project scalar type (`float` for `SINGLE_PRECISION`,
  otherwise `double`).
- `N_GHOSTS` is derived from `SHAPE_ORDER`; maintain this relation when
  changing deposition/ghost-cell assumptions.
- Prefer existing aliases from `global.h` (`coord_t`, `vec_t`, `npart_t`,
  `ncells_t`, `timestep_t`, `path_t`, etc.) over raw primitives.

### Enum wrapper pattern (`enums.h`)
Each enum wrapper follows this shape:
- `enum type : uint8_t` with `INVALID = 0`.
- `variants[]`, `lookup[]`, and `total`.
- `to_string()`, `contains()`, and `pick()`.

When adding/updating enum values:
- Keep `variants`, `lookup`, and `total` in sync.
- Keep lookup strings lowercase to match existing parsing behavior.
- Update unit tests in `src/global/tests/enums.cpp`.

### Error and logging conventions
- Use `raise::ErrorIf`, `raise::Error`, `raise::Fatal`, `raise::Warning`.
- Always pass `HERE` for file/function/line context.
- For device-side failures, use `raise::KernelError` /
  `raise::KernelNotImplementedError`.
- Use `logger::Checkpoint(...)` for milestone logging instead of ad hoc prints.
- Use `CallOnce(...)` for rank-0-only behavior to avoid MPI log spam.

### MPI/Kokkos guard conventions
- Wrap MPI-specific code with `#if defined(MPI_ENABLED)`.
- Wrap ADIOS2 parameter writing with `#if defined(OUTPUT_ENABLED)`.
- Keep host/device-safe behavior in `KOKKOS_LAMBDA` and `Inline` code paths.

---

## Subdirectory Notes

### `arch/`
- `kokkos_aliases.h` defines standard aliases/macros used throughout the code:
  `Lambda`, `ClassLambda`, `Function`, `Inline`, `array_t`, `ndfield_t`,
  `range_t`, `CreateRangePolicy`, and RNG aliases.
- `mpi_aliases.h` provides `CallOnce` and `mpi::get_type<T>()`.
- `directions.h` and `mpi_tags.h` encode logical directions and particle send
  tags; keep direction ordering stable because tag mapping depends on it.
- `traits.h` is shared compile-time detection infrastructure; add traits here
  when new optional pgen hooks are introduced.

### `utils/`
- `log.h` and `plog.h` define logging semantics and output file handling.
- `error.h` is the canonical failure path for host and device code.
- `timer.*`, `diag.*`, `progressbar.*` drive runtime diagnostics output.
- `param_container.h` is the typed parameter store used by framework params.
  If introducing a new stored type, ensure retrieval/stringization behavior is
  covered; for output mode also update ADIOS write registration in
  `param_container.cpp`.
- `toml.h` is an in-tree TOML library header; treat it as vendored unless the
  task explicitly targets TOML library updates.

---

## Change Guidance

### Editing checklist
- Keep changes minimal and localized; this module has broad include reach.
- Preserve compile-time/static assertions and template constraints.
- Reuse existing helpers before adding new utility layers.
- Do not bypass `raise::*` or `logger::*` with inconsistent error/print logic.
- If you add new `.cpp` compilation units, update `src/global/CMakeLists.txt`.

### Common change entry points
- Add/adjust core aliases or flags: `src/global/global.h`
- Change global startup/shutdown behavior: `src/global/global.cpp`
- Add default parameter values: `src/global/defaults.h`
- Add enum options/parsing: `src/global/enums.h`
- Add Kokkos/MPI portability helpers: `src/global/arch/*.h`/`*.cpp`
- Add diagnostics/timer/logging helpers: `src/global/utils/*.h`/`*.cpp`

---

## Testing

### Current module tests
`src/global/tests/CMakeLists.txt` defines:
- `GLOBAL::global`
- `GLOBAL::enums`
- `GLOBAL::kokkos_aliases`
- `GLOBAL::directions`
- `GLOBAL::comparators`
- `GLOBAL::numeric`
- `GLOBAL::param_container`
- `GLOBAL::sorting`

### Recommended test workflow
```bash
cmake -B build -D TESTS=ON -D mpi=OFF -D output=ON
cmake --build build -j "$(nproc)"
ctest --test-dir build --output-on-failure -R '^GLOBAL::'
```

If you touch MPI-conditioned logic, also run at least one `-D mpi=ON` variant.
If you touch `param_container.cpp` or ADIOS writing paths, ensure `-D output=ON`.

---

## When Unsure
Prefer existing patterns in `src/global` tests and call sites over introducing
new conventions. If a change alters behavior consumed by other modules, document
the impact in the same PR/commit.
