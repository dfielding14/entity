# AGENTS.md

## Scope
This file covers `/src/metrics` only.

Use with:
- `/Users/dbf75/Work/Research/AthenaK/entity/AGENTS.md`
- `/Users/dbf75/Work/Research/AthenaK/entity/src/AGENTS.md`

If instructions conflict, follow the closest `AGENTS.md` to the file you are
editing.

---

## What This Layer Owns
`src/metrics` defines the geometry layer used by framework, kernels, engines,
and archetypes.

It currently contains:
- `metric_base.h`
  Base metric interface (`MetricBase<D>`) with shared geometry metadata and
  `dxMin()` contract.
- `minkowski.h`
  Cartesian flat-space metric.
- `spherical.h`, `qspherical.h`
  SR spherical and quasi-spherical metrics.
- `kerr_schild.h`, `qkerr_schild.h`, `kerr_schild_0.h`
  GR axisymmetric metric variants.
- `tests/`
  Module tests registered as `METRICS::*`.

Build model:
- `ntt_metrics` is an `INTERFACE` target in `src/metrics/CMakeLists.txt`.
- Keep this module effectively header-only unless there is a strong reason not
  to.
- `ntt_metrics` depends on `ntt_global` only; avoid adding higher-layer module
  dependencies here.

---

## Supported Metric Matrix
Allowed engine/metric/dimension tuples are declared in
`/Users/dbf75/Work/Research/AthenaK/entity/src/framework/specialization_registry.h`.
Current matrix:

- `SRPIC + Minkowski + 1D`
- `SRPIC + Minkowski + 2D`
- `SRPIC + Minkowski + 3D`
- `SRPIC + Spherical + 2D`
- `SRPIC + QSpherical + 2D`
- `GRPIC + KerrSchild + 2D`
- `GRPIC + QKerrSchild + 2D`
- `GRPIC + KerrSchild0 + 2D`

Per-class constraints:
- `Minkowski<D>` supports `D = 1, 2, 3`.
- `Spherical<D>`, `QSpherical<D>`, `KerrSchild<D>`, `QKerrSchild<D>`,
  `KerrSchild0<D>` currently static-assert out for `D = 1` and `D = 3`.

Do not assume a metric is usable just because it compiles:
- Runtime dispatch uses specialization registry entries.
- `user::PGen<S, M>` compatibility traits still gate launch at compile/runtime.

---

## Metric Contract

### Core Interface (all metrics)
All metric classes are consumed as template types (no virtual dispatch in hot
paths). The practical contract used across framework/kernels includes:

- Static metadata:
  - `is_metric`, `Dim`, `PrtlDim`, `CoordType`, `MetricType`, `Label`.
- Constructor:
  - `Metric(resolution, extent, metric_params_map)`.
- Scalar geometry:
  - `dxMin()`, `find_dxMin()`, `totVolume()`.
  - `h_<i,j>(x_code)`, `sqrt_h_<i,j>(x_code)`, `sqrt_det_h(x_code)`.
- Coordinate transforms:
  - `convert<i, in, out>(real_t)` and `convert<in, out>(coord_t<Dim>)`.
  - `convert_xyz<in, out>(coord_t<PrtlDim>)` for Cartesian bridging.
- Vector transforms:
  - `transform<i, in, out>(x_code, scalar)` and
    `transform<in, out>(x_code, vec3)`.
  - `transform_xyz<in, out>(x_code_prtl, vec3)` for Cartesian bridging.

### GR-Required Extensions
GR kernels additionally require:
- `h<i,j>(x_code)` (upper-index spatial metric)
- `alpha(x_code)`, `beta1(x_code)`
- Derivatives used in pusher equations:
  - `dr_alpha`, `dt_alpha`
  - `dr_beta1`, `dt_beta1`
  - `dr_h11`, `dr_h22`, `dr_h33`, `dr_h13`
  - `dt_h11`, `dt_h22`, `dt_h33`, `dt_h13`
- Axisymmetric helpers used by GR field updates:
  - `sqrt_det_h_tilde(x_code)`, `polar_area(x1_code)`

If you add/modify GR metrics, verify against:
- `/Users/dbf75/Work/Research/AthenaK/entity/src/kernels/particle_pusher_gr.hpp`
- `/Users/dbf75/Work/Research/AthenaK/entity/src/kernels/aux_fields_gr.hpp`
- `/Users/dbf75/Work/Research/AthenaK/entity/src/kernels/ampere_gr.hpp`
- `/Users/dbf75/Work/Research/AthenaK/entity/src/kernels/faraday_gr.hpp`

---

## Coordinate And Transform Conventions
- Coordinate systems are encoded by `Crd` (`Cd`, `Ph`, `XYZ`, `Sph`) and
  metric-level `CoordType` (`cart`, `sph`, `qsph` via `ntt::Coord`).
- Vector index spaces are encoded by `Idx` (`U`, `D`, `T`, `XYZ`, `Sph`,
  `PU`, `PD`).
- For non-Cartesian metrics, `PrtlDim` is intentionally not always equal to
  `Dim` (for example SR spherical variants use `PrtlDim = 3` in 2D runs).
  Preserve this behavior when changing transform code.
- Keep transform paths invertible where expected:
  - `Cd <-> Ph`
  - `Cd <-> Sph`
  - `Cd <-> XYZ` via `convert_xyz`
  - corresponding vector transforms

---

## Parameter Wiring And Compatibility Rules
Runtime metric setup is centered in
`/Users/dbf75/Work/Research/AthenaK/entity/src/framework/parameters.cpp`.

Current behavior:
- `grid.metric.metric` selects enum `ntt::Metric`.
- Metric choice infers/stores `grid.metric.coord`.
- Quasi-spherical metrics use:
  - `grid.metric.qsph_r0`
  - `grid.metric.qsph_h`
- GR Kerr metrics use:
  - `grid.metric.ks_a` (except `Kerr_Schild_0`)
- A compact map is stored at `grid.metric.params` and passed to metric
  constructors.

Domain consistency:
- `Metadomain::metricCompatibilityCheck()` enforces consistent `dxMin()`
  across all local domains and MPI ranks.
- Changes to `find_dxMin()` should preserve this invariant for decomposed runs.

Defaults:
- `qsph_r0`, `qsph_h`, `ks_a` defaults live in
  `/Users/dbf75/Work/Research/AthenaK/entity/src/global/defaults.h`.

---

## Change Checklist
When adding a metric or changing metric semantics:

1. Implement/update metric header in `/Users/dbf75/Work/Research/AthenaK/entity/src/metrics`.
2. Add/update enum entries in `/Users/dbf75/Work/Research/AthenaK/entity/src/global/enums.h`.
3. Wire parameter parsing and inferred params in
   `/Users/dbf75/Work/Research/AthenaK/entity/src/framework/parameters.cpp`.
4. Keep `input.example.toml` metric docs in sync.
5. Register valid engine/metric/dimension tuples in
   `/Users/dbf75/Work/Research/AthenaK/entity/src/framework/specialization_registry.h`.
6. Confirm target problem generators declare matching compatibility traits.
7. Add/update tests in `/Users/dbf75/Work/Research/AthenaK/entity/src/metrics/tests`.
8. Run at least metrics tests and any affected framework tests.

---

## Tests To Run
Metrics module tests:
- `METRICS::minkowski`
- `METRICS::vec_trans`
- `METRICS::coord_trans`
- `METRICS::sph-qsph`
- `METRICS::ks-qks`
- `METRICS::sr-cart-sph`

Suggested workflow:
```bash
cmake -B build -D TESTS=ON -D mpi=OFF -D output=ON
cmake --build build -j "$(nproc)"
ctest --test-dir build --output-on-failure -R '^METRICS::'
```

If metric parsing/dispatch changed:
```bash
ctest --test-dir build --output-on-failure -R '^FRAMEWORK::(parameters|grid_mesh)$'
```

Note:
- `METRICS::vec_trans` currently exercises a narrower subset of metrics than
  other metric tests; treat it as partial coverage.

---

## Common Failure Modes
- Adding a metric class but forgetting specialization registry entries.
- Adding a metric enum but not wiring parameter parsing and docs.
- Missing required keys in `grid.metric.params` (constructor `params.at(...)`
  throws at runtime).
- Breaking `PrtlDim`/`CoordType` assumptions used by particle and injector
  kernels.
- Using host-only behavior in code paths that run in `KOKKOS_LAMBDA`.
- Changing `find_dxMin()` in a way that violates cross-domain consistency.
