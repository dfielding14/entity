# AGENTS.md

## Scope
This file covers `/src/framework/domain` only.

Use with:
- `/Users/dbf75/Work/Research/AthenaK/entity/AGENTS.md`
- `/Users/dbf75/Work/Research/AthenaK/entity/src/AGENTS.md`
- `/Users/dbf75/Work/Research/AthenaK/entity/src/framework/AGENTS.md`

If instructions conflict, follow the closest `AGENTS.md` to the file being edited.

---

## What this layer owns
`framework/domain` owns domain decomposition and per-domain orchestration:
1. Grid and mesh indexing/range APIs.
2. Global metadomain construction and validation.
3. Neighbor mapping and effective boundary assignment.
4. Field/particle communication and synchronization.
5. Domain-level output, checkpoint, and stats integration.

Files:
- `grid.h/.cpp`
- `mesh.h`
- `domain.h`
- `metadomain.h/.cpp`
- `communications.cpp`
- `comm_mpi.hpp`
- `comm_nompi.hpp`
- `output.cpp`
- `checkpoint.cpp`
- `stats.cpp`

---

## Core data model

### `Grid<D>`
- Holds active resolution and ghost-zone-aware index helpers (`i_min/i_max`,
  `n_active/n_all`).
- Produces host/device Kokkos ranges for active/all/custom cell regions.

### `Mesh<M>`
- Extends `Grid` with metric instance, physical extents, and directional
  field/particle BC maps.
- Handles physical extent intersection and coordinate-to-index range conversion.

### `Domain<S, M>`
- Owns one local block: `mesh`, `fields`, `species`, and neighbor indices.
- Supports placeholder domains for non-local ownership in metadomain layouts.

### `Metadomain<S, M>`
- Owns global decomposition metadata and all domain objects.
- Provides iteration over local domains, comm operations, and
  output/stats/checkpoint entry points.

---

## Construction and validation lifecycle
`Metadomain` constructor flow in `metadomain.cpp`:
1. `initialValidityCheck()`
2. `createEmptyDomains()`
3. `redefineNeighbors()`
4. `redefineBoundaries()`
5. `finalValidityCheck()`
6. `metricCompatibilityCheck()`

Do not reorder/remove these stages without replacing equivalent guarantees.

Important invariants:
- With MPI enabled, current design requires `global_ndomains == MPI size`
  and effectively one local domain per rank.
- Non-local domains must stay placeholders (no full field/particle allocation).
- Neighbor relation must be reciprocal (`neighbor(-dir(neighbor(dir(self)))) == self`).
- Boundary maps must never remain `INVALID` after `redefineBoundaries()`.

---

## Boundary semantics
`redefineBoundaries()` converts global BC intent to per-domain effective BC:
- Interior faces become `SYNC`.
- Edge faces inherit global configured BC.
- If a face is periodic but maps to a different domain, it is converted to `SYNC`
  for explicit communication.
- Corner-direction BCs are derived from associated orthogonal directions.

When modifying BC behavior, validate:
- orthogonal directions
- corner directions
- periodic and sync interactions
- consistency between field and particle BC maps

---

## Communication semantics
Communication entry points:
- `Metadomain::CommunicateFields(...)`
- `Metadomain::SynchronizeFields(...)`
- `Metadomain::CommunicateParticles(...)`

Backend split:
- MPI path: `comm_mpi.hpp`
- Non-MPI path: `comm_nompi.hpp`

Current constraints:
- Multi-domain single-rank communication is not implemented.
- Non-MPI multi-domain communication is not implemented.
- Additive vs non-additive field exchange paths must preserve existing
  indexing/slicing semantics.

When editing `communications.cpp`:
- Keep send/recv rank/index derivation in sync with BC logic.
- Keep component-range selection aligned with engine mode and `CommTags`.
- Preserve ghost-zone versus active-zone slice contracts.

---

## Output, checkpoint, and stats constraints
Domain-level output paths (`output.cpp`, `checkpoint.cpp`, `stats.cpp`) currently assume:
- Exactly one local subdomain per rank.
- Local domain must not be a placeholder.

If you change these assumptions, you must update all three subsystems together:
1. Output writer layout and particle/species writes.
2. Checkpoint define/read/write layout and metadata flow.
3. Stats writer aggregation and local-domain iteration model.

Checkpoint flow depends on:
- `step-%08lu.bp` naming
- `meta-%08lu.toml` metadata conventions

Do not change naming/layout without coordinated updates to resume logic.

---

## Feature guards in this subtree
High-impact guards:
- `MPI_ENABLED`
- `OUTPUT_ENABLED`
- `DEVICE_ENABLED`
- `GPU_AWARE_MPI`
- Engine template branches (`SRPIC` vs `GRPIC`)

Rule:
- Keep both guarded branches buildable and behaviorally coherent after edits.

---

## Safe change points
1. `grid.*`
   Range policy and indexing helpers.
2. `mesh.h`
   Extent/BC mapping utilities.
3. `metadomain.cpp`
   Decomposition, neighbor/BC assignment, consistency checks.
4. `communications.cpp` and `comm_*.hpp`
   Field communication routing and transfer mechanics.
5. `output.cpp` / `checkpoint.cpp` / `stats.cpp`
   Domain-facing I/O and reductions.

Prefer incremental edits and preserve assertions unless replacing them with
equivalent stronger checks.

---

## Tests to run for domain changes
Domain behavior is covered by framework tests:
- `/Users/dbf75/Work/Research/AthenaK/entity/src/framework/tests/grid_mesh.cpp`
- `/Users/dbf75/Work/Research/AthenaK/entity/src/framework/tests/metadomain.cpp`
- `/Users/dbf75/Work/Research/AthenaK/entity/src/framework/tests/comm_nompi.cpp`
- `/Users/dbf75/Work/Research/AthenaK/entity/src/framework/tests/comm_mpi.cpp`

Suggested commands:
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

## Common failure modes
- Breaking reciprocal neighbor mapping when changing decomposition logic.
- Leaving BCs invalid for corner directions.
- Diverging MPI and non-MPI communication behavior.
- Changing slice conventions and corrupting ghost/active updates.
- Altering checkpoint/output layout without updating resume/reader logic.
