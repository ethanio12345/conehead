# Dose Calculation Code Patterns

Code patterns used throughout the conehead dose calculation engine.

## 1. CUDA Kernel Pattern

All GPU-accelerated computation follows this template:

```python
from numba import cuda
import numpy as np

@cuda.jit
def cuda_compute(input_array, output_array, param1, param2):
    """Standard CUDA kernel with 3D grid indexing."""
    x, y, z = cuda.grid(3)

    # Bounds checking — always required
    if x >= output_array.shape[0]:
        return
    if y >= output_array.shape[1]:
        return
    if z >= output_array.shape[2]:
        return

    # Computation
    value = input_array[x, y, z] * param1 + param2
    output_array[x, y, z] = np.float32(value)

# Launch configuration
threads_per_block = (8, 8, 8)
blocks_per_grid = (
    (nx + threads_per_block[0] - 1) // threads_per_block[0],
    (ny + threads_per_block[1] - 1) // threads_per_block[1],
    (nz + threads_per_block[2] - 1) // threads_per_block[2],
)
cuda_compute[blocks_per_grid, threads_per_block](input, output, p1, p2)
```

Key points:
- Use `cuda.grid(3)` for 3D voxel indexing
- Always bounds-check all three dimensions
- Pass parameters as `np.float32()` constants
- Use separate input/output arrays (no in-place modification)

## 2. DDA Raytracing Pattern

Digital Differential Analyzer for stepping through voxels along a ray:

```python
# Direction components
dx, dy, dz = direction_vector
stepX = 1 if dx > 0 else -1
stepY = 1 if dy > 0 else -1
stepZ = 1 if dz > 0 else -1

# Initial t-values (parameterized ray position)
tMaxX = (voxel_boundary_x - origin_x) / dx
tMaxY = (voxel_boundary_y - origin_y) / dy
tMaxZ = (voxel_boundary_z - origin_z) / dz

# t-delta (voxel crossing increment)
tDeltaX = grid_spacing_x / abs(dx)
tDeltaY = grid_spacing_y / abs(dy)
tDeltaZ = grid_spacing_z / abs(dz)

# March through voxels
for step in range(max_steps):
    # Choose smallest t — next voxel boundary crossing
    if tMaxX < tMaxY and tMaxX < tMaxZ:
        tMaxX += tDeltaX
        ix += stepX
    elif tMaxY < tMaxZ:
        tMaxY += tDeltaY
        iy += stepY
    else:
        tMaxZ += tDeltaZ
        iz += stepZ

    # Accumulate value at voxel (ix, iy, iz)
    # ... process voxel ...
```

Key points:
- Fixed-size local arrays for recording traversed voxels (444 or 800 elements)
- t-values track ray parameter, not geometric distance
- Must handle zero direction components (ray parallel to axis)

## 3. Kernel Convolution Pattern

Collapsed-cone superposition over TERMA grid:

```python
@cuda.jit
def cuda_dose(terma, dose, kernel_cum, grid_params):
    x, y, z = cuda.grid(3)
    if x >= terma.shape[0]: return
    if y >= terma.shape[1]: return
    if z >= terma.shape[2]: return

    if terma[x, y, z] == 0.0:
        return  # Skip zero-TERMA voxels

    t = terma[x, y, z]

    for theta_idx in range(n_theta):
        for phi_idx in range(n_phi):
            # Get cone direction vector
            dx = cos_phi[phi_idx] * cos_theta[theta_idx]
            dy = cos_phi[phi_idx] * sin_theta[theta_idx]
            dz = sin_phi[phi_idx]

            # Normalise direction
            norm = sqrt(dx*dx + dy*dy + dz*dz)
            dx /= norm; dy /= norm; dz /= norm

            # DDA raytrace from (x,y,z) along direction
            # ... accumulate dose via kernel interpolation ...

            # Atomic add to dose grid (thread-safe)
            cuda.atomic.add(dose, (ix, iy, iz), contribution)
```

## 4. Fluence Calculation Pattern

Three-component source model summation:

```python
@cuda.jit
def cuda_fluence(oad, blocked, fluence, sPri, sAnn, zAnn, ...):
    x, y, z = cuda.grid(3)
    # ... bounds checking ...

    dist = distance_to_source(x, y, z)
    o = oad[x, y, z]
    b = blocked[x, y, z]

    # Primary component
    phi = sPri / (dist * dist)

    # Annular component (only if OAD > rInner)
    if o > rInner:
        phi += sAnn / (dist * dist)

    # Exponential filter component
    phi += sExp * math.exp(-kExp * o) / (dist * dist)

    fluence[x, y, z] = phi * b
```

## 5. Settings Dict Pattern

All configurable parameters passed as a dictionary:

```python
settings = {
    "sPri": np.float32(0.90924),
    "sAnn": np.float32(2.887e-3),
    "zAnn": np.float32(4.0),
    "rInner": np.float32(0.2),
    "rOuter": np.float32(1.4),
    "zExp": np.float32(12.5),
    "sExp": np.float32(8.289e-3),
    "kExp": np.float32(0.4816),
    "softRatio": np.float32(0.0025),
    "softLimit": np.float32(20.0),
    "hornRatio": np.float32(0.0065),
    "energy_weights": {"0.5": 0.08196, "1.0": 0.12385, ...},
    "fluenceResampling": np.int32(10),
}
```

Key points:
- All float values stored as `np.float32`
- Descriptive key names matching parameter symbols
- Energy weights use string keys to match kernel dict
