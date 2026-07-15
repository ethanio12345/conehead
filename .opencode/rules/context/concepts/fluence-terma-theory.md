# Fluence and TERMA Calculation Theory

## Photon Fluence

Photon fluence Φ(**r**) is the total photon energy passing through a unit area at point **r**, measured in MeV/cm². In conehead, fluence is computed per voxel as a sum of three source components, each modified by the voxel's block transmission factor.

## Three-Component Source Model

Following Yang et al (2002), the photon source is modelled as three distinct components:

### 1. Point Source (Primary Fluence)

The primary photon source, located at the target position (virtual point source):

$$\Phi_{\text{pri}}(\mathbf{r}) = \frac{s_{\text{Pri}}}{d(\mathbf{r})^2} \cdot f_{\text{blocked}}(\mathbf{r})$$

- `sPri` — primary source strength (0.90924 for 6MV)
- `d(r)` — distance from source to voxel (cm), inverse-square law
- `f_blocked` — transmission factor from hit-test (0 = fully blocked, 1 = open)

### 2. Annular Source (Collimator Scatter)

Scatter from the collimator jaws, modelled as an annular ring at height `zAnn`:

$$\Phi_{\text{ann}}(\mathbf{r}) = \frac{s_{\text{Ann}}}{d_{\text{eff}}(\mathbf{r})^2} \cdot \mathbb{1}[\text{OAD} > r_{\text{inner}}] \cdot f_{\text{blocked}}(\mathbf{r})$$

- `sAnn` — annular source strength (2.887×10⁻³)
- `zAnn` — height of annular source plane (4.0 cm from target)
- `rInner` — inner radius of annular source (0.2 cm)
- `rOuter` — outer radius (1.4 cm)
- Only contributes when OAD > rInner (outside primary field)

### 3. Exponential Filter Source (Wedge/Tray Scatter)

Scatter from flattening filter and any accessories:

$$\Phi_{\text{exp}}(\mathbf{r}) = \frac{s_{\text{Exp}} \cdot e^{-k_{\text{Exp}} \cdot \text{OAD}(\mathbf{r})}}{d(\mathbf{r})^2} \cdot f_{\text{blocked}}(\mathbf{r})$$

- `sExp` — exponential source strength (8.289×10⁻³)
- `kExp` — exponential decay constant (0.4816 cm⁻¹)
- `zExp` — source height (12.5 cm)

## Off-Axis Distance (OAD)

The radial distance from the beam central axis to the voxel, projected to the isocentric plane:

$$\text{OAD}(\mathbf{r}) = \sqrt{x'^2 + y'^2}$$

where x', y' are coordinates in the beam's transverse plane. OAD determines which source components contribute and their magnitude.

## Effective Depth (d_eff)

The radiological path length from the source to each voxel through the heterogeneous medium:

$$d_{\text{eff}}(\mathbf{r}) = \int_0^{\mathbf{r}} \rho(\mathbf{r'}) \, dl$$

- Computed via DDA raytracing through the density grid
- Each voxel traversed contributes `voxel_length × density`
- Units: cm (radiological depth, not geometric depth)
- Critical for heterogeneity correction — bone has higher ρ, increasing d_eff

## Beam Softening Factor

Accounts for spectral hardening at larger off-axis distances (beam becomes "softer" away from central axis):

$$f_{\text{soften}}(\mathbf{r}) = \frac{1}{1 - \text{softRatio} \cdot \text{OAD}(\mathbf{r})}$$

- `softRatio` = 0.0025 cm⁻¹ (6MV)
- Only applied when OAD < `softLimit` (20 cm)
- Physically: off-axis photons pass through less flattening filter material, retaining lower energies

## Horn Tuning Factor

Empirical correction for energy fluence variations near field edges ("horns"):

$$f_{\text{horn}}(\mathbf{r}) = 1 + \text{hornRatio} \cdot \text{OAD}(\mathbf{r})$$

- `hornRatio` = 0.0065 %/cm (6MV)
- Compensates for the flattening filter's imperfect beam flattening

## TERMA (Total Energy Released per unit MAss)

The final TERMA combines fluence, spectral weighting, and attenuation:

$$T(\mathbf{r}) = \sum_E w_E \cdot \Phi_E(\mathbf{r}) \cdot E \cdot \frac{\mu}{\rho}(E) \cdot e^{-\mu(E) \cdot d_{\text{eff}}(\mathbf{r})} \cdot f_{\text{soften}} \cdot f_{\text{horn}}$$

- $w_E$ — spectral weight for energy bin $E$
- $\mu/\rho$ — mass attenuation coefficient from NIST data (cm²/g)
- $\mu = (\mu/\rho) \times \rho$ — linear attenuation coefficient
- Exponential term accounts for primary photon attenuation along d_eff

## References

- **Yang et al (2002)**: Three-component source model
- **Cho et al (2012)**: 6MV beam parameter values
- **NIST XCOM**: Mass attenuation coefficient data
