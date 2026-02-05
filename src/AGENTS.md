# AGENTS.md

## Scope
This file covers work inside `/src` only. Use it together with the repository
root `AGENTS.md`.

If guidance conflicts:
1. Explicit user request
2. Closest `AGENTS.md` to edited files
3. Root `AGENTS.md`

---

## Purpose of `src/`
`/src` contains the simulation runtime and all core modules used to build
`entity.xc`.

Entrypoint:
- `entity.cpp`

Primary modules:
- `global/`
- `framework/`
- `engines/`
- `kernels/`
- `metrics/`
- `output/`
- `archetypes/`

---

## Module Dependency Graph
The local CMake wiring in `src/CMakeLists.txt` and `src/*/CMakeLists.txt`
implies this effective dependency shape:

1. `ntt_global` is foundational.
2. `ntt_metrics` and `ntt_kernels` depend on `ntt_global`.
3. `ntt_archetypes` depends on `ntt_global` + `ntt_kernels`.
4. `ntt_output` depends on `ntt_global` (extra sources when `output=ON`).
5. `ntt_framework` depends on `ntt_global` + `ntt_metrics` + `ntt_kernels` + `ntt_output`.
6. `ntt_engines` depends on `ntt_global` + `ntt_framework` + `ntt_metrics` + `ntt_kernels` + `ntt_archetypes` + `ntt_pgen` (+ `ntt_output` if enabled).
7. `entity.xc` links `ntt_global`, `ntt_framework`, `ntt_metrics`, `ntt_engines`, `ntt_pgen`.

Implication:
- Do not introduce dependency edges that invert this layering without a strong
  reason and matching CMake updates.

---

## Runtime Dispatch Model
The runtime path inside `/src` is:

1. `entity.cpp` constructs `ntt::Simulation`.
2. `framework/simulation.cpp` reads input, sets requested engine/metric/dim,
   and initializes global runtime state.
3. `framework/specialization_registry.h` enumerates allowed
   engine-metric-dimension specializations.
4. `engines/engine_traits.h` maps `SimEngine` to concrete engine classes.
5. `entity.cpp` checks problem-generator compatibility at compile time
   (`should_compile`) before launching.
6. `engines/engine_init.cpp` and `engines/engine_run.cpp` execute initialization
   and timestep loops.

Key invariant:
- A run must satisfy all three:
  - Requested tuple appears in specialization registry.
  - Tuple is enabled by build/config.
  - Selected `user::PGen` declares compatibility traits that match.

---

## Change Entry Points

### Add or change simulation parameters
- Edit `framework/parameters.cpp` (+ declarations in `framework/parameters.h` if needed).
- Keep `input.example.toml` in sync with required/new behavior.
- Preserve immutable vs mutable vs checkpoint-resume semantics.

### Add a new metric or specialization
- Implement metric in `metrics/`.
- Register tuple(s) in `framework/specialization_registry.h`.
- Ensure engine and pgen compatibility traits match expected usage.
- Add/adjust tests in `metrics/tests/` and affected module tests.

### Change SR/GR algorithm behavior
- Primary files: `engines/srpic.hpp`, `engines/grpic.hpp`.
- Keep step ordering and comm/boundary sequencing coherent.
- Push reusable math/compute into `kernels/` when logic is not engine-specific.

### Change domain/data orchestration
- `framework/domain/*` for decomposition/communications/stats/output hooks.
- `framework/containers/*` for field/particle ownership and transport formats.

### Output/checkpoint changes
- `output/*` and related `framework/domain/output.cpp` /
  `framework/domain/checkpoint.cpp`.
- Guard output-specific code with existing `OUTPUT_ENABLED` patterns.

---

## Compile-Time & Macro Guards
Respect existing feature guards and keep both branches buildable:
- `MPI_ENABLED`
- `OUTPUT_ENABLED`
- `DEVICE_ENABLED`
- `CUDA_ENABLED`
- `HIP_ENABLED`
- `GPU_AWARE_MPI`
- `SINGLE_PRECISION`
- `SHAPE_ORDER`

When adding behavior tied to one flag, verify fallback behavior for the opposite
branch.

---

## Includes, Naming, and Style in `src`
- Keep include grouping/order formatter-compatible with `.clang-format`.
- Prefer existing shared aliases/types (`real_t`, `coord_t`, enum wrappers).
- Reuse current error/log idioms (`raise::ErrorIf`, `raise::Fatal`,
  `logger::Checkpoint`) instead of custom patterns.
- Avoid host-only assumptions inside `KOKKOS_LAMBDA` paths.

---

## Testing for `src` Changes
Module tests live under:
- `src/global/tests`
- `src/metrics/tests`
- `src/kernels/tests`
- `src/archetypes/tests`
- `src/framework/tests`
- `src/output/tests`

Typical workflow:
```bash
cmake -B build -D TESTS=ON -D mpi=OFF -D output=ON
cmake --build build -j "$(nproc)"
ctest --test-dir build --output-on-failure
```

MPI-sensitive changes should also run:
```bash
cmake -B build -D TESTS=ON -D mpi=ON -D output=ON
cmake --build build -j "$(nproc)"
ctest --test-dir build --output-on-failure
```

---

## Out of Scope for `/src`
- CMake policy/dependency plumbing outside module-level source wiring:
  see `/cmake`.
- Built-in setup definitions and physics scenario selection:
  see `/pgens`.
- External third-party code:
  see `/extern` (treat as vendored unless explicitly updating submodules).

---

## Next Local Docs
For deeper detail, prefer nearest docs once created:
- `/src/global/AGENTS.md`
- `/src/framework/AGENTS.md`
- `/src/engines/AGENTS.md`
- `/src/kernels/AGENTS.md`
- `/src/metrics/AGENTS.md`
- `/src/output/AGENTS.md`
- `/src/archetypes/AGENTS.md`
