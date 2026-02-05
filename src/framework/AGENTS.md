# AGENTS.md

## Scope
This file governs work in `/src/framework`.

Use with:
- `/Users/dbf75/Work/Research/AthenaK/entity/AGENTS.md`
- `/Users/dbf75/Work/Research/AthenaK/entity/src/AGENTS.md`

If instructions conflict, prefer the closest `AGENTS.md` to the edited file.

---

## What `framework` owns
`framework` is the simulation orchestration and data-management layer between
high-level engine steps and low-level kernels/metrics.

Main responsibilities:
1. Parse and validate runtime configuration (`simulation.cpp`, `parameters.cpp`).
2. Construct derived/inferred parameters and enforce constraints.
3. Define global decomposition (`Metadomain`) and local blocks (`Domain`).
4. Own field and particle container lifecycle.
5. Coordinate field/particle communication and synchronization.
6. Integrate output, stats, and checkpoint paths with domain state.

---

## Directory map
- `simulation.h/.cpp`
  CLI/input handling, startup/shutdown, engine request tuple extraction.
- `parameters.h/.cpp`
  Parameter ingestion, inference, promises, mutable/immutable checkpoint logic.
- `specialization_registry.h`
  Allowed engine/metric/dimension combinations.
- `domain/`
  Grid/mesh, decomposition, neighbors, BC propagation, comm/output/stats/checkpoint hooks.
- `containers/`
  `Fields`, `Particles`, and `ParticleSpecies` layouts and IO/comm helpers.
- `tests/`
  Framework-focused unit tests and MPI/non-MPI comm checks.

---

## Critical invariants

### Parameter lifecycle
`SimulationParams` is intentionally split:
1. `setImmutableParams(...)`
2. `setMutableParams(...)`
3. `setCheckpointParams(...)`
4. `setSetupParams(...)`
5. `checkPromises()`

Rules:
- Do not bypass `promiseToDefine(...)`/`checkPromises()` semantics.
- Preserve resume behavior: immutable values come from checkpoint metadata,
  mutable values may come from current input.
- Keep `input.example.toml` aligned with any required/new keys.

### Decomposition and domain ownership
- `Metadomain` enforces shape/consistency checks at construction.
- With MPI enabled, current implementation assumes exactly one domain per rank.
- Non-local domains must remain placeholders without full field/particle allocation.
- Neighbor and boundary assignments are validated in `finalValidityCheck()`.

### Communication behavior
- Field communication path: `domain/communications.cpp` plus
  `domain/comm_mpi.hpp` or `domain/comm_nompi.hpp`.
- `SYNC` boundaries drive inter-domain exchange; periodic edges may map to sync.
- Keep send/recv slicing logic and additive-vs-copy semantics consistent.

### Output/checkpoint/stats constraints
- Current output/stats/checkpoint paths assume one local subdomain per rank.
- Maintain guard logic around `OUTPUT_ENABLED` and MPI-specific branches.
- Checkpoint resume logic depends on metadata TOML files and step naming.

---

## Safe change points

### Add or modify runtime params
- Primary file: `parameters.cpp`.
- Update any related validation, inference, defaults usage, and promises.
- Ensure behavior is coherent in both fresh run and resume mode.

### Update simulation startup behavior
- Primary file: `simulation.cpp`.
- Preserve CLI compatibility (`-input`, resume aliases) and error reporting.
- Keep `GlobalInitialize`/`GlobalFinalize` ownership unchanged.

### Change container schemas
- Files: `containers/fields.*`, `containers/particles.*`, `containers/species.h`.
- Treat layout changes as high-risk: check comm, checkpoint, and output codepaths.
- Validate memory footprint and allocation behavior for all engine/macro branches.

### Change domain/decomposition logic
- Files under `domain/`.
- Revalidate neighbor mapping, boundary propagation, and local/non-local domain assumptions.
- Do not silently relax one-domain-per-rank assumptions without updating all dependent logic.

---

## Build guards and feature switches
Common guards used heavily in this subtree:
- `MPI_ENABLED`
- `OUTPUT_ENABLED`
- `DEVICE_ENABLED`
- `GPU_AWARE_MPI`

When changing guarded code, keep both sides compilable and logically consistent.

---

## Testing expectations
Framework tests are in:
- `/Users/dbf75/Work/Research/AthenaK/entity/src/framework/tests`

CMake behavior:
- Non-MPI builds run tests like `parameters`, `particles`, `fields`, `grid_mesh`,
  and `comm_nompi` (plus `metadomain` in debug builds).
- MPI builds run `comm_mpi`.

Recommended commands:
```bash
cmake -B build -D TESTS=ON -D mpi=OFF -D output=ON
cmake --build build -j "$(nproc)"
ctest --test-dir build --output-on-failure
```

```bash
cmake -B build -D TESTS=ON -D mpi=ON -D output=ON
cmake --build build -j "$(nproc)"
ctest --test-dir build --output-on-failure
```

---

## Common pitfalls in this subtree
- Changing parameter parsing without updating promise definitions.
- Breaking resume semantics by mixing mutable and immutable sources.
- Introducing multi-domain-per-rank assumptions in one file but not all comm/output paths.
- Modifying particle/field layouts without updating checkpoint/output declarations.
- Forgetting to mirror behavior across MPI and non-MPI comm backends.

---

## Next local docs
For deeper detail in nested areas (planned):
- `/Users/dbf75/Work/Research/AthenaK/entity/src/framework/domain/AGENTS.md` (created)
- `/Users/dbf75/Work/Research/AthenaK/entity/src/framework/containers/AGENTS.md`
