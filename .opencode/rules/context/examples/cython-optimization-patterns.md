# Cython Optimization Patterns

Cython patterns relevant to conehead's CPU-based computation paths.

## 1. Typed Memoryviews

Typed memoryviews provide C-level array access without Python overhead:

```python
# cython: boundscheck=False, wraparound=False, cdivision=True
import numpy as np
cimport numpy as np

def compute_dose(double[:, :, :] density,    # 3D typed memoryview
                 double[:, :, :] terma,
                 double[:] kernel_cum,
                 double dx, double dy, double dz):
    cdef int nx = density.shape[0]
    cdef int ny = density.shape[1]
    cdef int nz = density.shape[2]
    cdef int ix, iy, iz

    for ix in range(nx):
        for iy in range(ny):
            for iz in range(nz):
                # Direct C-level indexing — no Python overhead
                terma[ix, iy, iz] = density[ix, iy, iz] * ...
```

Key points:
- `double[:, :, :]` = 3D C-contiguous memoryview
- `boundscheck=False` — disables runtime bounds checking (production only)
- `wraparound=False` — disables negative indexing
- `cdivision=True` — C-style division (no ZeroDivisionError)

## 2. cdef/cpdef Functions

```python
cdef double _attenuation(double mu, double d_eff) nogil:
    """C-level function — no Python overhead, cannot be called from Python."""
    return exp(-mu * d_eff)

cpdef double attenuation(double mu, double d_eff):
    """Callable from both Cython and Python."""
    return _attenuation(mu, d_eff)
```

- `cdef` — C-level only, fastest, no Python interaction
- `cpdef` — dual C/Python callable, slight overhead from Python wrapper
- `def` — standard Python function, full overhead

## 3. OpenMP Parallelization

```python
from cython.parallel import prange

def parallel_dose(double[:, :, :] terma, double[:, :, :] dose):
    cdef int ix, iy, iz
    cdef int nx = terma.shape[0]
    cdef int ny = terma.shape[1]
    cdef int nz = terma.shape[2]

    with nogil:
        for ix in prange(nx, schedule='guided'):
            for iy in range(ny):
                for iz in range(nz):
                    dose[ix, iy, iz] = _compute(terma[ix, iy, iz])

cdef double _compute(double t) nogil:
    # Must be nogil for use inside prange
    return t * 0.5
```

Key points:
- `prange` distributes outer loop across CPU cores
- All code inside `nogil` block must be C-level (no Python objects)
- `schedule='guided'` — good default for uneven workloads
- Thread safety: avoid writing to shared locations without synchronisation

## 4. Compilation Setup

```python
# setup.py
from setuptools import setup, Extension
from Cython.Build import cythonize
import numpy as np

extensions = [
    Extension(
        "conehead.dda_3d_c",
        sources=["conehead/dda_3d_c.pyx"],
        include_dirs=[np.get_include()],
        extra_compile_args=["-O0"],  # Debug — change to -O3 for production
    ),
]

setup(
    ext_modules=cythonize(extensions, language_level="3"),
)
```

Build command:
```bash
uv run python setup.py build_ext --inplace
```

## 5. Existing Cython Files

### dda_3d_c.pyx

CPU-based DDA raytracing with typed memoryviews:
- Uses `nogil` for inner loop performance
- Typed memoryviews for grid data (density, positions)
- Fallback/alternative to CUDA DDA implementation

### convolve_c.pyx (Currently Unused)

Wrapper for C-level convolution:
- Placeholder for CPU-based collapsed-cone convolution
- May be activated when CUDA is unavailable

## 6. Cython vs CUDA Decision Guide

| Factor | Use Cython | Use CUDA |
|--------|-----------|----------|
| Data size | Small grids (<64³) | Large grids (≥128³) |
| GPU available | No | Yes |
| Algorithm | Sequential or modestly parallel | Highly parallel (per-voxel independent) |
| Development speed | Faster compile/debug cycle | Slower (GPU debugging is hard) |
| Precision | float64 easy | float32 default, float64 possible |
| Deployment | No GPU dependency | Requires NVIDIA GPU |

## 7. Compilation Flags

| Flag | Use Case |
|------|----------|
| `-O0` | Debug — no optimisation, clear stack traces |
| `-O2` | Production — good optimisation, reasonable compile time |
| `-O3` | Maximum optimisation — best runtime, longer compile |
| `-ffast-math` | Unsafe math optimisations — slightly faster but may change results |

Current conehead uses `-O0` for development. Should switch to `-O3` for production runs.

## 8. Memory Layout

Cython typed memoryviews assume **C-contiguous** (row-major) layout:

```python
# Correct — C-contiguous
arr = np.zeros((nx, ny, nz), dtype=np.float64, order='C')

# Wrong — Fortran-contiguous
arr_f = np.zeros((nx, ny, nz), dtype=np.float64, order='F')
```

For cache efficiency in nested loops, iterate in memory order:
```python
for ix in range(nx):        # Outer — slowest varying
    for iy in range(ny):    # Middle
        for iz in range(nz): # Inner — fastest varying (contiguous in memory)
```
