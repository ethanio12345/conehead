# Codebase Data Flow

Data flow through the conehead dose calculation pipeline. Each step documents inputs, outputs, and data structures.

## Pipeline Overview

```
Source → Hit Test → d_eff → OAD → Fluence → Softening → Horn → TERMA → Kernel → Dose
```

All arrays use `np.float32` dtype unless otherwise noted.

---

## Step 1: Hit Testing

**CUDA Kernel**: `cuda_hit_test` in `conehead/conehead.py`

**Inputs:**
- Dose grid geometry: shape `(nx, ny, nz)`, positions in cm
- Source position: `(sx, sy, sz)` in cm
- Block plane values: 2D transmission grid from `block.py`

**Output:**
- `dose_grid_blocked`: shape `(nx, ny, nz)`, dtype `float32`
  - Values: 0.0 (fully blocked) to 1.0 (fully open)
  - Represents transmission factor for each voxel

**Method**: DDA raytrace from source through each voxel to block plane, look up transmission from block grid.

---

## Step 2: Effective Depth

**CUDA Kernel**: `cuda_d_eff` in `conehead/conehead.py`

**Inputs:**
- Density grid: shape `(nx, ny, nz)`, values in g/cm³
- Source position: `(sx, sy, sz)` in cm
- Grid spacing: `(dx, dy, dz)` in cm

**Output:**
- `d_eff`: shape `(nx, ny, nz)`, dtype `float32`
  - Values: radiological depth in cm from source to each voxel
  - Computed as: `Σ(ρ_i × Δl_i)` along ray from source

**Method**: DDA raytrace from source to each voxel, accumulate `density × step_length` at each voxel traversed.

---

## Step 3: Off-Axis Distance

**CUDA Kernel**: `cuda_oad` in `conehead/conehead.py`

**Inputs:**
- Dose grid geometry: positions and spacing
- Source transform: rotation angles for gantry/collimator

**Output:**
- `oad`: shape `(nx, ny, nz)`, dtype `float32`
  - Values: distance from beam central axis in cm
  - Computed as: `sqrt(x'² + y'²)` in beam transverse plane

**Method**: Transform each voxel position to beam coordinate system, compute radial distance.

---

## Step 4: Fluence

**CUDA Kernel**: `cuda_fluence` in `conehead/conehead.py`

**Inputs:**
- `oad[x,y,z]` — off-axis distance (cm)
- `dose_grid_blocked[x,y,z]` — transmission factor
- Source parameters: `sPri`, `sAnn`, `zAnn`, `rInner`, `rOuter`, `sExp`, `zExp`, `kExp`
- Source position for distance calculation

**Output:**
- `fluence`: shape `(nx, ny, nz)`, dtype `float32`
  - Values: photon fluence in MeV/cm²

**Method**: Three-component sum:
```
Φ = Φ_primary + Φ_annular + Φ_exponential
  = sPri/d² + sAnn/d²·Θ(OAD>rInner) + sExp·exp(-kExp·OAD)/d²
```
Each multiplied by `blocked` transmission factor.

---

## Step 5: Softening Factor

**CPU**: NumPy operations in `conehead/conehead.py`

**Inputs:**
- `oad[x,y,z]` — off-axis distance
- `softRatio` = 0.0025 cm⁻¹
- `softLimit` = 20.0 cm

**Output:**
- `f_soften`: shape `(nx, ny, nz)`, dtype `float32`

**Method**: `f_soften = 1 / (1 - softRatio × OAD)` where OAD < softLimit, else 1.0

---

## Step 6: Horn Factor

**CPU**: NumPy operations in `conehead/conehead.py`

**Inputs:**
- `oad[x,y,z]`
- `hornRatio` = 0.0065 %/cm

**Output:**
- `f_horn`: shape `(nx, ny, nz)`, dtype `float32`

**Method**: `f_horn = 1 + hornRatio × OAD`

---

## Step 7: TERMA

**CUDA Kernel**: `cuda_terma` in `conehead/conehead.py`

**Inputs:**
- `fluence[x,y,z]`
- `d_eff[x,y,z]`
- `f_soften[x,y,z]`
- `f_horn[x,y,z]`
- Energy spectrum weights: dict of `{energy_str: weight}`
- Mass attenuation coefficients: `mu_rho[energy]` in cm²/g
- Density: `rho` in g/cm³

**Output:**
- `TERMA`: shape `(nx, ny, nz)`, dtype `float32`
  - Values: Total Energy Released per unit MAss

**Method**: For each energy bin:
```
T_E = Φ × E × (μ/ρ)_E × exp(-(μ/ρ)_E × ρ × d_eff) × f_soften × f_horn
```
Sum over all energy bins with weights.

---

## Step 8: Kernel Construction

**CPU**: NumPy operations in `conehead/kernel.py`

**Inputs:**
- Monoenergetic kernel files (`.egslst`): 12 energies × 48 angles × 24 radii
- Energy spectrum weights

**Output:**
- `kernel_cum`: shape `(n_phi, n_theta, 5996)`, dtype `float32`
  - Cumulative polyenergetic kernel at 0.1 mm resolution

**Method**: Weighted sum of monoenergetic kernels → normalise → cumsum → resample to 0.1 mm.

---

## Step 9: Dose Convolution

**CUDA Kernel**: `cuda_dose` in `conehead/conehead.py`

**Inputs:**
- `TERMA[x,y,z]`
- `kernel_cum[phi, theta, radius]`
- Grid geometry and spacing

**Output:**
- `dose`: shape `(nx, ny, nz)`, dtype `float32`
  - Values: absorbed dose (arbitrary units, requires calibration)

**Method**: For each TERMA voxel, for each cone direction, DDA raytrace, interpolate cumulative kernel, atomic add dose contribution.

---

## Array Shape Summary

| Array | Shape | dtype | Units |
|-------|-------|-------|-------|
| density | (nx, ny, nz) | float32 | g/cm³ |
| blocked | (nx, ny, nz) | float32 | 0–1 |
| d_eff | (nx, ny, nz) | float32 | cm |
| oad | (nx, ny, nz) | float32 | cm |
| fluence | (nx, ny, nz) | float32 | MeV/cm² |
| f_soften | (nx, ny, nz) | float32 | dimensionless |
| f_horn | (nx, ny, nz) | float32 | dimensionless |
| TERMA | (nx, ny, nz) | float32 | MeV/g |
| kernel_cum | (24, 2, 5996) | float32 | dimensionless |
| dose | (nx, ny, nz) | float32 | Gy (after calibration) |
