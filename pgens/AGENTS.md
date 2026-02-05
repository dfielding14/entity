# AGENTS.md

## Scope
This file governs work in `/pgens`.

Use with:
- `/Users/dbf75/Work/Research/AthenaK/entity/AGENTS.md`
- `/Users/dbf75/Work/Research/AthenaK/entity/src/AGENTS.md`
- `/Users/dbf75/Work/Research/AthenaK/entity/src/archetypes/AGENTS.md`

If instructions conflict, follow the closest `AGENTS.md` to edited files.

---

## What This Layer Owns
`pgens` contains problem-specific physics setup code selected at configure
time with `-D pgen=<name_or_path>`.

Primary responsibilities:
1. Define `user::PGen<S, M>` specialization behavior for a setup.
2. Declare compatibility traits (`engines`, `metrics`, `dimensions`).
3. Provide optional setup hooks consumed by engine/runtime trait checks.
4. Provide setup-local input examples (`*.toml`) for reproducible runs.

`pgens` does not own engine stepping algorithms or container/kernel contracts;
those belong to `/src/engines`, `/src/framework`, `/src/kernels`, and
`/src/archetypes`.

---

## Directory Map
- `pgens/pgen.hpp`
  Generic fallback `user::PGen` template (compatible with all built-in tuples;
  mostly initialization/logging baseline).
- `pgens/<setup>/pgen.hpp`
  Setup-specific `user::PGen` implementation.
- `pgens/<setup>/*.toml`
  Example runtime configuration for that setup.
- `pgens/<setup>/sketch.*`, `*.py`
  Optional helper assets/scripts for setup geometry visualization.

Built-in setups in this repository:
- `accretion`
- `magnetosphere`
- `reconnection`
- `shock`
- `streaming`
- `turbulence`
- `wald`

---

## Build Selection Contract
Problem-generator selection is configured in CMake (`cmake/config.cmake`):
- Built-ins: `-D pgen=<dir_under_pgens>`
- External submodule setups: `-D pgen=pgens/<name>` (from `extern/entity-pgens`)
- Arbitrary path: `-D pgen=/absolute/or/relative/path/to/setup_dir`

The selected setup directory is added as an include path for `ntt_pgen`, and
the engine includes `pgen.hpp` from that path.

Required invariant:
- The selected directory must contain `pgen.hpp`.

---

## `user::PGen` Contract

### Base shape
Every setup defines:
- `template <SimEngine::type S, class M> struct PGen : arch::ProblemGenerator<S, M>`

Keep these in each setup:
- `using arch::ProblemGenerator<S, M>::D;`
- `using arch::ProblemGenerator<S, M>::C;`
- `using arch::ProblemGenerator<S, M>::params;`

### Compatibility traits (required)
Each pgen must define:
- `static constexpr auto engines = traits::compatible_with<...>::value;`
- `static constexpr auto metrics = traits::compatible_with<...>::value;`
- `static constexpr auto dimensions = traits::compatible_with<...>::value;`

Engine construction enforces compatibility with:
- selected simulation engine (`S`)
- selected metric (`M::MetricType`)
- selected dimension (`M::Dim`)

Trait mismatch is a runtime error during engine construction.

### Constructor pattern
Common constructor forms:
- `PGen(const SimulationParams&, const Metadomain<S, M>&)`
- `PGen(const SimulationParams&, Metadomain<S, M>&)` when mutating BCs/domain state

Constructors should:
- parse setup-local parameters from `[setup]`
- validate species assumptions early (`raise::ErrorIf`)
- initialize field/distribution helper functors

### Species indexing
Pgen injector APIs use 1-based species indices (`spidx_t`):
- `{1, 2}`, `{n + 1, n + 2}`, etc.
- direct domain access remains 0-based (`domain.species[sp - 1]`)

Do not mix these conventions.

---

## Optional Hook Catalog
Hooks are detected via SFINAE traits in `src/global/arch/traits.h`.
Only implement what your setup needs.

Initialization and stepping:
- `init_flds` member functor
  consumed by `Engine::init()` through `arch::SetEMFields_kernel`.
- `InitPrtls(Domain<S, M>&)`
  called once on non-resume startup.
- `CustomPostStep(timestep_t, simtime_t, Domain<S, M>&)`
  called each step after `step_forward`.

Field boundary integration (SR and/or GR depending on engine path):
- `MatchFields(simtime_t)` or directional variants:
  `MatchFieldsInX1/X2/X3(simtime_t)`
- `AtmFields(simtime_t)` for atmosphere field BCs.
- `FixFieldsConst(const bc_in&, const em&) -> std::pair<real_t, bool>`
  for fixed-value BC components.

External source terms:
- `ext_current` member functor (with `jx1/jx2/jx3`) for SRPIC Ampere source.
- `ext_force` member functor (optional; currently not used by built-in pgens
  here, but supported by engine traits).

Output/stats extension hooks:
- `CustomFieldOutput(const std::string& name,
                     ndfield_t<M::Dim, 6>& buff,
                     index_t idx,
                     timestep_t step,
                     simtime_t time,
                     const Domain<S, M>& dom)`
  used when names are listed in `output.fields.custom`.
- `CustomStat(const std::string& name,
              timestep_t step,
              simtime_t time,
              const Domain<S, M>& dom) -> real_t`
  used when names are listed in `output.stats.custom`.

If custom names are requested in TOML but hook methods are absent, runtime
errors are raised during output/stats writes.

---

## Built-In Setup Patterns

### `streaming`
- Compatible with `SRPIC + Minkowski + (1D/2D/3D)`.
- Uses `init_flds` and `InitPrtls`.
- Injects paired uniform Maxwellian species; validates even `nspec` and charge
  symmetry by species pairs.

### `reconnection`
- Compatible with `SRPIC + Minkowski + (2D/3D)`.
- Uses `init_flds`, `InitPrtls`, `CustomPostStep`.
- Implements directional field matching (`MatchFieldsInX1`, `MatchFieldsInX2`).
- Uses non-uniform replenishment in custom post-step.

### `shock`
- Compatible with `SRPIC + Minkowski + (1D/2D/3D)`.
- Uses `init_flds`, `InitPrtls`, `CustomPostStep`.
- Implements `MatchFields` and `FixFieldsConst`.
- Custom post-step advances a moving injector, resets fields, tags/removes
  particles, and reinjects slab plasma.

### `turbulence`
- Compatible with `SRPIC + Minkowski + (2D/3D)`.
- Uses `ext_current`, `init_flds`, `InitPrtls`, `CustomPostStep`.
- Post-step updates antenna amplitudes and handles particle escape/resampling.

### `magnetosphere`
- Compatible with `SRPIC + (Spherical/QSpherical) + 2D`.
- Uses `init_flds`, `AtmFields`, `MatchFields`.
- Designed for atmosphere and matched-boundary operation.

### `accretion`
- Compatible with `GRPIC + (Kerr_Schild/QKerr_Schild/Kerr_Schild_0) + 2D`.
- Uses `init_flds`, `InitPrtls`, `CustomPostStep`.
- Uses non-uniform injection based on sigma/density criteria.

### `wald`
- Compatible with `GRPIC + (Kerr_Schild/QKerr_Schild/Kerr_Schild_0) + 2D`.
- Primarily field initialization (`init_flds`), vacuum-oriented baseline.

---

## Setup TOML Conventions
Pgen-specific runtime controls live under `[setup]` in each setup TOML.

Common cross-section dependencies in pgen code:
- `[particles]`:
  `ppc0`, `use_weights`, species list and species metadata.
- `[scales]`:
  values such as `n0`, `sigma0`, `skindepth0`, `B0`.
- `[algorithms.timestep]`:
  `dt` is often used by custom post-step logic.
- `[grid.boundaries.*]`:
  required when setup relies on MATCH/ATMOSPHERE/FIXED behavior.

When adding new setup keys:
1. Parse in setup `pgen.hpp` constructor.
2. Add validation and defaults where appropriate.
3. Update at least one `pgens/<setup>/*.toml` example.
4. Update `input.example.toml` only if behavior is intended as a global input
   contract, not setup-local.

---

## Change Guidance

### Adding a new setup
1. Create `pgens/<name>/pgen.hpp`.
2. Add one or more runnable `pgens/<name>/*.toml`.
3. Define compatibility traits conservatively.
4. Reuse archetype helpers (`InjectUniform...`, `InjectNonUniform`,
   `SetEMFields_kernel`) before writing bespoke kernels.
5. Keep hook signatures exactly matching trait-detected expectations.

### Modifying an existing setup
Update together:
1. `pgen.hpp` logic and setup parameter parsing.
2. Sample TOML defaults and comments.
3. Any setup-specific helper scripts/docs if geometry meaning changed.

---

## Validation Checklist
There are currently no dedicated unit tests under `/pgens`, so validate with
runtime scenarios.

Recommended workflow:
```bash
cmake -B build -D pgen=<setup> -D output=ON
cmake --build build -j "$(nproc)"
build/src/entity.xc -input pgens/<setup>/<example>.toml
```

If hook behavior depends on specific features, also test at least one matching
variant:
- `-D mpi=ON` for MPI-sensitive paths (boundary exchange, seeded randomness).
- `-D output=ON` for custom output/stats hooks.
- engine/metric combination implied by pgen compatibility traits.

---

## Common Failure Modes
- Compatibility traits too broad or too narrow for actual hook assumptions.
- Mixing 1-based species IDs in injector calls with 0-based container access.
- Forgetting `particles.use_weights` consistency in non-Cartesian setups.
- Hook signatures not matching trait-expected names/arguments.
- Requesting `output.fields.custom` or `output.stats.custom` without implementing
  matching pgen hooks.
- Mutating global BC/domain state in post-step without using a mutable
  `Metadomain` reference.

