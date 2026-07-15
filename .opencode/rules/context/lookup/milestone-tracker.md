# Milestone Tracker

> **Living tracking document for conehead project milestones.**
> **Updated after each feature completion or verification.**

| # | Milestone | Status | Priority | Dependencies | OpenSpec Ref | Notes |
|---|-----------|--------|----------|-------------|--------------|-------|
| 1 | Linac Geometry | ✅ Complete | - | - | - | Verified working |
| 2 | Water Phantom | ✅ Complete | - | - | - | SimplePhantom class |
| 3 | Point Source | ✅ Complete | - | - | - | Primary fluence |
| 4 | Annular Source | ✅ Complete | - | - | - | Collimator scatter |
| 5 | Exponential Filter Source | ✅ Complete | - | - | - | Filter scatter |
| 6 | Voxel Blocking | ✅ Complete | - | - | - | Jaws + MLC |
| 7 | Fluence Calculation | ✅ Complete | - | - | - | Three-component model |
| 8 | TERMA Calculation | ✅ Complete | - | - | - | Polyenergetic |
| 9 | Kernel (EDKnrc) | ✅ Complete | - | - | - | Mono + Poly |
| 10 | Kernel Convolution | ✅ Complete | - | - | - | Collapsed cone |
| 11 | Horn Tuning | ✅ Complete | - | - | - | Empirical factor |
| 12 | Square Fields | ✅ Complete | - | - | - | Block.set_square() |
| 13 | 3DCRT DICOM | ✅ Complete | - | - | - | STATIC beams |
| 14 | Kernel Tilting | ⬜ Remaining | High | Convolution | - | Account for beam divergence |
| 15 | CT Image Import | ⬜ Remaining | High | - | - | Replace water phantom |
| 16 | IMRT/VMAT DICOM | ⬜ Remaining | High | DICOM specialist | - | DYNAMIC beams + control points |
| 17 | Density Overrides | ⬜ Remaining | Medium | CT Import | - | Structure-based |
| 18 | Out-of-Field Dosimetry | ⬜ Remaining | Low | All above | - | Optional optimization |

## Summary

| Status | Count |
|--------|-------|
| ✅ Complete | 13 |
| ⬜ Remaining | 5 |
| 🔧 In Progress | 0 |

## Priority Definitions

| Priority | Meaning |
|----------|---------|
| High | Required for clinical accuracy or next major feature |
| Medium | Improves accuracy but not blocking |
| Low | Nice-to-have, optimization or edge case handling |

## Dependency Notes

- **Kernel Tilting (#14)**: Depends on convolution being verified first. Affects dose accuracy for large fields and off-axis points.
- **CT Image Import (#15)**: Needed before heterogeneity corrections have clinical value. Currently using water phantom (uniform density).
- **IMRT/VMAT DICOM (#16)**: Requires DYNAMIC beam support in DICOM parser. Currently only STATIC beams handled.
- **Density Overrides (#17)**: Requires CT import for base image. Overrides applied as structure-based ROI masks.
- **Out-of-Field (#18)**: All previous milestones should be complete. Focus is on peripheral dose optimization.

## Completion Log

| Date | Milestone | Notes |
|------|-----------|-------|
| - | 1-13 | Core pipeline completed in initial development |
