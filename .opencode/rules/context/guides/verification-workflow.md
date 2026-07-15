# Verification Workflow Guide

Step-by-step guide for verifying a conehead module's physics correctness.

## Step 1: Read the Target Source File

Read the entire source file. Do not skim. Understand every function, every variable, every equation. Map each function to its purpose in the calculation pipeline.

## Step 2: Identify All Physics Equations

For each function, identify:
- What physical quantity it computes
- The reference paper equation it implements
- Whether the equation is exact, approximate, or empirical

Create a mapping table:
```
Function → Paper Equation → Variables → Approximations
```

## Step 3: Check Unit Consistency

Conehead uses mixed units across the pipeline. Verify each computation:

| Quantity | Expected Unit | Where Used |
|----------|---------------|------------|
| Distance | cm | Grid positions, OAD, d_eff |
| Energy | MeV | Spectrum, TERMA |
| Density | g/cm³ | Density grid, phantom |
| Block positions | tenths-of-mm (0.1 mm) | Block plane indices |
| Kernel angles | degrees | Phi, theta arrays |
| Trig functions | radians | Must convert from degrees |
| μ/ρ | cm²/g | NIST data |
| μ | cm⁻¹ | (μ/ρ) × ρ |

**Red flags**: Multiplying cm by mm, adding MeV to keV, using degrees in `sin()`/`cos()`.

## Step 4: Verify CUDA Kernel Indexing

For all `@cuda.jit` functions:
- Thread/block grid dimensions match the data arrays
- `cuda.grid(3)` returns (x, y, z) in the correct order
- Bounds checking: `if x >= shape[0]: return` pattern present
- Shared/local array sizes sufficient for worst-case raytracing
- No race conditions except where `cuda.atomic.add` is used

## Step 5: Check Float32 Precision

CUDA kernels default to float32. Identify locations where float64 might be needed:
- Cumulative kernel interpolation (many small increments)
- DDA t-value accumulation over long rays
- Exponential attenuation for large d_eff values
- Summation of many small dose contributions

Flag these but do not change — document for future review.

## Step 6: Validate Edge Cases

Check handling of:
- **OAD → 0**: Division by zero in inverse-square terms. Code should use max(epsilon, OAD).
- **d_eff = 0**: Surface voxels with zero radiological depth. TERMA should still compute correctly.
- **Blocked voxels**: Transmission = 0. TERMA should be 0, not NaN.
- **Ray termination**: DDA ray reaches grid boundary before kernel is fully deposited.
- **Empty kernel bins**: Zero spectral weights (4.5, 5.5 MeV) — skip or multiply by zero.

## Step 7: Cross-Reference with Existing Tests

Read the corresponding test file. Check:
- Do tests cover the specific physics being verified?
- Are there both happy-path and edge-case tests?
- Do test values come from published data or hand calculations?
- Are tolerances appropriate for float32 precision?

## Step 8: Compare Against Analytical Solutions

Where possible, verify against known solutions:
- **Point source in water**: Fluence ∝ 1/d², TERMA ∝ exp(-μd)/d²
- **Square field blocking**: Transmission should match geometric projection
- **Kernel normalisation**: Sum of all differential values should equal 1.0
- **DDA raytrace**: Number of voxels traversed should match expected path length

## Step 9: Document Findings

Write a verification report with:
- Module name and file path
- Date and verifier
- List of findings with severity ratings
- Summary assessment

## Step 10: Update Verification Dashboard

Edit `context/lookup/verification-dashboard.md`:
- Change status from "Pending" to "In Progress" or "Pass" or "Issues Found"
- Update Critical/Warning/Info counts
- Add OpenSpec reference if proposal created

## Step 11: Propose Fixes for Issues

If issues rated Critical or Warning are found:
- Use `/opsx-propose` to create a formal OpenSpec change
- Reference the verification finding in the proposal
- Link the OpenSpec change in the dashboard

## Severity Definitions

| Severity | Definition | Action Required |
|----------|-----------|-----------------|
| **Critical** | Wrong physics answer — incorrect equation, wrong units, missing term | Must fix before any clinical use |
| **Warning** | Correct result but fragile — float32 risk, missing bounds check, unclear code | Should fix, OpenSpec proposal recommended |
| **Info** | Style or documentation issue — missing comments, naming inconsistency | Fix opportunistically |

## Key Reference Equations for Spot-Checks

- **Inverse square**: Φ ∝ 1/d² — check no d³ or d terms
- **Exponential attenuation**: exp(-μd) — check μ in correct units (cm⁻¹)
- **TERMA**: T = Φ × E × (μ/ρ) — check all three terms present
- **Kernel sum**: Σk(r,θ)×Δr = 1.0 — verify normalisation
