# Verification Dashboard

> **This file is updated by the physics-verifier agent after each module verification.**
> **Status values**: Pending, In Progress, Pass, Issues Found.
> **Do not edit manually unless fixing errors.**

| # | Module | File | Status | Critical | Warning | Info | OpenSpec Ref | Last Verified |
|---|--------|------|--------|----------|---------|------|--------------|---------------|
| 1 | Geometry | conehead/geometry.py | Issues Found | 0 | 2 | 3 | - | 2026-06-24 |
| 2 | Source | conehead/source.py | Issues Found | 1 | 2 | 1 | pipeline-verification | 2026-06-24 |
| 3 | Block | conehead/block.py | Pending | - | - | - | - | - |
| 4 | Phantom | conehead/phantom.py | Pass | 0 | 0 | 2 | - | 2026-06-24 |
| 5 | Kernel | conehead/kernel.py | Issues Found | 0 | 2 | 1 | pipeline-verification | 2026-06-24 |
| 6 | NIST | conehead/nist.py | Issues Found | 1 | 1 | 2 | - | 2026-06-24 |
| 7 | Dose Grid | conehead/dosegrid.py | Issues Found | 0 | 2 | 3 | - | 2026-06-24 |
| 8 | DDA Algorithm | conehead/dda_3d.py | Pass | 0 | 0 | 1 | - | 2026-06-24 |
| 9 | Hit Testing | conehead/conehead.py (cuda_hit_test) | Issues Found | 0 | 2 | 2 | - | 2026-06-24 |
| 10 | Effective Depth | conehead/conehead.py (cuda_d_eff) | Issues Found | 1 | 0 | 2 | pipeline-verification | 2026-06-24 |
| 11 | Off-Axis Distance | conehead/conehead.py (cuda_oad) | Pass | 0 | 0 | 3 | - | 2026-06-24 |
| 12 | Fluence | conehead/conehead.py (cuda_fluence) | Issues Found | 0 | 2 | 2 | - | 2026-06-24 |
| 13 | TERMA | conehead/conehead.py (cuda_terma) | Issues Found | 2 | 0 | 2 | pipeline-verification | 2026-06-24 |
| 14 | Dose Convolution | conehead/conehead.py (cuda_dose) | Issues Found | 1 | 3 | 2 | pipeline-verification | 2026-06-24 |

## Summary Statistics

| Status | Count |
|--------|-------|
| Pending | 1 |
| In Progress | 0 |
| Pass | 3 |
| Issues Found | 10 |

> **5 unique Critical issues found** (see CRITICAL ISSUES REGISTER below). Fixes deferred to T17.
> Row 3 (Block) is out of scope for this verification pass (not in the 14-module plan).

## CRITICAL ISSUES REGISTER (for T17)

5 unique Critical bugs confirmed across the pipeline. Each is pinned by regression tests.

| # | Issue | File:Line | Effect | Fix |
|---|-------|-----------|--------|-----|
| C1 | `np.trapz` removed in NumPy 2.0 | `varian_clinac_6MV.py:63,114` | Polyenergetic pipeline crashes on modern numpy | Replace `np.trapz` → `np.trapezoid` (2 sites) |
| C2 | Wrong coefficient for TERMA deposition | `conehead.py:375` (+ nist.py:186) | Uses μ_total/ρ instead of μ_en/ρ; **overestimates TERMA ~2×** | Expose `mu_en_water(E)` in nist.py (column 2); use it on line 375 only (keep μ_total for attenuation exp at line 374) |
| C3 | Double application of `blocked` factor | `conehead.py:377` (+360) | Transmission squared (50%→25%); under-doses penumbra | Remove `* dose_grid_blocked` from cuda_terma line 377 (already applied in fluence) |
| C4 | Missing Δθ/2π azimuthal weighting | `conehead.py:466-488` | 12 azimuthal cones each deposit full 2π kernel; **dose over-counted 12×**; energy-budget violation | Multiply deposition by `1/len(kernel_thetas)` (1/12) at line 488 |
| C5 | `d_eff` max_array_length overflow | `conehead.py:220` | `max_array_length=444` exceeded (up to 451) at gantry 143/144/216/217°; out-of-bounds cuda.local.array write | Increase to ≥471 or add bounds check before write |

**Combined impact**: C2+C4 together make absolute dose wrong by ~24× (2× TERMA × 12× over-counting). The pipeline's structural logic (geometry, DDA, fluence model, kernel normalization) is correct; the errors are localized to these 5 fixable sites.

### Notable Warnings (T17 if quick, else follow-up)
- W: 1.0 MeV kernel loads 1.5 MeV file (`conehead.py:688`) — copy-paste error
- W: KernelPoly aliasing (`kernel.py:64`) — references un-normalised param
- W: Float32 `floor()` index fragility (`conehead.py:457`) — up to 33% rel error at moderate-dose voxels
- W: Off-by-one block index (`conehead.py:64-67`) — ~1mm field shift, sub-clinical
- W: dose_grid.dose never populated by calculate() (`conehead.py:864`) — dose unreachable
- W: linspace spacing bug (`conehead.py:624,629,634,677`) — 0.201 vs 0.2 cm in density interpolator

## Layer 2 Infrastructure — T7 Complete (2026-06-24)

| Component | File | Status | Notes |
|-----------|------|--------|-------|
| Simulator pinning | tests/conftest.py | Pass | Pre-imports numba with `NUMBA_ENABLE_CUDASIM=1` before conehead's line-2 override; verified all `cuda_*` dispatchers bind to `FakeCUDAKernel` (CPU simulator). |
| Reusable fixtures | tests/conftest.py | Pass | `small_phantom` (11³ water, origin [-1.1, 0, -1.1]), `medium_phantom` (21³), `g0_source`, `g90_source`, `open_block`/`closed_block`/`wide_open_block` (skip if pydicom missing), `varian_6mv_settings`. |
| Pure-Python references | tests/cuda_reference.py | Pass | Faithful transcriptions of `cuda_dot`, `cuda_line_block_plane_collision`, `cuda_block_transmission`, `cuda_dda_3d`, `cuda_hit_test`, `cuda_d_eff`, `cuda_oad`, `cuda_fluence`, `cuda_terma`, `cuda_dose` plus whole-grid wrappers. Verified against simulator on 11³ grid for `cuda_oad` and `cuda_d_eff` to float32 tolerance. |
| Smoke test | tests/test_cuda_infrastructure.py | Pass | 20 passed, 4 skipped (pydicom not installed — pre-existing env limitation). |
| Comparison framework | tests/cuda_reference.py::assert_close | Pass | `rtol=1e-5, atol=1e-6` (float32 headroom, matches tests/test_geometry.py:14). |

### Findings flagged during transcription (for T8–T13 to investigate)

| Severity | File:Line | Quirk | Action |
|----------|-----------|-------|--------|
| Info | conehead/conehead.py:118 | `cuda_block_transmission` indexes `block_values[int(pos)-1, int(pos)-1]` — likely off-by-one from the matlab-style origin convention. Could shift the block grid by 0.1 mm relative to its declared extent. | T8 to verify visually. |
| Info | conehead/conehead.py:251-254 | `cuda_d_eff` first-voxel path uses `t[0]` directly (not `t[0]-0`). This is consistent with DDA starting at the voxel CENTRE (half-voxel from boundary) but is not documented anywhere. | T9 to confirm physical interpretation matches expectation. |
| Info | conehead/conehead.py:356 | `cuda_fluence` clamps `oad` to 2.0 cm when computing the exponential source term only: `oad = 2.0 if oad < 2.0 else oad`. This means on-axis voxels behave as if OAD=2 cm for the extra-focal source, distorting central-axis fluence. | T11 to investigate against Yang (2002). |
| Info | conehead/conehead.py:377 | `cuda_terma` multiplies by `dose_grid_blocked` AFTER fluence has already done so (line 360). Likely Critical for partial transmission (transmission gets squared); harmless if blocked is binary. | T12 to determine if `dose_grid_blocked` is ever continuous. |
| Info | conehead/conehead.py:472-488 | `cuda_dose` indexes kernel as `kernel[j, ...]` — uses only `j` (φ index), ignores `i` (θ index). This assumes the kernel is azimuthally symmetric in θ, which may not be what the production data encodes (data is `[θ, r]`). | T13b to investigate. |
| Info | conehead/conehead.py:220,390 | `max_array_length` is hardcoded to 444 (`d_eff`) and 800 (`dose`). No bounds check before writing to `voxels_traversed[n_voxels]` — if the ray traverses more than this many voxels before exiting the grid, the write is out-of-bounds. For 11³ and 21³ grids this is safe; for the 201³ production grid the longest diagonal is ~350 voxels (still safe), but no safety margin for future grid changes. | T13c to verify. |

## Severity Definitions

| Severity | Definition | Example |
|----------|-----------|---------|
| Critical | Wrong physics answer | Incorrect equation, unit mismatch producing wrong dose |
| Warning | Correct but fragile | Float32 precision risk, missing bounds check, unclear variable name |
| Info | Style/documentation | Missing docstring, naming inconsistency |

## Update Protocol

1. Change status from Pending → In Progress when starting verification
2. Update Critical/Warning/Info counts upon completion
3. Set status to Pass (zero issues) or Issues Found (any Critical/Warning)
4. Add OpenSpec reference if a proposal was created for findings
5. Record Last Verified date (YYYY-MM-DD format)
6. Update Summary Statistics table to reflect current counts

## Module 2 (Source) Findings Detail — Verified 2026-06-24

| Severity | File:Line | Issue | Suggested Fix |
|----------|-----------|-------|---------------|
| Critical | conehead/varian_clinac_6MV.py:63,114 | `np.trapz` removed in NumPy 2.0; both `weights_ali` and `weights_sheikh_bagheri` crash for array inputs. Polyenergetic pipeline cannot run on modern numpy (project pinned to 1.22.4 in requirements.txt but environment uses 2.4.6). | Replace `np.trapz` with `np.trapezoid` (or `scipy.integrate.trapezoid`). Defer to T17. |
| Warning | conehead/source.py:110-111 | `np.linalg.inv(M)` used to invert an orthonormal rotation matrix. For orthonormal matrices `inv(M) == M.T` exactly; `inv` uses LU decomposition which is slower and introduces ~1e-7 numerical error. Not a physics bug — float32 tolerance absorbs the error. | Replace `np.linalg.inv(self.transform)` with `self.transform.T` for orthonormal basis. |
| Warning | conehead/varian_clinac_6MV.py:55-56 | `np.log(E_e*(E_e-E)/E + 1.65)` evaluates `log(negative)` when E > E_e=5.76 MeV, emitting `RuntimeWarning: invalid value encountered in log`. Result is correctly clamped to 0 by the `psi[psi<0]=0` line below, but the warning is noisy and propagates to test output. | Clamp argument before log: `arg = np.maximum(E_e*(E_e-E)/E + 1.65, 1e-30)`. |
| Info | conehead/source.py:37,57,74 | Property setters re-annotate `self._position`/`self._gantry`/`self._collimator`, causing mypy `no-redef` errors. Pure type-annotation smell — no runtime effect. | Remove the redundant type annotation in setter bodies (write `self._x = value`, not `self._x: SomeType = value`). |

### Items Verified Correct (Module 2)

- **SAD = 100 cm default**: `conehead/source.py:7` — matches IEC 61217 standard linac geometry.
- **Rotation order matches spec EXACTLY**: `R_z(+gantry) @ R_y(-collimator)` applied to identity basis. Collimator (`source.py:93-98`) rotates `v_x` and `v_z` only; gantry (`source.py:101-107`) rotates all three basis vectors. Verified against analytical reconstruction for 8 angle combinations.
- **Beam direction is `v_y`**: `source.py:24,89,106` — confirmed NOT `v_z`. `v_y` always points from source position toward isocenter (verified for 7 gantry angles).
- **Source position at G=0° = [0, -100, 0]**: verified analytically and at 9 gantry angles via `phi = (90-theta) % 360` mapping (`source.py:81`).
- **IEC 61217 phi mapping**: `phi = (90-theta) % 360` is consistent with gantry 0 = source "above" isocenter in code's convention (where +y is the beam direction at G=0).
- **Basis orthonormality**: verified `R @ R.T = I` for 5 angle combinations; `det(R) = +1` (proper rotation, no reflection).
- **Transform is world-to-source-local**: verified at G=90° that `transform @ world = local` and that transform is orthonormal inverse at 20 angle combinations.
- **Collimator does not change source position**: verified at G=45°, coll=90° (collimator rotates basis around beam axis only).
- **Ali & Rogers (2012) parameters**: C_1=1.222, C_2=5.147, C_3=-1.186, E_e=5.76 MeV (`varian_clinac_6MV.py:51-54`). Spectrum integrates to 1.0 (verified), peak in 0.5-2.5 MeV range (verified), zero above E_e (verified).
- **Sheikh-Bagheri & Rogers (2002) parameters**: 24 bins from 0.25-6.0 MeV, all non-negative, integrates to ~1.0 (within 5% due to cubic interpolation + bounds).
- **Cho et al (2012) lognormal**: mu=0.30, sigma=0.8 — produces finite, non-negative spectrum with peak in clinical range.

## Module 6 (NIST) Findings Detail — Verified 2026-06-24

| Severity | File:Line | Issue | Suggested Fix |
|----------|-----------|-------|---------------|
| Critical | conehead/conehead.py:484 | **Wrong coefficient for TERMA energy deposition.** `cuda_terma` uses `mu_w[i]` (μ_total/ρ) as the final energy-deposition multiplier. The published CCC formulation (Ahnesjö 1989; Cho 2012) requires μ_en/ρ. This overestimates TERMA by 1.5–3× across the clinical energy range and propagates into ALL dose. The μ_en/ρ data IS present as column 2 in nist.py arrays but never exposed. | Add `mu_en_water`/`mu_en_Al`/`mu_en_W` accessors to nist.py (return column 2); update `cuda_terma` to accept both μ_total (for `exp(-μ×d_eff)` attenuation, line 482 — CORRECT) and μ_en (for the deposition multiplier, line 484 — currently WRONG). Defer to T17. |
| Warning | conehead/nist.py:77,132,186 | All three `interp1d` calls hard-code column index 1. Column 2 (μ_en/ρ) of every material's data array is dead data — present in the literal but unreachable. Maintenance trap. | Expose column 2 via accessor, or drop the column. |

### Items Verified Correct (Module 6)
- Data array values for μ_total/ρ match NIST XCOM for water, aluminium, tungsten at all sampled tabulated energies.
- Linear interpolation correct (verified at 1.2 MeV midpoint between 1.0/1.25 MeV tabulated points).
- **Attenuation exponential `exp(-mu_w × f_soften × d_eff)` at conehead.py:482 uses the CORRECT coefficient (μ_total/ρ)** and is dimensionally correct (density carried inside d_eff, confirmed by T9 prerequisite).
- `mu_W`/`mu_Al` usage in `varian_clinac_6MV.py:57` correctly uses μ_total/ρ for target/filter attenuation (photon-removal calc).

## Module 7 (Dose Grid) Findings Detail — Verified 2026-06-24

| Severity | File:Line | Issue | Suggested Fix |
|----------|-----------|-------|---------------|
| Warning | conehead/conehead.py:864 | **`dose_grid.dose` is never populated by `calculate()`.** The container's `dose` field is initialized to zero but never written. The actual calculated dose lives in local `dose_grid_dose` that is `copy_to_host()`'d on line 864 then discarded when `calculate()` returns. After a full calculation `self.dose_grid.dose` remains all zeros — the dose distribution is unreachable from the object graph. | At conehead.py:864, also do `self.dose_grid.dose = dose_grid_dose` before returning. Defer to T17. |
| Warning | conehead/dosegrid.py:9-10 | `origin` and `spacing` stored by reference, not copied (`dg.origin is origin` → True). Mutation of caller's array after construction silently changes grid geometry. | `self.origin = np.array(origin, dtype=np.float32, copy=True)` and same for spacing. |

### Items Verified Correct (Module 7)
- Container physically correct: shape, dtype (float32), zero-initialization (safe for `cuda.atomic.add`), geometry parity with phantom.
- Voxel→world mapping `origin + spacing*idx` consistent with CUDA convention.

## Cross-Cutting Finding (flagged by Modules 3 & 7, affects density interpolator)

| Severity | File:Line | Issue | Suggested Fix |
|----------|-----------|-------|---------------|
| Info→Warning | conehead/conehead.py:624,629,634,677-694 | **linspace spacing bug.** `np.linspace(origin, origin + size*spacing, size)` produces effective grid step 0.201 cm, not 0.2 cm (0.5% discrepancy). The CUDA kernels use literal `origin + spacing*idx` (consistent with each other) but INCONSISTENT with the host-side `RegularGridInterpolator` density resampler. ~1mm shift at distal edge. | Use `origin + np.arange(size)*spacing` or `np.linspace(origin, origin + (size-1)*spacing, size)`. Track for T9/Layer 2. |
