# Varian Clinac 6MV Linear Accelerator Beam Model

## Source Model Parameters (Cho et al 2012)

The photon source is modelled as three components following Yang et al (2002):

### Point Source (Primary)

| Parameter | Value | Description |
|-----------|-------|-------------|
| `sPri` | 0.90924 | Primary source strength (fraction) |

### Annular Source (Collimator Scatter)

| Parameter | Value | Description |
|-----------|-------|-------------|
| `sAnn` | 2.887×10⁻³ | Annular source strength |
| `zAnn` | 4.0 cm | Height above target |
| `rInner` | 0.2 cm | Inner ring radius |
| `rOuter` | 1.4 cm | Outer ring radius |

### Exponential Filter Source

| Parameter | Value | Description |
|-----------|-------|-------------|
| `sExp` | 8.289×10⁻³ | Exponential source strength |
| `zExp` | 12.5 cm | Height above target |
| `kExp` | 0.4816 cm⁻¹ | Exponential decay constant |

## Beam Softening

| Parameter | Value | Description |
|-----------|-------|-------------|
| `softRatio` | 0.0025 cm⁻¹ | Softening rate per OAD |
| `softLimit` | 20.0 cm | Maximum OAD for softening |

Formula: `f_soften = 1 / (1 - softRatio × OAD)` for OAD < softLimit

## Horn Tuning

| Parameter | Value | Description |
|-----------|-------|-------------|
| `hornRatio` | 0.0065 %/cm | Horn correction rate |

Formula: `f_horn = 1 + hornRatio × OAD`

## 6MV Energy Spectrum Weights

The polyenergetic beam is discretized into 12 energy bins from 0.5 to 6.0 MeV:

| Energy (MeV) | Weight |
|--------------|--------|
| 0.5 | 0.08196 |
| 1.0 | 0.12385 |
| 1.5 | 0.10605 |
| 2.0 | 0.08307 |
| 2.5 | 0.05881 |
| 3.0 | 0.03911 |
| 3.5 | 0.02131 |
| 4.0 | 0.02426 |
| 4.5 | 0.00000 |
| 5.0 | 0.00881 |
| 5.5 | 0.00000 |
| 6.0 | 0.00498 |

Note: Weights are normalised to sum to ~1.0. The 4.5 and 5.5 MeV bins have zero weight.

## Geometry Constants

| Parameter | Value | Description |
|-----------|-------|-------------|
| SAD | 100.0 cm | Source-to-Axis Distance |
| Source position (gantry 0°) | (0, -100, 0) | In IEC 61217 coordinates |

## MLC Configuration

- **Type**: Millennium-style multi-leaf collimator
- **Leaf boundaries**: Read from DICOM RT Plan (BeamLimitingDevicePositionSequence)
- **Leaf count**: Typically 60 pairs (120 leaves)
- **Leaf widths**: Vary — inner leaves 5 mm, outer leaves 10 mm at isocentre
- **Leaf transmission**: Hardcoded edge values (0.20, 0.50, 0.75 for first 3 rows)

## Coordinate System (IEC 61217)

| Axis | Positive Direction |
|------|--------------------|
| X | Patient left |
| Y | Posterior to anterior (gantry 0°: source at Y = -SAD) |
| Z | Inferior to superior |

**Gantry rotation**: About Y-axis. At gantry angle θ, the source position is:
- X = SAD × sin(θ)
- Y = -SAD × cos(θ)
- Z = 0

**Collimator rotation**: About Z-axis (beam central axis).

**Note on angle convention**: IEC 61217 gantry angles differ from mathematical convention:
- phi = (90 - theta) % 360 for coordinate transformations

## Key Physics Notes

1. **Inverse square law**: All source components use 1/d² correction for fluence falloff
2. **Attenuation**: Primary beam attenuated by exp(-μ × d_eff) through heterogeneous medium
3. **Spectral hardening**: Higher-energy photons penetrate deeper (lower μ/ρ)
4. **Field size dependence**: Scatter contribution from annular source increases with field size
5. **Effective SSD**: Not used in conehead — d_eff handles heterogeneity directly

## References

- **Cho et al (2012)**: All parameter values above
- **Yang et al (2002)**: Three-component source model theory
- **IEC 61217**: Coordinate system standard for radiotherapy equipment
