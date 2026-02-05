# AGENTS.md

## Scope
This file covers `/src/kernels` only.

Use with:
- `/Users/dbf75/Work/Research/AthenaK/entity/AGENTS.md`
- `/Users/dbf75/Work/Research/AthenaK/entity/src/AGENTS.md`

If instructions conflict, follow the closest `AGENTS.md` to the edited file.

---

## What `kernels` owns
`kernels` is the low-level compute layer: mostly header-only Kokkos functors
that implement per-cell and per-particle math used by engines, framework
communication, output transforms, and archetype injectors.

CMake shape (`src/kernels/CMakeLists.txt`):
- `ntt_kernels` is an `INTERFACE` library.
- There are no `.cpp` sources in this module; behavior lives in `.hpp` files.
- New kernel headers usually do not require CMake edits unless you add tests.

---

## Directory map

### Field solver kernels
- `ampere_mink.hpp`, `ampere_sr.hpp`, `ampere_gr.hpp`
- `faraday_mink.hpp`, `faraday_sr.hpp`, `faraday_gr.hpp`
- `aux_fields_gr.hpp` (GR auxiliary E/H + time-averaging helpers)

### Particle evolution and deposition kernels
- `particle_pusher_sr.hpp`, `particle_pusher_gr.hpp`
- `currents_deposit.hpp`
- `particle_moments.hpp`
- `particle_shapes.hpp`
- `injectors.hpp`

### Boundary and communication kernels
- `fields_bcs.hpp`
- `comm.hpp`

### Output/stats/diagnostic kernels
- `digital_filter.hpp`
- `divergences.hpp`
- `fields_to_phys.hpp`
- `prtls_to_phys.hpp`
- `reduced_stats.hpp`
- `utils.hpp`

### Tests
- `/Users/dbf75/Work/Research/AthenaK/entity/src/kernels/tests`
- CTest names follow `KERNELS::<name>` from
  `/Users/dbf75/Work/Research/AthenaK/entity/src/kernels/tests/CMakeLists.txt`.

---

## Namespace conventions
Common namespace split:
- `kernel::mink`
  Cartesian Minkowski specializations.
- `kernel::sr`
  Curvilinear SRPIC specializations.
- `kernel::gr`
  GRPIC specializations.
- `kernel::bc`
  Boundary-condition kernels (with GR-specific sub-areas in this header).
- `kernel::comm`
  Particle communication packing/unpacking kernels.
- `kernel`
  Generic kernels and output/stat transforms.

Keep new kernels in the smallest fitting namespace to avoid mixed SR/GR logic.

---

## Integration points outside this folder
Primary call sites:
- `/Users/dbf75/Work/Research/AthenaK/entity/src/engines/srpic.hpp`
- `/Users/dbf75/Work/Research/AthenaK/entity/src/engines/grpic.hpp`
- `/Users/dbf75/Work/Research/AthenaK/entity/src/framework/containers/particles_comm.cpp`
- `/Users/dbf75/Work/Research/AthenaK/entity/src/framework/domain/output.cpp`
- `/Users/dbf75/Work/Research/AthenaK/entity/src/framework/domain/stats.cpp`
- `/Users/dbf75/Work/Research/AthenaK/entity/src/framework/containers/particles_io.cpp`
- `/Users/dbf75/Work/Research/AthenaK/entity/src/archetypes/particle_injector.h`

Rule:
- Validate behavior at the call site after kernel edits, especially when a
  kernel constructor takes metric, BC tags, or field component mappings.

---

## Kernel contracts and invariants

### Template and type contracts
- Many kernels require `M::is_metric` via `static_assert`.
- Dimension assumptions are explicit and strict (`Dim::_1D`, `_2D`, `_3D`).
- Some kernels constrain enums/IDs at compile time (e.g. moments/stats IDs).
- Deposition shape order is compile-time constrained (`O <= 11`).

### Runtime safety checks
- Use `raise::ErrorIf(...)` for constructor argument validation.
- Use `raise::KernelError(...)` for invalid runtime paths in device kernels.
- Keep existing guard behavior; do not silently return on invalid dimension or
  engine branches.

### Data and memory conventions
- Kernels pass Kokkos views/aliases (`ndfield_t`, `array_t`, `scatter_ndfield_t`)
  by value into functors.
- Current deposition uses `Kokkos::Experimental::create_scatter_view(...)` and
  requires matching `Kokkos::Experimental::contribute(...)` at call sites.
- Particle-tag semantics matter (`alive`, `dead`, outbound tags); preserve tag
  transitions in pusher/comm/injector kernels.

---

## Kokkos and device-code rules
- Prefer project aliases/macros from `arch/kokkos_aliases.h` (`Inline`,
  `Lambda`, `CreateRangePolicy`, etc.).
- Keep `operator()` paths device-safe:
  no host I/O, no exceptions, no host-only containers in kernel body.
- Use `if constexpr` for dimension/engine branching when possible.
- Keep overload sets consistent for rank-1/rank-2/rank-3 policies when a kernel
  supports multiple dimensions.

---

## Change guidance

### Adding or changing a kernel
1. Choose namespace (`mink`/`sr`/`gr`/`bc`/`comm`/generic) first.
2. Encode compile-time constraints with `static_assert`.
3. Validate constructor arguments with `raise::ErrorIf`.
4. Preserve BC/axis/horizon logic where applicable.
5. Update every call site that depends on constructor signature changes.

### Adding a new test
- Add `<name>.cpp` under
  `/Users/dbf75/Work/Research/AthenaK/entity/src/kernels/tests`.
- Register it in
  `/Users/dbf75/Work/Research/AthenaK/entity/src/kernels/tests/CMakeLists.txt`
  via `gen_test(<name>)`.

---

## Testing expectations
Kernel tests are included by `cmake/tests.cmake` only in non-MPI test builds.

Recommended workflow:
```bash
cmake -B build -D TESTS=ON -D mpi=OFF -D output=ON
cmake --build build -j "$(nproc)"
ctest --test-dir build --output-on-failure -R '^KERNELS::'
```

If kernel changes affect engine integration, also run broader module tests:
```bash
ctest --test-dir build --output-on-failure -R '^(KERNELS|FRAMEWORK|OUTPUT)::'
```

---

## Common failure modes in this subtree
- Calling the wrong dimension overload for a range policy rank.
- Breaking staggered-field indexing (edge/face/cell-center assumptions).
- Forgetting scatter-view contribution after current deposition.
- Changing particle-tag logic and breaking communication/pusher handoff.
- Editing axis/horizon/conductor boundary logic without matching engine-side
  boundary sequencing.
- Introducing host-only operations into device functors.

---

## Related docs
- `/Users/dbf75/Work/Research/AthenaK/entity/src/engines/AGENTS.md` (planned)
- `/Users/dbf75/Work/Research/AthenaK/entity/src/framework/AGENTS.md`
- `/Users/dbf75/Work/Research/AthenaK/entity/src/framework/domain/AGENTS.md`
