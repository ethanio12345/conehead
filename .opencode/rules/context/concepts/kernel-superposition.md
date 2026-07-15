# Energy Deposition Kernels and Superposition

## Monoenergetic Kernels

Energy deposition kernels are pre-computed via Monte Carlo simulation (EDKnrc) for discrete photon energies from 0.5 to 6.0 MeV in 0.5 MeV steps (12 energies). Each kernel describes the spatial distribution of energy deposited by a single photon interaction at the origin.

### Kernel Structure

- **Angular discretization**: 48 cones — 24 polar angles (phi) at 3.75° spacing × 2 azimuthal angles (theta = 0°, 180°)
- **Radial discretization**: 24 spherical shells from 0 to 60 cm radius
- **File format**: `.egslst` files with columns: angle (degrees), radius (cm), differential value

### Physical Meaning

Each entry `k(r, θ)` gives the **differential** fraction of incident energy deposited in the volume element at distance `r` in direction `θ` from the interaction point. Units: MeV⁻¹ cm⁻² (energy per unit solid angle per unit radius).

## Kernel Normalisation

Kernel values are normalised so the integral over all directions and radii equals 1.0:

$$\sum_{\theta} \sum_{r} k(r, \theta) \cdot \Delta r = 1.0$$

This represents energy conservation — all energy released at the interaction site must be deposited somewhere in the medium.

In code: `kernel_diff /= kernel_diff.sum()` after reading raw Monte Carlo output.

## Cumulative Distribution

For efficient interpolation during dose calculation, the differential kernel is converted to a cumulative distribution along the radius axis:

```python
kernel_cum = np.cumsum(kernel_diff, axis=1)  # cumsum along radius
```

The cumulative kernel is then **resampled** to 0.1 mm resolution (5996 points from 0 to ~60 cm) using linear interpolation. This provides a dense lookup table for determining dose deposited between any two radii along a ray.

## Polyenergetic Kernel

The polyenergetic kernel is a weighted sum of monoenergetic kernels:

$$k_{\text{poly}}(r, \theta) = \sum_E w_E \cdot k_E(r, \theta)$$

where $w_E$ are the spectral weights from the beam energy spectrum. This combines the energy dependence into a single kernel, avoiding separate convolutions per energy bin.

In code, this happens in the `KernelPoly` constructor:
```python
for energy_str, weight in energy_weights.items():
    kernel_mono = self.kernel_dict[energy_str]
    kernel_poly += weight * kernel_mono
```

## Kernel Tilting (Not Yet Implemented)

In a divergent beam, the kernel coordinate system at each TERMA point should be **tilted** so that the cone directions point away from the photon source, not along the fixed z-axis.

**Why it matters**: Without tilting, scattered energy is transported along the grid z-axis rather than along the actual beam direction. For large fields and off-axis points, this introduces dose errors, particularly at depth.

**Implementation approach**: Rotate each kernel direction vector by the angle between the source-to-TERMA-point vector and the z-axis. This requires per-voxel direction transformation.

## The Convolution Step

For each TERMA voxel, the collapsed-cone superposition proceeds as:

1. **Select voxel** where TERMA > 0 (optimisation: skip zero-TERMA voxels)
2. **For each cone direction** (theta, phi):
   - Compute direction vector from (theta, phi) angles
   - **If tilting**: rotate direction toward source
   - **Raytrace** via DDA from the TERMA voxel along the direction
   - At each voxel boundary, **interpolate cumulative kernel** at that radius
   - **Dose contribution** = TERMA × (cumulative kernel at exit − cumulative kernel at entry)
   - **Atomic add** to dose grid (GPU thread safety)
3. **Repeat** for all TERMA voxels and all directions

### Array Layout

- `kernel_diff`: shape (n_phi, n_theta, n_radius), dtype float32
- `kernel_cum`: shape (n_phi, n_theta, n_resampled), dtype float32
- Cumulative kernel stored at 0.1 mm resolution (5996 points)

## Double Normalisation Caveat

The `KernelPoly` constructor normalises the summed kernel, but individual monoenergetic kernels may already be normalised. This results in **double normalisation** — the polyenergetic kernel is normalised twice. Currently this is harmless (both normalisations preserve the integral = 1.0 property) but should be documented clearly.

## References

- **Ahnesjö (1989)**: Collapsed cone convolution method
- **Mackie et al (1988)**: Energy deposition kernels for photon dose calculation
- **Cho et al (2012)**: Kernel parameters for Varian 6MV
