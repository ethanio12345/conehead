# Collapsed-Cone Convolution Superposition (CCCS)

## The Physical Problem

In megavoltage photon radiotherapy, dose deposition is not purely local. A photon interacting at point **r** deposits energy both at **r** (primary) and at surrounding points via scattered photons and secondary electrons. Accurate dose calculation requires accounting for this non-local energy transport throughout the entire patient volume.

The total absorbed dose at any point is the superposition of contributions from **all** interaction sites:

$$D(\mathbf{r}) = \int T(\mathbf{r'}) \cdot k(\mathbf{r} - \mathbf{r'}) \, d^3\mathbf{r'}$$

where $T(\mathbf{r'})$ is TERMA (Total Energy Released per unit MAss) at the interaction site, and $k(\mathbf{r} - \mathbf{r'})$ is the energy deposition kernel describing how energy spreads from $\mathbf{r'}$ to $\mathbf{r}$.

## Key Insight — Angular Discretization

Direct evaluation of the 3D convolution is O(N^2) in the number of voxels — computationally intractable for clinical grid sizes (e.g., 256^3 = 16.7M voxels → 280 trillion pairs).

**Ahnesjö (1989)** showed that the kernel can be discretized into a finite number of solid-angle cones. Instead of convolving the full 3D kernel, energy is transported **along rays** through each cone direction. This collapses the 3D problem into a set of 1D ray-trace operations.

## The Algorithm

1. **Compute TERMA** at every voxel — the energy released per unit mass from primary photon interactions.
2. **Discretize directions** — divide the unit sphere into a fixed number of cones. Conehead uses 48 cones: 24 polar angles (phi) at 3.75° spacing × 2 azimuthal angles (theta = 0°, 180°).
3. **For each TERMA voxel**, trace a ray through each cone direction using the DDA (Digital Differential Analyzer) algorithm.
4. **Interpolate the kernel** — at each voxel boundary along the ray, look up the cumulative energy deposition from the pre-computed kernel. The difference between successive boundary values gives the dose deposited in that voxel.
5. **Accumulate dose** — add each contribution to the dose grid via atomic operations (GPU) or critical sections (CPU).

## The Energy Deposition Kernel

The kernel $k(r, \theta)$ represents the fraction of energy deposited at distance $r$ along direction $\theta$ from the interaction site. It is:

- **Pre-computed** via Monte Carlo simulation (EDKnrc) for specific photon energies (0.5–6.0 MeV)
- **Stored** as differential values: angle × radius tables
- **Normalised** so that the sum over all directions and radii equals 1.0 (energy conservation)
- **Converted** to cumulative form for efficient interpolation during raytracing

## Why "Collapsed Cone"?

The 3D kernel is **collapsed** into a finite set of radial rays. Each cone collects all energy within its solid angle and transports it along a single ray. This reduces computational complexity from:

- **Full convolution**: O(N^2) voxel pairs
- **Collapsed cone**: O(N × n_cones × ray_length) ≈ O(N × 48 × ~200)

For a 256^3 grid: full = ~2.8×10^14 pairs → collapsed = ~1.6×10^8 operations. A speedup of ~1.7 million.

## Key Equations

**Discrete dose summation:**

$$D(\mathbf{r}) = \sum_i T(\mathbf{r}_i) \cdot k_{\text{collapsed}}(\mathbf{r} - \mathbf{r}_i)$$

**TERMA (Total Energy Released per unit MAss):**

$$T(\mathbf{r}) = \Phi(\mathbf{r}) \cdot E \cdot \frac{\mu}{\rho}(E) \cdot e^{-\mu(E) \cdot d_{\text{eff}}(\mathbf{r})}$$

**Polyenergetic dose (energy spectrum summation):**

$$D(\mathbf{r}) = \sum_E w_E \sum_i T_E(\mathbf{r}_i) \cdot k_E(\mathbf{r} - \mathbf{r}_i)$$

where $w_E$ are spectral weights and the subscript $E$ denotes energy-dependent quantities.

## References

- **Ahnesjö (1989)**: "Collapsed cone convolution of radiant energy for photon dose calculation in heterogeneous media" — the foundational CCCS paper.
- **Cho et al (2012)**: "Reference dose distribution and Monte Carlo modelling of the Varian Clinac 6MV beam" — source model and beam parameters used in conehead.
- **Yang et al (2002)**: "Modelling of extra-focal radiation for fast dose calculation in photon beams" — three-component source model.
