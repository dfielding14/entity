# AGENTS.md

## Purpose
This file orients automated agents working in Entity. It summarizes the codebase
layout and the expected approach to implementing changes. Use it as a map, but
verify details in the source before editing or documenting behavior.

---

## Codebase Overview

### What Entity is
Entity is a C++17, Kokkos-based particle-in-cell (PIC) code for relativistic
plasma astrophysics. It supports both SRPIC and GRPIC engines, optional MPI
domain decomposition, and optional ADIOS2-based output/checkpointing.

Core traits of this codebase:
- Heavily templated compile-time specialization over engine, metric, and
  dimension.
- Runtime behavior configured via TOML input files.
- Problem-specific physics is injected through pluggable problem generators
  (`pgen.hpp`).

### Execution flow (high level)
1. `src/entity.cpp` is the executable entrypoint.
2. `ntt::Simulation` (`src/framework/simulation.cpp`) parses CLI flags
   (`-input`, resume aliases), initializes Kokkos/MPI, and loads TOML params.
3. Requested engine/metric/dimension are matched against
   `src/framework/specialization_registry.h`.
4. `EngineSelector` (`src/engines/engine_traits.h`) maps to `SRPICEngine` or
   `GRPICEngine`.
5. `Engine::init()` (`src/engines/engine_init.cpp`) builds metadomain state,
   initializes optional output/checkpoint writers, and runs pgen init hooks.
6. `Engine::run()` (`src/engines/engine_run.cpp`) executes the timestep loop:
   engine step, optional pgen post-step hooks, diagnostics, output, checkpoint.
7. Shutdown finalizes MPI/Kokkos via `GlobalFinalize()`.

### Source tree map (core areas)
- `src/`
  Main implementation (see `src/AGENTS.md`).
- `src/global/`
  Fundamental types, enums, defaults, logging, error handling, utilities
  (see `src/global/AGENTS.md`).
- `src/framework/`
  `SimulationParams`, domain/metadomain orchestration, containers, MPI comms
  (see `src/framework/AGENTS.md`).
- `src/engines/`
  Engine base and SRPIC/GRPIC step algorithms (see `src/engines/AGENTS.md`).
- `src/kernels/`
  Low-level computational kernels used by engines (see `src/kernels/AGENTS.md`).
- `src/metrics/`
  Coordinate/metric implementations and transforms
  (see `src/metrics/AGENTS.md`).
- `src/output/`
  Stats/output/checkpoint wiring (when `output=ON`)
  (see `src/output/AGENTS.md`).
- `src/archetypes/`
  Reusable pgen helpers (injectors, distributions, field setters)
  (see `src/archetypes/AGENTS.md`).
- `pgens/`
  Built-in problem generators and sample `.toml` setups
  (see `pgens/AGENTS.md`).
- `cmake/`
  Build configuration, defaults, dependency resolution, tests wiring
  (see `cmake/AGENTS.md`).
- `minimal/`
  Third-party sanity checks (Kokkos/MPI/ADIOS2 outside full Entity runtime)
  (see `minimal/AGENTS.md`).
- `benchmark/`
  Benchmark entry scaffold.
- `dev/`
  Development containers, Nix shell definitions, runner tooling
  (see `dev/AGENTS.md`).
- `extern/`
  Submodules and external pgen repository hooks.

### Subdirectory AGENTS files (completed index)
The rollout index is complete. The following subdirectory guides are available:

1. `src/AGENTS.md`
   Cross-module architecture map, compile-time specialization model, and module boundaries.
2. `src/global/AGENTS.md`
   Core aliases/enums/utilities, logging/error conventions, and global init/finalize.
3. `src/framework/AGENTS.md`
   Parameter lifecycle, simulation orchestration, domain ownership, and MPI/output integration points.
4. `src/framework/domain/AGENTS.md`
   Domain decomposition, communication patterns, and checkpoint/output hooks.
5. `src/framework/containers/AGENTS.md`
   Particle/field container layouts, ownership semantics, and IO/comm coupling.
6. `src/engines/AGENTS.md`
   SRPIC/GRPIC stepping pipelines, expected kernel call order, and extension points.
7. `src/kernels/AGENTS.md`
   Kernel contracts, Kokkos execution/memory-space assumptions, and testing strategy.
8. `src/metrics/AGENTS.md`
   Metric interfaces, coordinate transforms, and dimension/engine compatibility.
9. `src/output/AGENTS.md`
   Writer/checkpoint/stats behavior, ADIOS2 assumptions, and output integration.
10. `src/archetypes/AGENTS.md`
    Reusable setup primitives and how pgens should compose them.
11. `pgens/AGENTS.md`
    `user::PGen` patterns, compatibility traits, common hooks, and per-setup layout.
12. `cmake/AGENTS.md`
    CMake option semantics, dependency resolution flow, and tests wiring.
13. `dev/AGENTS.md`
    Container/Nix dev environments, runner usage, and expected local workflows.
14. `minimal/AGENTS.md`
    Third-party validation binaries and when to use each minimal mode.

Guidance hierarchy:
- Use this root file for global rules and cross-module context.
- Use the closest `AGENTS.md` to the files being edited for local conventions.
- If instructions conflict, pause and ask for clarification before making changes.

### `extern/` dependencies and submodules
`extern/` is not optional for a stable, reproducible checkout. The repository
tracks these submodules in `.gitmodules`:
- `extern/Kokkos`
  Performance portability backend used across the codebase.
- `extern/adios2`
  Output/checkpoint backend when `output=ON`.
- `extern/plog`
  Logging library used by runtime and tests.
- `extern/entity-pgens`
  Additional problem generators outside the core `pgens/` tree.

If submodules are missing, initialize/sync them from repo root:
```bash
git submodule sync --recursive
git submodule update --init --recursive
```

Quick health check:
```bash
git submodule status --recursive
```

Build-time dependency resolution behavior (`cmake/dependencies.cmake`):
- First tries `find_package(...)` for installed system packages.
- If not found and online, may fetch with `FetchContent`.
- If fetch is unavailable/offline, falls back to `extern/<dep>` submodules.

For external problem generators from `extern/entity-pgens`, select them with:
- `-D pgen=pgens/<name>`

### Key abstractions
- `ntt::Simulation`
  Top-level runtime setup and specialization dispatch.
- `ntt::SimulationParams`
  Parameter container with immutable/mutable/checkpoint-aware splits.
- `ntt::Metadomain<S, M>` and `ntt::Domain<S, M>`
  Global decomposition plus local domain data and communication operations.
- `ntt::Engine<S, M>`
  Common engine lifecycle; specialized by `SRPICEngine` and `GRPICEngine`.
- `arch::ProblemGenerator<S, M>` and `user::PGen<S, M>`
  Extension point for setup-specific initial conditions and custom hooks.
- Specialization registry/macros
  Compile-time allowed engine/metric/dimension combinations.

---

## Implementation & Change Guidelines

### Approach to changes
- Be direct and pragmatic: small, incremental changes that build and test cleanly.
- Prefer clarity over cleverness; if it needs a long explanation, simplify.
- Study nearby code patterns first; mirror existing interfaces and data layouts.
- Avoid premature abstractions; add only what the current change needs.
- Preserve compile-time compatibility checks and static assertions.
- Keep optional-feature guards consistent (`MPI_ENABLED`, `OUTPUT_ENABLED`,
  `CUDA_ENABLED`, `HIP_ENABLED`, etc.).
- Use existing error/reporting helpers (`raise::ErrorIf`, `raise::Fatal`,
  `logger::Checkpoint`) instead of ad hoc control flow.
- Treat `extern/*` as vendored third-party code:
  do not patch submodule contents unless the task explicitly requires it.
  Prefer changes in Entity wrappers/config first.

### Where to implement specific changes
- New runtime parameter semantics:
  `src/framework/parameters.cpp` and `input.example.toml`.
- Engine algorithm updates:
  `src/engines/srpic.hpp` or `src/engines/grpic.hpp`, plus relevant kernels.
- Metric/coordinate behavior:
  `src/metrics/*.h`.
- Output/checkpoint behavior:
  `src/output/*` and framework output/checkpoint integration points.
- New problem setups:
  add/update `pgens/<name>/pgen.hpp` and matching `.toml` examples.
- External pgen additions:
  add in `extern/entity-pgens` only when the change explicitly targets external
  pgens and submodule updates are intended.

### Testing expectations for changes
- Prefer module-level unit tests under `src/*/tests/`.
- Follow existing naming patterns in per-module `tests/CMakeLists.txt`:
  executable `test-<module>-<name>.xc`, CTest name `<MODULE>::<name>`.
- If behavior depends on MPI/output toggles, test at least one relevant
  configuration of those flags.

### Documentation
Local documentation sources (in priority order):
- Source files and inline comments.
- `input.example.toml` for parameter contract and inferred values.
- `pgens/*/*.toml` for scenario-specific setup patterns.
- `README.md` for project positioning/community context.
- `minimal/README.md` for third-party dependency sanity checks.

External docs:
- Wiki: https://entity-toolkit.github.io/wiki/

When docs and code disagree, treat the code as the current truth and update docs
in the same change if needed.

---

## Build, Test, and Style

### Build (examples)
```bash
# Configure a normal run build (choose an actual pgen)
cmake -B build -D pgen=streaming -D output=ON

# Compile
cmake --build build -j "$(nproc)"

# Run
build/src/entity.xc -input pgens/streaming/twostream.toml
```

```bash
# Example GPU configure (set the right Kokkos arch flags for your hardware)
cmake -B build \
  -D pgen=streaming \
  -D output=ON \
  -D Kokkos_ENABLE_CUDA=ON \
  -D Kokkos_ARCH_AMPERE80=ON
```

### Tests
```bash
# Configure unit tests
cmake -B build -D TESTS=ON -D precision=double -D mpi=OFF -D output=ON

# Build tests
cmake --build build -j "$(nproc)"

# Run tests
ctest --test-dir build --output-on-failure
```

```bash
# MPI-enabled test variant
cmake -B build -D TESTS=ON -D mpi=ON -D output=ON
cmake --build build -j "$(nproc)"
ctest --test-dir build --output-on-failure
```

### Style checks
```bash
# C++ formatting check (run on changed files)
clang-format --dry-run --Werror <changed_file1> <changed_file2>
```

```bash
# TOML formatting check for simulation inputs/problem setups
taplo format --check input.example.toml pgens/**/*.toml
```

```bash
# Optional CMake linting (if cmake-lint is installed)
cmake-lint CMakeLists.txt cmake/*.cmake src/*/CMakeLists.txt src/*/tests/CMakeLists.txt
```

### Style reminders
- C++ standard is C++17.
- Use the existing `.clang-format` rules (2-space indent, 80-column target,
  include regrouping/sorting, explicit brace style).
- Keep include order consistent with existing files and formatter categories.
- Prefer existing aliases/types (`real_t`, `coord_t`, enum wrappers) over raw
  primitive duplication.
- Follow existing naming patterns (`m_` members, explicit enum wrappers,
  template constraints via traits/static_assert).
- Keep kernels/device code Kokkos-safe and avoid host-only behavior in
  `KOKKOS_LAMBDA` code paths.

---

## Common CMake Flags
- `-D pgen=<name_or_path>`: required for normal executable builds.
- `-D pgen=pgens/<name>`: use pgens provided by `extern/entity-pgens`.
- `-D TESTS=ON`: build test targets instead of main executable.
- `-D BENCHMARK=ON`: build benchmark target.
- `-D mpi=ON|OFF`: enable/disable MPI integration.
- `-D output=ON|OFF`: enable/disable ADIOS2 output/checkpoint.
- `-D precision=single|double`: floating precision.
- `-D deposit=zigzag|esirkepov` and `-D shape_order=<int>`: deposition settings.

---

## When in doubt
Prefer the source code over documentation. If behavior is unclear, stop and ask
for clarification rather than guessing.
