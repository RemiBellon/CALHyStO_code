# CALHYSTO

Full-wave finite-element solver for the coupling of **lower hybrid (LH) waves** in tokamak
edge plasmas, built on [NGSolve](https://ngsolve.org).

CALHYSTO is an open-source successor of the ALOHA coupling code
(Hillairet *et al.*, *Nucl. Fusion* **50** (2010) 125010). Its goal is to handle
**arbitrary electron density profiles** — measured, steep or non-monotonic — in front of
LH antennas, first in 2D, then in 3D.

> ⚠️ **Status: pre-alpha.** The project is built incrementally, starting from the
> theoretical foundations. No solver is available yet. The [roadmap](#6-roadmap) states
> what is actually implemented and verified.

---

## 1. Installation

The project uses [uv](https://docs.astral.sh/uv/). Supported platforms:
**Linux x86-64** and **macOS** (Intel and Apple Silicon). Python **3.14**.

```bash
# Install uv (once)
curl -LsSf https://astral.sh/uv/install.sh | sh

# Get the code and create the environment
git clone <repository-url> calhysto
cd calhysto
uv python install 3.14
uv sync
```

`uv sync` creates `.venv/`, resolves the versions recorded in `uv.lock`, installs
`calhysto` in editable mode, and installs the development tools (pytest, ruff).
`uv.lock` is versioned: it guarantees that every contributor gets the same environment.

Check the installation:

```bash
uv run python -c "import ngsolve; print(ngsolve.__version__)"
```

**Platform notes**

- NGSolve ships Python 3.14 wheels for macOS (`universal2`) and Linux `x86_64` only.
  Other platforms require building NGSolve from source.
- Never put an upper bound on `ngsolve`: older releases have no Python 3.14 wheels and uv
  would fall back to a source build.

---

## 2. Design principles

- **Rigour before speed.** Every numerical method is introduced in its simplest standard
  form first. A more complex method is adopted only after the simple one has been shown,
  by a measured test, to fail.
- **One simulation = one `.yaml` file** describing the whole case (geometry, plasma,
  absorbing layers, mesh, solver). *Planned for roadmap step 2.*
- **The user drives.** Runs are launched from `main.py`, a script the user edits.
  Nothing runs in cascade.
- **Computation and figures are decoupled.** A run writes a self-contained, timestamped
  results folder; figures are produced afterwards from that folder only, so a figure can
  be redrawn without re-running the solver. *Planned for roadmap step 2.*

Target usage:

```python
from calhysto import plotting, run

result_dir = run.sim_2D("configs/my_case.yaml")
plotting.power_spectrum(result_dir)
```

---

## 3. Code layout

```
calhysto/
├── main.py              # driver script, edited by the user
├── configs/             # .yaml simulation cases
├── experiments/         # self-contained studies comparing numerical methods
├── calhysto/
│   ├── config/          # .yaml schema and validated loading
│   ├── geometry/        # computational domain, waveguides, antenna (PAM/FAM), mesh
│   ├── plasma/          # density profiles, magnetic field, cold Stix tensor
│   ├── pml/             # absorbing layers
│   ├── fem/             # FE spaces, bilinear forms, boundary conditions, linear solver
│   ├── run/             # orchestration: sim_2D, sim_3D, parameter scans
│   ├── io/              # timestamped result folders, xarray read/write
│   └── plotting/        # figures, from a results folder only
└── tests/               # mirrors calhysto/: one test file per module
```

Dependency rules: `config` depends on nothing else; `plasma` produces NGSolve
`CoefficientFunction`s but never builds a mesh or a FE space; `pml` knows the geometry and
the medium, not the solver; `fem` is the only layer that assembles and solves; `run` is the
only layer that orchestrates and writes to disk; `plotting` only reads results folders.

**Experiments.** Each folder `experiments/Exx_<name>/` answers *one* question (for
instance: Cartesian versus circular-arc PMLs) with a quantitative criterion fixed before
running it. `calhysto` never imports `experiments`. A method that is retained is rewritten
in `calhysto/` with its tests.

---

## 4. Physical and numerical conventions

| Topic | Convention |
|---|---|
| Time dependence | `exp(-i ω t)` — fixes the sign of losses and of the PML stretching |
| Imaginary unit | `i`; the letter `j` is reserved for current density |
| Units | SI everywhere; conversions only when reading input files |
| Finite elements | Nédélec edge elements, `HCurl(mesh, order=..., complex=True)` |
| Linear solver | direct solver UMFPACK (`inverse="umfpack"`); the matrix is complex, non-symmetric, non-Hermitian. `sparsecholesky` is **forbidden**: it assumes symmetry and returns wrong results silently |
| Dielectric tensor | cold-plasma Stix tensor (`S`, `D`, `P`), built as a compiled NGSolve `CoefficientFunction` |
| PML | written explicitly in the variational formulation (complex coordinate stretching with a directional sign); NGSolve's built-in PMLs are not used |
| Lower hybrid resonance | `S = 0` is not tractable in a collisionless cold plasma; the domain is truncated before it, explicitly and in the input file |

Some reference papers use the `exp(+j ω t)` convention (for instance ALOHA). Comparing
with them requires replacing `i` by `-j` (complex conjugation).

---

## 5. Tests

Every test compares the result with a truth **independent of the code**, in order of
strength: analytical solution, method of manufactured solutions (observed convergence order
versus the theoretical order of Nédélec elements), physical invariants (energy,
reciprocity, symmetries), published references (ALOHA, OLGA, TOPLHA, WEST measurements),
and finally non-regression. Numerical tolerances are explicit and justified.

```bash
uv run pytest -m "not slow"     # fast tests, at each change
uv run pytest                   # everything
uv run ruff check --fix . && uv run ruff format .
```

Markers: `slow`, `convergence`, `physics`.

---

## 6. Roadmap

Each step is fully verified before the next one starts.

| # | Step | Passing criterion | Status |
|---|---|---|---|
| 0 | Theoretical foundations | every upstream choice (Nédélec elements, NGSolve, `exp(-iωt)`, UMFPACK) is justified | 🟡 |
| 1a | 2D plane wave in vacuum, waveguides infinite vertically | geometry and mesh checked in ParaView; convergence order matches theory (MMS) | 🔲 |
| 1b | Standard PML in vacuum | reflection decreases with refinement, in propagating and evanescent regimes | 🔲 |
| 1c | Wave packet | coherent superposition, no spurious reflection | 🔲 |
| 2 | Skeleton: `.yaml` configuration, results I/O | configuration round-trip and results folder, tested | 🔲 |
| 3a | Uniform density | numerical dispersion = cold analytical dispersion | 🔲 |
| 3b | Linear density gradient | cutoff position as predicted by theory | 🔲 |
| 3c | PML parameter scans, backward-wave instability, directional Bermúdez PML | reflection below threshold over the useful n∥ range | 🔲 |
| 4 | Physics-driven adaptive meshing | same accuracy at lower cost, order preserved | 🔲 |
| 5a | Antenna geometry (FAM, PAM) | 2D then 3D geometry checked in ParaView; vacuum S-parameters = analytical mode matching | 🔲 |
| 5b | Waveguide/plasma interface, meshing at discontinuities | reflection coefficient reproduces ALOHA | 🔲 |
| 6 | Arbitrary density profiles (WEST) | scientific goal of the project | 🔲 |

Legend: 🔲 to do · 🟡 in progress · ✅ verified and closed

**Known difficulty.** The LH slow wave is *backward* in the radial direction: its phase
and group velocities point in opposite directions. Standard PMLs are proven unstable for
such waves (Bécache, Fauqueux & Joly, *J. Comput. Phys.* **188** (2003) 399). The project
will start with a standard PML, measure its failure in plasma, and only then introduce the
directional Bermúdez PML (Bermúdez *et al.*, *J. Comput. Phys.* **223** (2007) 469;
Colas *et al.*, *EPJ Web Conf.* **346** (2026) 01026).

---

## 7. Main references

- J. Hillairet *et al.*, "ALOHA: an Advanced LOwer Hybrid Antenna coupling code",
  *Nucl. Fusion* **50** (2010) 125010.
- M. Brambilla, "Slow-wave launching at the lower hybrid frequency using a phased
  waveguide array", *Nucl. Fusion* **16** (1976) 47.
- J. Schöberl, S. Zaglmayr, "High order Nédélec elements with local complete sequence
  properties", *COMPEL* **24** (2005) 374.
- E. Bécache, S. Fauqueux, P. Joly, "Stability of perfectly matched layers, group
  velocities and anisotropic waves", *J. Comput. Phys.* **188** (2003) 399.
- A. Bermúdez *et al.*, "An optimal perfectly matched layer with unbounded absorbing
  function for time-harmonic acoustic scattering problems", *J. Comput. Phys.* **223**
  (2007) 469.
- L. Colas *et al.*, "Perfectly Matched Layers for time-harmonic transverse electric wave
  propagation in cylindrical and toroidal gyrotropic media", *J. Comput. Phys.* **389**
  (2019) 94.

The full bibliography is in `CALHYSTO_IRFM_references.bib`.

---

## 8. License

MIT — see [`LICENSE`](LICENSE).
