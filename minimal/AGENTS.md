# AGENTS.md

## Scope
This file governs work in `/minimal`.

Use with:
- `/Users/dbf75/Work/Research/AthenaK/entity/AGENTS.md`

If instructions conflict, follow the closest `AGENTS.md` to the file being
edited.

---

## Purpose of `minimal/`
`minimal/` is a standalone third-party validation harness, separate from the
main `Entity` runtime build in `/src`.

Use it to isolate toolchain/runtime issues in:
1. Kokkos allocation and kernel execution.
2. MPI send/recv behavior with Kokkos Views.
3. ADIOS2 writing paths with and without MPI.

This subtree is for dependency/environment sanity checks, not physics
validation.

---

## Directory map
- `/Users/dbf75/Work/Research/AthenaK/entity/minimal/CMakeLists.txt`
  Standalone CMake project and mode selection (`MODES`).
- `/Users/dbf75/Work/Research/AthenaK/entity/minimal/README.md`
  Basic usage overview.
- `/Users/dbf75/Work/Research/AthenaK/entity/minimal/kokkos.cpp`
  Kokkos-only allocation and kernel fill test.
- `/Users/dbf75/Work/Research/AthenaK/entity/minimal/mpi-simple.cpp`
  Lightweight MPI `Sendrecv` test over 2D Kokkos Views.
- `/Users/dbf75/Work/Research/AthenaK/entity/minimal/mpi.cpp`
  Heavier multi-dimension/multi-type MPI halo-style communication test.
- `/Users/dbf75/Work/Research/AthenaK/entity/minimal/adios2.cpp`
  ADIOS2 output test for constant/unknown dimensions and multi-step writes.

---

## Build and run model
`minimal/` is a separate CMake project. Configure it directly:

```bash
cmake -S minimal -B build/minimal -D MODES="KOKKOS;ADIOS2_NOMPI"
cmake --build build/minimal -j "$(nproc)"
```

Run outputs from `build/minimal/`:
- `kokkos.xc`
- `mpi-simple.xc`
- `adios2-nompi.xc`
- `adios2-mpi.xc`

For MPI executables, launch with `mpirun`/`mpiexec`:

```bash
mpirun -n 2 build/minimal/mpi-simple.xc
mpirun -n 2 build/minimal/adios2-mpi.xc bp
```

---

## Minimal modes and when to use each

### `KOKKOS`
Builds:
- `kokkos.xc` from `kokkos.cpp`

Use when:
- checking basic Kokkos initialization and execution-space configuration.
- verifying device/host allocation and simple `parallel_for` behavior.
- debugging failures before introducing MPI or ADIOS2.

### `MPI_SIMPLE`
Builds:
- `mpi-simple.xc` from `mpi-simple.cpp`

Use when:
- you need a fast MPI+Kokkos smoke test.
- validating GPU-aware MPI vs host-staging fallback behavior.
- checking rank-to-rank `MPI_Sendrecv` with rectangular Kokkos subviews.

### `MPI`
Current CMake wiring also builds:
- `mpi-simple.xc` from `mpi-simple.cpp`

Use when:
- you want the same lightweight MPI test path as `MPI_SIMPLE`.

Current-state note:
- `mpi.cpp` is present but not currently wired to any build mode in
  `minimal/CMakeLists.txt`.

### `ADIOS2_NOMPI`
Builds:
- `adios2-nompi.xc` from `adios2.cpp`

Use when:
- validating ADIOS2 engine setup without MPI (`hdf5` or `bp`).
- checking ADIOS2 variable/attribute definitions and multi-step writes.
- isolating ADIOS2 install/runtime issues from MPI transport problems.

### `ADIOS2_MPI`
Builds:
- `adios2-mpi.xc` from `adios2.cpp` with `MPI_ENABLED`

Use when:
- validating distributed ADIOS2 writes and per-rank selections.
- checking global-shape/offset handling for unknown-dimension arrays.
- reproducing issues that appear only with ADIOS2 + MPI integration.

---

## Important current constraints
- `MPI` and `MPI_SIMPLE` currently define the same executable target
  (`mpi-simple.xc`) in `minimal/CMakeLists.txt`.
- If both modes are requested together, CMake target redefinition may occur.
- `mpi.cpp` is currently unused by mode selection despite being a stronger MPI
  stress test implementation.

Document current behavior first. If you change mode wiring, update:
1. `minimal/CMakeLists.txt`
2. `minimal/README.md`
3. this file

in the same change.

---

## Dependency resolution behavior
`minimal/CMakeLists.txt` tries:
1. `find_package(Kokkos)` / `find_package(adios2)`
2. `FetchContent` from upstream repos if not found

MPI is required only for `MPI`, `MPI_SIMPLE`, and `ADIOS2_MPI`.

When debugging cluster issues, prefer preinstalled packages first and use
`FetchContent` only when needed.

---

## Guard and runtime conventions in this subtree
- MPI paths are guarded by `MPI_ENABLED` in `adios2.cpp`.
- MPI transfer path in MPI tests toggles device-direct vs host-mirror behavior
  based on `GPU_AWARE_MPI` and `DEVICE_ENABLED`.
- ADIOS2 executable accepts optional engine argument:
  - `hdf5` (default engine in code)
  - `bp`

Keep both guarded branches buildable and runnable after edits.

---

## Safe change points
1. Add a new minimal mode:
   - add mode block in `minimal/CMakeLists.txt`
   - map to a unique executable target
   - document in `minimal/README.md`
2. Extend an existing test payload:
   - keep it minimal and dependency-focused
   - avoid importing Entity framework/src abstractions
3. Adjust MPI copy behavior:
   - preserve both GPU-aware and host-staging branches
   - avoid assumptions about contiguous layouts without explicit checks

---

## Validation workflow after changes
1. Configure the specific mode(s) you touched.
2. Build from a clean-ish build directory.
3. Run each produced executable at least once.
4. For MPI paths, run with `-n 2` minimum and one larger rank count when
   practical.
5. For ADIOS2 paths, verify expected output artifacts are produced (`steps/`,
   `allsteps.h5` or `allsteps.bp`).

---

## Common failure modes
- Configuring from repo root without `-S minimal` and accidentally building the
  main project instead of minimal tests.
- Enabling both `MPI` and `MPI_SIMPLE` simultaneously with current duplicate
  target naming.
- Assuming `mpi.cpp` is exercised by current `MODES` wiring.
- GPU-aware MPI enabled on systems that require host staging.
- ADIOS2 engine mismatch (`hdf5` vs `bp`) or missing backend support.
