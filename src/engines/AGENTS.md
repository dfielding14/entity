# AGENTS.md

## Scope
This file governs work in `/src/engines`.

Use with:
- `/Users/dbf75/Work/Research/AthenaK/entity/AGENTS.md`
- `/Users/dbf75/Work/Research/AthenaK/entity/src/AGENTS.md`

If instructions conflict, follow the closest `AGENTS.md` to the edited file.

---

## What `engines` owns
`engines` owns the simulation stepping layer between framework domain state and
kernel execution:
1. Base engine lifecycle (`init`, `run`, reporting, output/checkpoint hooks).
2. Engine selection (`SRPIC` vs `GRPIC`) via traits/dispatch helpers.
3. Per-timestep algorithm ordering for SRPIC and GRPIC.
4. Engine-side integration of optional problem-generator hooks.

Directory map:
- `engine.hpp`
  Base `Engine<S, M>` template and shared state.
- `engine_init.cpp`
  Startup flow, initial-condition path, checkpoint resume path.
- `engine_run.cpp`
  Main timestep loop, diagnostics, output/stats/checkpoint writes.
- `engine_printer.cpp`
  Runtime configuration and memory-footprint report.
- `engine_traits.h`
  `EngineSelector` mapping from `SimEngine` to concrete engine class.
- `srpic.hpp`
  SRPIC algorithm, boundaries, injection, current deposit/filter.
- `grpic.hpp`
  GRPIC algorithm, auxiliary fields, GR boundaries, current deposit/filter.

---

## Dispatch and instantiation model
- Runtime dispatch comes from `Simulation::run` + specialization matching in
  `src/entity.cpp`.
- `EngineSelector<SimEngine::SRPIC>::type` maps to `SRPICEngine`.
- `EngineSelector<SimEngine::GRPIC>::type` maps to `GRPICEngine`.
- Base engine methods are explicitly instantiated in:
  - `engine_init.cpp`
  - `engine_run.cpp`
  - `engine_printer.cpp`
  using `NTT_FOREACH_SPECIALIZATION(...)`.

When changing template signatures or required includes, keep explicit
instantiation units and specialization registry compatibility in sync.

---

## Base engine lifecycle contracts

### Construction and compatibility
- `Engine<S, M>` enforces:
  - `M::is_metric`
  - `user::PGen<S, M>::is_pgen`
  - compatibility of `PGen` engine/metric/dimension traits (`pgen_is_ok`)
- `raise::ErrorIf` aborts if the selected pgen is incompatible.

### `init()` contract
`engine_init.cpp` initializes in this order:
1. Stats writer (`m_metadomain.InitStatsWriter`).
2. Output/checkpoint writers when `OUTPUT_ENABLED`.
3. New-run path:
   - optional `init_flds` hook via `SetEMFields_kernel`
   - optional `InitPrtls(...)` hook
4. Resume path:
   - requires `OUTPUT_ENABLED`
   - requires `checkpoint.start_step > 0`
   - uses `ContinueFromCheckpoint(...)`.
5. Always prints the runtime report at the end.

### `run()` contract
`engine_run.cpp` loop order is:
1. `init()`.
2. Build timers (`FieldSolver`, `CurrentDeposit`, `ParticlePusher`, etc.).
3. While `step < max_steps`:
   - `step_forward(...)` on each local domain.
   - optional `CustomPostStep(step, time, dom)`.
   - advance `time` and `step`.
   - optional output/stats/checkpoint writes (`OUTPUT_ENABLED`), including:
     - optional `CustomFieldOutput(...)`
     - optional `CustomStat(...)`.
   - optional diagnostics print.
   - timer reset.

Note:
- `CustomPostStep` sees pre-increment `step/time`.
- output/checkpoint writes are called with both current and previous
  `(step, time)` pairs.

---

## Expected SRPIC step pipeline
`SRPICEngine::step_forward(...)` in `srpic.hpp` is ordered as:

1. First step bootstrap (`step == 0`):
   - communicate `B|E`
   - apply field boundaries `BC::B | BC::E`
   - run particle injector.
2. Field half-step (if `algorithms.fieldsolver.enable`):
   - `Faraday(HALF)`
   - communicate `B`
   - apply `BC::B`.
3. Particle/deposit block:
   - `ParticlePush`
   - if `algorithms.deposit.enable`:
     - zero `cur`
     - `CurrentsDeposit`
     - synchronize + communicate `J`
     - `CurrentsFilter`
   - communicate particles.
4. Field completion (if field solver enabled):
   - `Faraday(HALF)`
   - communicate/apply `BC::B`
   - `Ampere(ONE)`
   - optional `CurrentsAmpere` when deposit is enabled
   - communicate `E|J`
   - apply `BC::E`.
5. Injector pass.
6. Optional dead-particle cleanup (`particles.clear_interval`).

SRPIC boundary handlers currently implement:
- `MATCH`, `AXIS`, `ATMOSPHERE`, `FIXED`, `CONDUCTOR`
- `CUSTOM` path exists but currently raises an error
- `HORIZON` is rejected for SRPIC.

Important implementation detail:
- SRPIC current deposition kernel order is selected at compile time by
  `SHAPE_ORDER` (`deposit_with<SHAPE_ORDER>(...)`).

---

## Expected GRPIC step pipeline
`GRPICEngine::step_forward(...)` in `grpic.hpp` contains a larger bootstrap and
staggered-field sequence:

1. First step bootstrap (`step == 0`):
   - if field solver enabled, perform the full GR initialization chain:
     communicate/apply BCs for `B/B0/D/D0`, copy/swap staging fields, compute
     aux fields (`E/H`), run Faraday/Ampere bootstrap pushes, then swap fields.
   - if field solver disabled, copy `em -> em0`.
2. Field pre-push block (if field solver enabled):
   - `TimeAverageDB`
   - `ComputeAuxE`, communicate/apply `E` BCs
   - `Faraday(aux)`, communicate/apply `B` BCs
   - `ComputeAuxH`, communicate/apply `H` BCs.
3. Particle/deposit block:
   - `ParticlePush`
   - if deposit enabled:
     - zero `cur0`
     - `CurrentsDeposit`
     - synchronize + communicate `J`
     - apply current-boundary handling
     - `CurrentsFilter`
   - communicate particles.
4. Field completion block (if field solver enabled):
   - optional `TimeAverageJ` when deposit enabled
   - `ComputeAuxE`, communicate/apply `E` BCs
   - `Faraday(main)`, communicate/apply `B` BCs
   - `Ampere(aux)`, optional `AmpereCurrents(aux)`
   - communicate/apply `D` BCs
   - `ComputeAuxH`, communicate/apply aux BCs
   - `Ampere(main)`, optional `AmpereCurrents(main)`
   - `SwapFields`
   - communicate/apply `D` BCs.
5. Optional dead-particle cleanup.

GR boundary behavior is split by mode (`main`, `aux`, `curr`) and currently
supports `MATCH`, `AXIS`, `HORIZON` (and `CUSTOM`, which currently raises).

Current specialization registry enables GRPIC in 2D only; preserve this
assumption unless updating metrics + registry + kernels together.

---

## Problem-generator extension points
Engine code uses SFINAE (`traits::has_member` / `traits::has_method`) for
optional pgen hooks.

Shared hooks consumed by base engine:
- `init_flds`
- `InitPrtls(...)`
- `CustomPostStep(...)`
- `CustomFieldOutput(...)`
- `CustomStat(...)`

SRPIC-specific optional hooks/fields used in engine logic:
- `ext_force` (optional per-species gating via `ext_force.species`)
- `ext_current`
- `MatchFields(...)` or directional variants `MatchFieldsInX1/X2/X3`
- `FixFieldsConst(...)` for fixed BCs (non-const `FixFields(...)` path is not
  implemented)
- `AtmFields(...)` for atmosphere BC field enforcement.

When adding a new hook:
1. add a trait alias in `src/global/arch/traits.h`,
2. gate usage with `has_member`/`has_method`,
3. provide a clear fallback/error path.

---

## Change guidance and invariants

### Preserve ordering invariants
- Keep field solver, deposit, communication, and BC application order intact.
- Do not move particle communication before current deposition/filtering.
- Keep `em/em0` and `cur/cur0` staging semantics coherent when editing GRPIC.

### Preserve communication contracts
- Current arrays are explicitly zeroed before deposit.
- Field/current synchronization (`SynchronizeFields`) and halo communication
  (`CommunicateFields`) are both required in deposit-enabled paths.
- Boundary kernels depend on already-communicated ghost data.

### Keep report and params coherent
- If engine-facing runtime keys change, update `engine_printer.cpp` output
  where appropriate.
- Prefer existing `raise::*` and `logger::Checkpoint` patterns.

### Feature guards to maintain
- `OUTPUT_ENABLED`
- `MPI_ENABLED`
- `CUDA_ENABLED` / `HIP_ENABLED`
- `DEVICE_ENABLED`
- `GPU_AWARE_MPI`
- `SHAPE_ORDER`

Both guarded branches must remain buildable.

---

## Testing expectations for engine changes
There is currently no dedicated `/src/engines/tests` subtree. Validate changes
with:

1. Unit/integration test suite:
```bash
cmake -B build -D TESTS=ON -D mpi=OFF -D output=ON
cmake --build build -j "$(nproc)"
ctest --test-dir build --output-on-failure
```

2. At least one SRPIC runtime smoke case:
```bash
cmake -B build-sr -D pgen=streaming -D output=ON
cmake --build build-sr -j "$(nproc)"
build-sr/src/entity.xc -input pgens/streaming/twostream.toml
```

3. At least one GRPIC runtime smoke case:
```bash
cmake -B build-gr -D pgen=accretion -D output=ON
cmake --build build-gr -j "$(nproc)"
build-gr/src/entity.xc -input pgens/accretion/accretion.toml
```

For communication-sensitive edits, also run an MPI-enabled build/test variant.

---

## Common failure modes
- Reordering substeps and breaking field/current staggering.
- Updating only SRPIC or only GRPIC when shared behavior should match.
- Forgetting one side of a feature guard (`OUTPUT_ENABLED`, `MPI_ENABLED`, etc.).
- Adding pgen-dependent logic without SFINAE guards.
- Changing BC logic without matching comm/sync behavior.
- Introducing engine changes without validating at least one SR and one GR run.
