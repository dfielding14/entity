# AGENTS.md

## Scope
This file covers `/dev` only.

Use with:
- `/Users/dbf75/Work/Research/AthenaK/entity/AGENTS.md`

If instructions conflict, follow the closest `AGENTS.md` to the file being
edited.

---

## What `dev` Owns
`dev/` contains development environment tooling:
- Interactive Docker images for CUDA and ROCm development.
- Nix shell definitions for local reproducible toolchains.
- Dockerized GitHub self-hosted runner images and startup script.

This directory is infrastructure-focused. Changes here affect developer
workflows and CI execution environments, not simulation algorithms directly.

---

## Directory Map
- `Dockerfile.common`
  Shared Docker setup included by CUDA and ROCm dev images.
- `Dockerfile.cuda`, `Dockerfile.rocm`
  Interactive GPU dev images (with welcome script on shell login).
- `welcome.cuda`, `welcome.rocm`
  Startup UX scripts that print installed tool versions and usage hints.
- `nix/shell.nix`
  Main Nix dev shell entrypoint.
- `nix/kokkos.nix`
  Kokkos package derivation with GPU/arch-aware flags.
- `nix/adios2.nix`
  ADIOS2 package derivation with optional HDF5/MPI toggles.
- `runners/Dockerfile.runner.cpu`
- `runners/Dockerfile.runner.cuda`
- `runners/Dockerfile.runner.rocm`
  GitHub Actions self-hosted runner images.
- `runners/start.sh`
  Runner bootstrap, label-based AMD env setup, and cleanup trap.
- `runners/README.md`
  Manual commands to build and launch runner containers.

---

## Docker Dev Images

### Composition Model
- `Dockerfile.cuda` and `Dockerfile.rocm` use
  `# syntax = devthefuture/dockerfile-x` and `INCLUDE Dockerfile.common`.
- Keep shared package logic in `Dockerfile.common`; keep backend-specific logic
  in CUDA/ROCm files.
- If you add tooling needed by both images, prefer editing `Dockerfile.common`.

### Common Base Behavior
`Dockerfile.common` currently:
- Installs CMake 3.29.6 from Kitware tarball under `/opt`.
- Builds ADIOS2 from source at `/opt/adios2` with C++17 and examples/tests off.
- Installs developer CLI tools, Python 3.12, and a virtualenv at `/opt/venv`.
- Configures shell aliases and starship prompt for interactive sessions.

When changing this file:
- Keep CUDA and ROCm compatibility in mind.
- Keep ADIOS2 options aligned with expected runtime features.
- Keep PATH/env setup consistent with downstream Dockerfiles.

### Backend-Specific Notes
- `Dockerfile.cuda`
  - Base image: `nvidia/cuda:*`.
  - Sets `CUDA_HOME` and prepends CUDA bin path.
  - Adds `/root/.welcome.cuda` and runs it via `/etc/profile.d/welcome.sh`.
- `Dockerfile.rocm`
  - Base image: `rocm/rocm-terminal:latest`.
  - Installs extra ROCm dependencies (`rocThrust`, `rocPRIM`) from pinned tags.
  - Sets `CC=hipcc`, `CXX=hipcc`, and `CMAKE_PREFIX_PATH=/opt/rocm`.
  - Adds `/root/.welcome.rocm` and runs it at login.

If you bump ROCm package versions, update both cloned repos together and verify
they still build in the container.

---

## Nix Environments

### `nix/shell.nix` Contract
Main parameters:
- `gpu ? "NONE"`: expected values are `NONE`, `HIP`, `CUDA`.
- `arch ? "NATIVE"`: passed through to Kokkos package derivation.
- `hdf5 ? false`, `mpi ? false`: forwarded to ADIOS2 package derivation.

The shell:
- Pulls `adios2.nix` and `kokkos.nix` via `callPackage`.
- Exports compiler env vars from a GPU-keyed map
  (`CC/CXX` for `NONE` and `HIP`; no override for `CUDA`).
- Provides formatting/lint/editor tooling (`cmake-format`, `cmake-lint`,
  `taplo`, `black`, `pyright`, language server packages, etc.).

### `nix/kokkos.nix` Contract
- Builds Kokkos `4.7.01` from upstream git.
- Enables backend-specific CMake flags.
- Enforces explicit arch when GPU is enabled
  (`gpu != NONE && arch == NATIVE` throws).
- Uses `hipcc` for HIP and `nvcc_wrapper` for CUDA.

### `nix/adios2.nix` Contract
- Builds ADIOS2 `2.10.2` with C++17.
- Toggles HDF5 and MPI support from shell arguments.
- Generates post-install ADIOS2 config script output.

When changing Nix derivations:
- Preserve argument names used by `shell.nix`.
- Keep backend branches (`NONE`, `HIP`, `CUDA`) buildable.
- Keep version bumps and sha256 updates synchronized.

---

## Self-Hosted Runners

### Runner Image Behavior
`runners/Dockerfile.runner.{cpu,cuda,rocm}`:
- Build a container with compiler stack + ADIOS2.
- Create non-root `runner` user with passwordless sudo.
- Download GitHub Actions runner tarball (`RUNNER_VERSION=2.317.0`).
- Set container entrypoint to `./start.sh`.

The CUDA and ROCm runner images differ from CPU mainly by base image and GPU
toolchain setup.

### Startup Script Contract
`runners/start.sh`:
- Configures runner with:
  - `--url https://github.com/entity-toolkit/entity`
  - token from `${TOKEN}`
  - labels from `${LABEL}`
- Applies AMD runtime env vars when `LABEL == "amd-gpu"`.
- Registers traps to remove runner config on `INT`/`TERM`.
- Runs `./run.sh` and waits.

Do not change label semantics casually. Workflow routing often depends on these
exact labels.

### Operator Workflow
`runners/README.md` documents build/run commands for:
- `nvidia-gpu`
- `amd-gpu`
- `cpu`

If runner startup requirements change, update both:
1. `runners/start.sh`
2. `runners/README.md`

---

## Safe Change Points
1. Common developer package updates:
   `Dockerfile.common`, with compatibility checks in CUDA and ROCm images.
2. Nix package/version updates:
   `nix/adios2.nix`, `nix/kokkos.nix`, and `nix/shell.nix`.
3. Runner base image/toolchain updates:
   `runners/Dockerfile.runner.*`.
4. Runner registration/cleanup behavior:
   `runners/start.sh`.
5. User-facing startup guidance:
   `welcome.cuda`, `welcome.rocm`, `runners/README.md`.

---

## Validation
For Docker changes, run the relevant image build(s):
```bash
docker build -f dev/Dockerfile.cuda -t entity-dev:cuda dev
docker build -f dev/Dockerfile.rocm -t entity-dev:rocm dev
```

For runner changes, run at least one runner image build:
```bash
docker build -f dev/runners/Dockerfile.runner.cpu -t entity-runner:cpu dev/runners
```

For Nix changes, open shell variants you touched:
```bash
nix-shell dev/nix/shell.nix --argstr gpu NONE
nix-shell dev/nix/shell.nix --argstr gpu CUDA --argstr arch AMPERE80
nix-shell dev/nix/shell.nix --argstr gpu HIP --argstr arch AMD_GFX90A
```

---

## Common Failure Modes
- Editing `Dockerfile.cuda`/`Dockerfile.rocm` without considering shared
  behavior in `Dockerfile.common`.
- Breaking `INCLUDE Dockerfile.common` usage by removing or changing the custom
  Dockerfile frontend line.
- Enabling GPU mode in Nix shell with `arch=NATIVE` (explicit arch is required).
- Bumping package versions without updating related pinned revisions/hashes.
- Changing runner labels in `start.sh` without updating workflow expectations.
- Updating runner setup logic without reflecting new requirements in
  `runners/README.md`.
