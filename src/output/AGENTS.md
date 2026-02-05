# AGENTS.md

## Scope
This file covers `/src/output` only.

Use with:
- `/Users/dbf75/Work/Research/AthenaK/entity/AGENTS.md`
- `/Users/dbf75/Work/Research/AthenaK/entity/src/AGENTS.md`

If instructions conflict, follow the closest `AGENTS.md` to the edited file.

---

## What `output` owns
`output` owns output metadata parsing and writer helpers for:
1. Field/particle/spectra ADIOS2 outputs.
2. Checkpoint file lifecycle and retention.
3. CSV stats metadata and writing.
4. Generic ADIOS read/write helpers used by checkpoint I/O paths.

Build behavior (`/Users/dbf75/Work/Research/AthenaK/entity/src/output/CMakeLists.txt`):
- `ntt_output` always builds `stats.cpp`, `fields.cpp`,
  `utils/interpret_prompt.cpp`.
- ADIOS-heavy files (`writer.cpp`, `checkpoint.cpp`, `utils/writers.cpp`,
  `utils/readers.cpp`) are compiled only when `-D output=ON`.

---

## Directory map
- `writer.h/.cpp`
  Main ADIOS output writer for fields, particles, and spectra.
- `checkpoint.h/.cpp`
  Dedicated checkpoint writer with retention (`checkpoint.keep`).
- `fields.h/.cpp`
  Output field prompt parsing into `FldsID`, components, species, and
  `PrepareOutput`/interpolation flags.
- `stats.h/.cpp`
  Stats prompt parsing (`StatsID`) plus CSV stats writer/tracker.
- `spectra.h`
  Simple spectra output metadata wrapper.
- `utils/interpret_prompt.h/.cpp`
  Shared parser for species and component selectors.
- `utils/writers.h/.cpp`, `utils/readers.h/.cpp`
  Generic ADIOS+Kokkos view serialization helpers (used by framework container
  checkpoint/readback paths).
- `utils/attr_writer.h`
  `std::any` attribute writing helper for supported scalar/vector types.
- `tests/`
  Module tests (`OUTPUT::*`).

---

## Primary integration points
`output` is orchestrated from framework domain code:
- `/Users/dbf75/Work/Research/AthenaK/entity/src/framework/domain/output.cpp`
- `/Users/dbf75/Work/Research/AthenaK/entity/src/framework/domain/checkpoint.cpp`
- `/Users/dbf75/Work/Research/AthenaK/entity/src/framework/domain/stats.cpp`
- `/Users/dbf75/Work/Research/AthenaK/entity/src/framework/containers/fields_io.cpp`
- `/Users/dbf75/Work/Research/AthenaK/entity/src/framework/containers/particles_io.cpp`

If you change writer/checkpoint/read/write signatures, update these call sites
in the same change.

---

## Core contracts and invariants

### Metadomain-side assumptions
- Output/checkpoint/stats currently require exactly one local subdomain per rank.
- Local domain must not be a placeholder.
- These assumptions are enforced in framework domain call sites and should not
  be weakened in only one subsystem.

### `out::Writer` usage order
Required flow:
1. `init(...)`
2. `defineMeshLayout(...)`
3. `defineFieldOutputs(...)` / `defineSpectraOutputs(...)`
4. For each step/mode: `beginWriting(mode, step, time)` -> write calls ->
   `endWriting(mode)`

Rules:
- `defineFieldOutputs` requires mesh layout already defined.
- Only one active mode at a time (`WriteMode::Fields`, `Particles`, `Spectra`).
- `beginWriting`/`endWriting` mode tags must match.

### Mesh/downsampling constraints
- `defineMeshLayout` rejects downsampling when ghosts are requested:
  `dwn[i] != 1 && incl_ghosts` is invalid.
- Layout metadata includes `NGhosts`, `Dimension`, `Coordinates`, and
  `LayoutRight`; keep these stable unless coordinated with readers/tests.

### Field/stat prompt parsing
- Prompt parsing routes through `FldsID`/`StatsID` + `InterpretSpecies` and
  `InterpretComponents`.
- `Custom` names bypass enum parsing and are passed through.
- SR/GR compatibility checks are explicit (example: `A_phi` is GRPIC-only;
  bulk velocity output is not supported for GRPIC).

### Checkpoint writer behavior
- `checkpoint::Writer::init(...)` disables checkpointing when `keep == 0`.
- Checkpoint filenames are `step-%08lu.bp`; metadata filenames are
  `meta-%08lu.toml`.
- `endSaving()` removes oldest checkpoint pair when `keep > 0` and capacity is
  exceeded.
- Preserve filename and retention semantics unless resume logic is updated
  together.

### Stats writer behavior
- Stats output is CSV and append-based.
- `writeHeader()` emits `step,time,...` column names.
- `write(...)` uses MPI reduction when requested (`communicate=true`) and
  writes via `CallOnce` to avoid multi-rank file races.

---

## ADIOS/Kokkos data movement expectations
- Writer/readers generally mirror device views to host before `Put/Get`.
- `Write1DSubArray`/`Write2DArray` and matching read helpers handle non-
  contiguous host spans explicitly; keep this behavior if changing layout logic.
- Spectra write path reduces on MPI root and writes empty selections on
  non-root ranks.

---

## Change guidance

### Safe change points
1. Prompt parsing/name mapping:
   `/Users/dbf75/Work/Research/AthenaK/entity/src/output/fields.cpp`,
   `/Users/dbf75/Work/Research/AthenaK/entity/src/output/stats.cpp`,
   `/Users/dbf75/Work/Research/AthenaK/entity/src/output/utils/interpret_prompt.cpp`
2. ADIOS variable/attribute declarations:
   `/Users/dbf75/Work/Research/AthenaK/entity/src/output/writer.cpp`,
   `/Users/dbf75/Work/Research/AthenaK/entity/src/output/checkpoint.cpp`
3. Generic checkpoint serializers:
   `/Users/dbf75/Work/Research/AthenaK/entity/src/output/utils/writers.cpp`,
   `/Users/dbf75/Work/Research/AthenaK/entity/src/output/utils/readers.cpp`

### High-risk edits (coordinate with framework)
- Renaming output variable names/attributes.
- Changing checkpoint filename patterns or retention logic.
- Changing mesh layout/downsampling rules.
- Changing component/species parsing grammar.

---

## Tests
Defined in `/Users/dbf75/Work/Research/AthenaK/entity/src/output/tests/CMakeLists.txt`:
- Always: `OUTPUT::stats`
- When `output=ON` and `mpi=OFF`: `OUTPUT::fields`, `OUTPUT::writer-nompi`
- When `output=ON` and `mpi=ON`: `OUTPUT::writer-mpi` (runs via `mpiexec -n 4`)

Recommended commands:
```bash
cmake -B build -D TESTS=ON -D output=ON -D mpi=OFF
cmake --build build -j "$(nproc)"
ctest --test-dir build --output-on-failure -R '^OUTPUT::'
```

```bash
cmake -B build -D TESTS=ON -D output=ON -D mpi=ON
cmake --build build -j "$(nproc)"
ctest --test-dir build --output-on-failure -R '^OUTPUT::'
```

---

## Common failure modes
- Calling `defineFieldOutputs` before `defineMeshLayout`.
- Mismatched `beginWriting(...)` / `endWriting(...)` modes.
- Enabling ghosts and downsampling simultaneously.
- Forgetting 1-based species indexing in output prompts (`_1`, `_2`, ...).
- Requesting custom field/stats output without providing callback functions in
  framework write calls.
- Breaking GRPIC/SRPIC-specific output compatibility checks.
- Updating checkpoint naming without updating resume/readback assumptions.

---

## Related docs
- `/Users/dbf75/Work/Research/AthenaK/entity/src/framework/AGENTS.md`
- `/Users/dbf75/Work/Research/AthenaK/entity/src/framework/domain/AGENTS.md`
- `/Users/dbf75/Work/Research/AthenaK/entity/src/framework/containers/AGENTS.md`
