# AGENTS.md

## Scope
This file covers `/src/framework/containers` only.

Use with:
- `/Users/dbf75/Work/Research/AthenaK/entity/AGENTS.md`
- `/Users/dbf75/Work/Research/AthenaK/entity/src/AGENTS.md`
- `/Users/dbf75/Work/Research/AthenaK/entity/src/framework/AGENTS.md`

If instructions conflict, follow the closest `AGENTS.md` to edited files.

---

## What This Layer Owns
`framework/containers` defines the core runtime data containers for domain state:
- `fields.h/.cpp` and `fields_io.cpp`
- `species.h`
- `particles.h/.cpp`, `particles_comm.cpp`, `particles_io.cpp`

Primary responsibilities:
1. Field array allocation and layout per engine/dimension.
2. Particle species metadata and per-species particle storage.
3. Particle lifecycle operations (`npart`, tag-based compaction/sorting).
4. Particle MPI exchange packing/unpacking.
5. Particle and field output/checkpoint declarations and read/write wiring.

---

## Core Data Model

### `ParticleSpecies`
- Immutable metadata: index, label, mass, charge, pusher, cooling, payload counts.
- Mutable capacity: `m_maxnpart`.
- Tracking contract:
  - non-MPI tracking requires `npld_i >= 1`
  - MPI tracking requires `npld_i >= 2`

### `Particles<D, C>`
- Inherits `ParticleSpecies`.
- Stores per-particle arrays:
  - indices/displacements: `i*`, `dx*`, and `_prev` variants
  - momenta: `ux1/ux2/ux3`
  - `weight`, `tag`, optional payload arrays `pld_r/pld_i`
  - `phi` only for `D == 2 && C != Coord::Cart`
- Runtime state:
  - `m_npart` active particles
  - `m_counter` species counter
  - `m_is_sorted` (compacted by alive/dead tags)
- Tag-space size:
  - non-MPI: `2` (`dead`, `alive`)
  - MPI: `2 + (3^D - 1)` (dead/alive + send-direction tags)

### `Fields<D, S>`
- Allocates ghost-zone-inclusive `ndfield_t` arrays.
- Always allocated:
  - `em(6)`, `bckp(6)`, `cur(3)`, `buff(3)`
- GR-only allocations:
  - `aux(6)`, `em0(6)`, `cur0(3)` when `S == GRPIC`

---

## Ownership And Lifecycle Invariants

1. Placeholder vs local domain semantics depend on container allocation.
- `Domain` placeholder constructor uses default `Fields{}` and empty `species`.
- `Domain::is_placeholder()` checks zero memory footprint in both containers.
- Keep default constructors lightweight and non-allocating.

2. `Particles::npart()` must stay `<= maxnpart()`.
- `set_npart()` enforces this with `raise::ErrorIf`.
- MPI receive paths rely on this guard.

3. Sortedness contract matters.
- Particle pushers and injectors mark species unsorted (`set_unsorted()`).
- `RemoveDead()` compacts alive particles, retags contiguous alive/dead ranges,
  and sets `m_is_sorted = true`.
- Output paths call `RemoveDead()` when needed.

4. Constructor/call-site bool ordering is easy to misuse.
- `ParticleSpecies` uses `(use_tracking, use_gca)`.
- `Particles` constructor declaration currently places bools as
  `(use_gca, use_tracking)`.
- Prefer constructing from `ParticleSpecies` when possible and verify argument
  ordering carefully in direct constructor calls.

---

## IO And Communication Coupling

### Particle communication (`particles_comm.cpp`)
- Compiled only when `mpi=ON`.
- Uses `kernels/comm.hpp` for:
  - outgoing selection and coordinate shifts
  - send-buffer packing
  - receive-buffer extraction
- Buffer layout in kernels is positional; if you change particle member layout,
  update both pack and unpack kernels consistently.
- Tags are expected to be:
  - `dead`/`alive` or MPI send tags from `mpi::SendTag(...)`.
- `GPU_AWARE_MPI` and `DEVICE_ENABLED` guard host-mirror fallback behavior.

### Particle output/checkpoint (`particles_io.cpp`)
- Compiled only when `output=ON`.
- Variable naming is part of checkpoint/output compatibility:
  - output: `pX*_*`, `pU*_*`, `pW_*`, `pPLDR*_*`, `pPLDI*_*`, `pIDX_*`, `pRNK_*`
  - checkpoint: `s{idx}_*`
- Track payload mapping:
  - tracked species consume first integer payload slots (`pldi::spcCtr`,
    and `pldi::domIdx` under MPI).
- `OutputWrite` template instantiations are driven by
  `NTT_FOREACH_SPECIALIZATION`; keep specialization registry and output
  instantiation expectations aligned.

### Field checkpoint (`fields_io.cpp`)
- Declares/writes `em` for all engines.
- GR paths additionally checkpoint `em0` and `cur`.
- Uses `out::ReadNDField`/`WriteNDField`; keep field rank and component counts
  consistent with declaration shape.

---

## Build And Guard Semantics

From `src/framework/CMakeLists.txt`:
- Always built:
  - `containers/particles.cpp`
  - `containers/fields.cpp`
- Built only with `output=ON`:
  - `containers/fields_io.cpp`
  - `containers/particles_io.cpp`
- Built only with `mpi=ON`:
  - `containers/particles_comm.cpp`

Keep header guards and implementation guards consistent with this split:
- `OUTPUT_ENABLED`
- `MPI_ENABLED`
- `DEVICE_ENABLED`
- `GPU_AWARE_MPI`

---

## Change Guidance

### If you change particle members
Update all affected paths together:
1. allocation in `particles.cpp`
2. compaction in `RemoveDead()`
3. MPI pack/unpack in `kernels/comm.hpp` and `particles_comm.cpp`
4. output/checkpoint read/write in `particles_io.cpp`
5. memory footprint accounting

### If you change field components/counts
Update all affected paths together:
1. allocations in `fields.cpp`
2. checkpoint shape declarations/read/write in `fields_io.cpp`
3. comm/output callers that assume component ranges (`CommTags`, output reducers)

### If you add engine/metric/dimension support
Verify explicit instantiations in:
- `fields.cpp`
- `fields_io.cpp`
- `particles.cpp`
- `particles_comm.cpp` (MPI)
- `particles_io.cpp` (output)

---

## Tests To Run

Direct container coverage:
- `/Users/dbf75/Work/Research/AthenaK/entity/src/framework/tests/fields.cpp`
- `/Users/dbf75/Work/Research/AthenaK/entity/src/framework/tests/particles.cpp`

Related communication coverage:
- `/Users/dbf75/Work/Research/AthenaK/entity/src/framework/tests/comm_nompi.cpp`
- `/Users/dbf75/Work/Research/AthenaK/entity/src/framework/tests/comm_mpi.cpp`

Suggested commands:
```bash
cmake -B build -D TESTS=ON -D mpi=OFF -D output=ON
cmake --build build -j "$(nproc)"
ctest --test-dir build --output-on-failure -R '^FRAMEWORK::(fields|particles|comm_nompi)$'
```

```bash
cmake -B build -D TESTS=ON -D mpi=ON -D output=ON
cmake --build build -j "$(nproc)"
ctest --test-dir build --output-on-failure -R '^FRAMEWORK::comm_mpi$'
```

Note:
- There is no dedicated framework unit test for `particles_comm.cpp` or
  `particles_io.cpp` behavior; validate these paths with at least one
  MPI+output runtime scenario when modifying them.
