# Common Physics Pitfalls in Dose Calculation

Known pitfalls and error patterns to watch for when working with the conehead codebase.

## 1. Unit Mismatches

The most common source of physics errors. Conehead uses three length scales:

| Context | Unit | Conversion |
|---------|------|------------|
| Internal geometry | cm | Base unit |
| DICOM positions | mm | ÷10 to get cm |
| Block plane positions | tenths-of-mm (0.1 mm) | ÷100 to get cm |

**Angular units** are another trap:
- Kernel angles stored in **degrees** (phi = 0° to 90°, theta = 0° or 180°)
- `math.sin()`, `math.cos()` expect **radians**
- Must convert: `math.sin(phi_deg * pi / 180.0)`
- CUDA `sin()` and `cos()` also expect radians

## 2. Float32 Precision Loss

CUDA kernels operate in float32 by default. Areas vulnerable to precision loss:

- **Cumulative kernel interpolation**: Many small increments summed over ~6000 resampled points. Cumulative error can be significant.
- **DDA t-value accumulation**: Over long raytraces (diagonal of 256³ grid ≈ 444 voxels), small errors compound.
- **Exponential attenuation**: `exp(-μ × d_eff)` for large d_eff (deep in phantom) can underflow to zero in float32.
- **Atomic add ordering**: `cuda.atomic.add` for float32 has no guaranteed operation order. Accumulated dose may vary slightly (±1 ULP) between runs.

**Mitigation**: Document where float64 would help but don't change without benchmarking — float64 on GPU is significantly slower.

## 3. Inverse Square Singularity

The inverse-square law (1/d²) diverges as distance → 0:

```python
# Dangerous — d can be near-zero for voxels close to source
fluence = sPri / (dist * dist)

# Safe — clamp minimum distance
min_dist = np.float32(0.1)  # 1 mm minimum
fluence = sPri / max(min_dist, dist * dist)
```

In practice, dose grid voxels should not be at the source position, but defensive coding prevents NaN propagation.

## 4. OAD Near Zero

The exponential source component has 1/r dependence that diverges:

```python
# Dangerous — OAD can be zero on central axis
phi_exp = sExp * exp(-kExp * oad) / oad

# Current mitigation — clamp to minimum
oad_clamped = max(np.float32(2.0), oad)
```

The clamp value of 2.0 cm means central-axis voxels use OAD=2cm for the exponential term. This is a known approximation.

## 5. DDA Array Bounds

DDA raytracing uses fixed-size local arrays in CUDA kernels:

```python
voxels_x = cuda.local.array(444, dtype=np.int32)  # Fixed size
voxels_y = cuda.local.array(444, dtype=np.int32)
voxels_z = cuda.local.array(444, dtype=np.int32)
t_values = cuda.local.array(800, dtype=np.float32)
```

These sizes must be ≥ the longest possible diagonal through the grid. For a 256³ grid with 0.25 cm spacing, the diagonal is ~444 voxels. **If grid size increases, these arrays must be enlarged** — otherwise silent buffer overflow.

## 6. Kernel Double Normalisation

The `KernelPoly` constructor performs:

```python
# Step 1: Monoenergetic kernels are already normalised (sum = 1.0)
# Step 2: Weighted sum preserves normalisation (weights sum to ~1.0)
kernel_poly = sum(weight * kernel_mono for energy, weight in weights)
# Step 3: Constructor normalises again
kernel_poly /= kernel_poly.sum()  # Redundant but harmless
```

This is not a bug (double normalisation of a value already equal to 1.0 gives 1.0), but it can mask errors in spectral weights. If weights don't sum to 1.0, the second normalisation silently fixes it.

## 7. Coordinate System Convention

IEC 61217 and mathematical conventions differ for gantry angle:

```python
# Mathematical: angle measured from positive x-axis, counter-clockwise
# IEC 61217: angle measured from vertical (negative y), clockwise

# Conversion: phi = (90 - theta) % 360
# Example: gantry 0° → source at (0, -SAD, 0)
#          gantry 90° → source at (SAD, 0, 0)
```

Getting this wrong rotates the beam 90° from the intended position.

## 8. Block Indexing (Off-by-One)

Block transmission lookup uses 1-based indexing:

```python
# Block plane indices are computed then decremented
block_idx = compute_index(position)
value = block_values[block_idx - 1]  # 1-based → 0-based conversion
```

If the index computation doesn't account for the 1-based convention, off-by-one errors cause systematic shifts in the blocked region.

## 9. Atomic Add Non-Determinism

```python
# Multiple threads may write to the same dose voxel simultaneously
cuda.atomic.add(dose, (ix, iy, iz), contribution)
```

Float32 atomic addition is not associative: `(a + b) + c ≠ a + (b + c)` in floating-point. Results may differ slightly between runs with different thread scheduling. Differences should be < 1 ULP per contributing thread.

## 10. Energy Weight Key Matching

```python
# Settings dict uses string keys
energy_weights = {"0.5": 0.08196, "1.0": 0.12385, ...}

# Kernel dict uses the same string keys
kernel_dict = {"0.5": kernel_05, "1.0": kernel_10, ...}

# Must match exactly — "1.00" ≠ "1.0"
```

A mismatch (e.g., "1.00" vs "1.0") silently skips that energy bin, producing wrong TERMA values with no error message.
