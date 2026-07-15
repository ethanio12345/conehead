# NIST Photon Interaction Data Reference

Quick reference for NIST photon interaction data used in conehead's TERMA calculation.

## Mass Attenuation Coefficients for Water

The function `mu_water(energy)` in `conehead/nist.py` returns the mass attenuation coefficient μ/ρ for water at a given photon energy.

### Data Source

- **Database**: NIST XCOM Photon Cross Sections Database
- **URL**: https://physics.nist.gov/PhysRefData/Xcom/html/xcom1.html
- **Medium**: Water (H₂O)
- **Interaction types included**: Photoelectric, Compton (incoherent), Pair production

### Key Values

| Energy (MeV) | μ/ρ (cm²/g) | Dominant Interaction |
|--------------|-------------|---------------------|
| 0.01 | 5.329 | Photoelectric |
| 0.1 | 0.171 | Compton |
| 0.5 | 0.0966 | Compton |
| 1.0 | 0.0707 | Compton |
| 1.5 | 0.0567 | Compton |
| 2.0 | 0.0493 | Compton |
| 3.0 | 0.0396 | Compton/Pair |
| 4.0 | 0.0340 | Compton/Pair |
| 5.0 | 0.0305 | Compton/Pair |
| 6.0 | 0.0275 | Compton/Pair |
| 7.0 | 0.0254 | Pair |

### Usage in TERMA Calculation

The mass attenuation coefficient converts photon fluence to TERMA via:

```
TERMA = Φ × E × (μ/ρ) × exp(-μ × d_eff)
```

Where:
- `μ = (μ/ρ) × ρ` — linear attenuation coefficient (cm⁻¹)
- `ρ` — medium density (g/cm³), water = 1.0
- `d_eff` — effective radiological depth (cm)

### Interpolation

- **Method**: Linear interpolation between tabulated values
- **Energy range**: 0.01 to 7.0 MeV for 6MV beam
- **Energy units**: MeV throughout (no keV conversion needed)

### Implementation Notes

```python
# nist.py provides tabulated data as arrays
energies = [0.01, 0.1, 0.5, 1.0, ...]  # MeV
mu_rho_values = [5.329, 0.171, 0.0966, 0.0707, ...]  # cm²/g

# Linear interpolation for arbitrary energy
mu_rho = np.interp(energy, energies, mu_rho_values)
```

### Energy Spectrum Integration

For the polyenergetic 6MV beam, TERMA is computed per energy bin:

```python
for energy_str, weight in energy_weights.items():
    energy = float(energy_str)
    mu_rho = nist.mu_water(energy)
    mu = mu_rho * density
    terma += weight * fluence * energy * mu_rho * np.exp(-mu * d_eff)
```

### Critical Consistency Check

The energy values used as dictionary keys in `energy_weights` (e.g., "0.5", "1.0") **must exactly match** the energy values available in the NIST lookup table. The code converts string keys to float for lookup — ensure no floating-point comparison issues arise from string-to-float conversion.

### Density Units

- Water density: 1.0 g/cm³
- Bone (cortical): ~1.85 g/cm³
- Lung: ~0.26 g/cm³
- Air: ~0.0012 g/cm³

These are used with μ/ρ to compute the linear attenuation coefficient μ for heterogeneous media.

## References

- **NIST XCOM**: National Institute of Standards and Technology, Photon Cross Sections Database
- **Hubbell & Seltzer (2004)**: Tables of X-Ray Mass Attenuation Coefficients
