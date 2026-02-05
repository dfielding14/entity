# AGENTS.md

## Scope
This file covers `/src/archetypes` only.

Use with:
- `/Users/dbf75/Work/Research/AthenaK/entity/AGENTS.md`
- `/Users/dbf75/Work/Research/AthenaK/entity/src/AGENTS.md`

If instructions conflict, follow the closest `AGENTS.md` to edited files.

---

## What This Layer Owns
`archetypes` provides reusable, composable building blocks for problem setup
logic used by `user::PGen` implementations:
1. Base pgen class (`ProblemGenerator`).
2. Energy-distribution functors.
3. Spatial-distribution functors.
4. Field-initialization kernel wrapper (`SetEMFields_kernel`).
5. Particle injection orchestration (`InjectUniform`, `InjectNonUniform`,
   `InjectGlobally`).
6. Small convenience wrappers in `utils.h`.

This module is header-only (`ntt_archetypes` is an `INTERFACE` target).

---

## Directory Map
- `problem_generator.h`
  Base class for `user::PGen<S, M>`.
- `energy_dist.h`
  `EnergyDistribution`, `Cold`, `Powerlaw`, `Maxwellian`, sampling helpers.
- `spatial_dist.h`
  `SpatialDistribution`, `Uniform`, `Replenish`, `ReplenishUniform`.
- `field_setter.h`
  `SetEMFields_kernel<I, S, M>` for stagger-aware EM initialization.
- `particle_injector.h`
  Region deduction, particle count estimation, uniform/non-uniform/global injectors.
- `utils.h`
  Convenience wrappers (`InjectUniformMaxwellian(s)`).
- `tests/`
  Archetype-focused unit tests (`ARCHETYPES::*`).

---

## Composition Contracts

### `arch::ProblemGenerator<S, M>`
- Declares:
  - `is_pgen`
  - `D` and `C` aliases (`M::Dim`, `M::CoordType`)
  - `params` reference (`SimulationParams`)
- Intended parent for `user::PGen<S, M>`.
- Typical pgen pattern in `pgens/*`:
  - define compatibility traits (`engines`, `metrics`, `dimensions`)
  - expose `using ProblemGenerator<S, M>::D/C/params`
  - provide optional members/hooks consumed by engine traits.

### Energy distributions (`energy_dist.h`)
- Must model the `operator()(coord_t<M::Dim>, vec_t<Dim::_3D>&)` contract.
- Built-ins expose `is_energy_dist = true`.
- Coordinate/basis semantics:
  - Cartesian SR: global Cartesian basis
  - non-Cartesian SR: tetrad basis
  - GR: covariant basis
- `Maxwellian` constraints:
  - temperature must be non-negative
  - drift vector must have size 3
  - drift boosting is effectively Cartesian-only.

### Spatial distributions (`spatial_dist.h`)
- Must model `operator()(coord_t<M::Dim>) -> real_t`.
- Built-ins expose `is_spatial_dist = true`.
- Expected return value is a local injection factor (usually in `[0, 1]`).
- `Replenish*` variants read existing density fields and compare to target.

### Field initializer functors (`field_setter.h`)
- `SetEMFields_kernel<I, S, M>` detects available methods via traits:
  - SR methods: `ex1/ex2/ex3`, `bx1/bx2/bx3`
  - GR methods: `dx1/dx2/dx3`, `bx1/bx2/bx3`
- At least one supported method must exist.
- GR static requirement: component triplets must be complete or absent
  (no partial `dx*` or partial `bx*` sets).
- Kernel handles staggering, ghost-cell offsets, coordinate conversion, and
  basis conversion to stored field representation.

---

## Injection Invariants

### Species indexing and counts
- Injector APIs use 1-based species indices (`spidx_t`) and index into
  `domain.species[sp - 1]`.
- Uniform and non-uniform pair injectors increase `npart` for both species.
- `InjectNonUniform` also updates species counters.

### Weights contract
- Non-Cartesian metrics require weighted particles.
- Cartesian metrics should not use weights.
- Runtime arg `use_weights` must match `params["particles.use_weights"]`.
- Violations are hard errors (`raise::ErrorIf`).

### Tracking payload contract
- Tracking-enabled species require integer payload slots:
  - non-MPI: at least 1 (`pldi::spcCtr`)
  - MPI: at least 2 (`pldi::spcCtr`, `pldi::domIdx`)
- Injector kernels enforce this before writing payload tracking metadata.

### Region and box semantics
- `DeduceRegion` and `ComputeNumInject` work in physical extents and convert to
  code coordinates.
- Empty `box` in non-uniform injection means full active local range.
- Non-empty `box` must match metric dimension.
- Global injectors are intentionally inefficient and intended for debug/small
  injection workloads.

### Charge pairing
- Pair injectors warn (not fatal) when total injected pair charge is non-zero.

---

## Integration Points

Primary consumers:
- `/Users/dbf75/Work/Research/AthenaK/entity/src/engines/engine_init.cpp`
  (`init_flds` via `SetEMFields_kernel`, optional `InitPrtls` hook call)
- `/Users/dbf75/Work/Research/AthenaK/entity/src/engines/srpic.hpp`
  (replenishment and atmosphere injectors)
- `/Users/dbf75/Work/Research/AthenaK/entity/pgens/*/pgen.hpp`
  (custom pgens composing distributions and injectors)
- `/Users/dbf75/Work/Research/AthenaK/entity/src/kernels/injectors.hpp`
  (device kernels backing archetype injector wrappers)

When editing archetypes, verify kernel constructor assumptions still match
container fields and payload layout in framework containers.

---

## Change Guidance

### Adding a new energy or spatial distribution
1. Add a functor with device-safe `Inline operator()`.
2. Preserve static marker (`is_energy_dist` or `is_spatial_dist`) if meant for
   injector templates.
3. Keep metric/dimension constraints explicit (`static_assert(M::is_metric)`).
4. Add or update archetype tests where feasible.

### Editing injector behavior
Update together:
1. wrapper logic in `particle_injector.h`
2. kernel expectations in `/src/kernels/injectors.hpp`
3. any `pgens/*` call sites depending on old behavior
4. counter/npart semantics for tracking paths

### Editing field initialization behavior
Keep SR/GR and dimensional branches coherent in `SetEMFields_kernel`, including:
- staggering offsets (`HALF`, `COORD`, ghost handling)
- transform calls (`Idx::*`, `Crd::*`)
- completeness requirements for GR components.

---

## Build Guards
Frequent guards in this module:
- `MPI_ENABLED` (tracking payload and global-domain index handling)
- `OUTPUT_ENABLED` is not central here, but archetype outputs feed codepaths
  that can be output-enabled later in the engine/runtime.

Keep guard branches buildable and behaviorally consistent.

---

## Tests To Run
Archetype tests are in:
- `/Users/dbf75/Work/Research/AthenaK/entity/src/archetypes/tests`

Registered tests:
- `ARCHETYPES::energy_dist`
- `ARCHETYPES::spatial_dist`
- `ARCHETYPES::field_setter`
- `ARCHETYPES::powerlaw`

Suggested commands:
```bash
cmake -B build -D TESTS=ON -D mpi=OFF -D output=ON
cmake --build build -j "$(nproc)"
ctest --test-dir build --output-on-failure -R '^ARCHETYPES::'
```

Note:
- There is currently no dedicated unit test for `particle_injector.h` wrappers
  or `problem_generator.h` itself; validate those changes with at least one
  runtime pgen scenario.

---

## Common Failure Modes
- Mixing basis assumptions (Cartesian/tetrad/covariant) in distribution output.
- Using weights inconsistent with geometry or input flags.
- Forgetting species index is 1-based in injector calls.
- Breaking tracking payload writes by changing `pld_i` assumptions.
- Implementing only partial GR field components in field initializer functors.
- Editing wrapper logic without matching `/src/kernels/injectors.hpp` behavior.
