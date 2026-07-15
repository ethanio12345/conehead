# Implementation Workflow Guide

Step-by-step guide for implementing a new feature in the conehead dose calculation engine.

## Step 1: Check for Existing OpenSpec Spec

```bash
ls openspec/changes/{feature-name}/
```

If a spec exists, read the proposal and design documents. The spec contains:
- Requirements and acceptance criteria
- Reference equations from papers
- Integration points in the pipeline
- Task breakdown with dependencies

If no spec exists, consider whether the feature warrants one (anything physics-critical does).

## Step 2: Read Reference Paper

Locate and read the relevant reference paper(s):
- Cho et al (2012) — Varian 6MV beam model, source parameters
- Yang et al (2002) — Three-component source model
- Ahnesjö (1989) — Collapsed cone convolution method

Extract the specific equations to implement. Note:
- Exact equation numbers from the paper
- Variable definitions and units
- Assumptions and approximations stated
- Validation data or benchmark results

## Step 3: Study Existing Codebase Patterns

Before writing any code, study how similar features are implemented:

### CUDA Kernel Pattern
```python
from numba import cuda

@cuda.jit
def cuda_feature(input_array, output_array, params):
    x, y, z = cuda.grid(3)
    if x >= output_array.shape[0]:
        return
    if y >= output_array.shape[1]:
        return
    if z >= output_array.shape[2]:
        return
    # ... computation ...
    output_array[x, y, z] = result
```

### Settings Dict Pattern
```python
settings = {
    "sPri": np.float32(0.90924),
    "zAnn": np.float32(4.0),
    # ... descriptive key names ...
}
```

### Type Hints
```python
import numpy.typing as npt
result: npt.NDArray[np.float32] = ...
```

### Progress Reporting
```python
print("Calculating TERMA...")
# ... computation ...
print("TERMA complete.")
```

## Step 4: Design the Implementation

List all files to modify or create:

```
Files to modify:
- conehead/conehead.py  (add CUDA kernel, integrate into calculate())
- conehead/settings.py  (add new parameters)

Files to create:
- tests/test_new_feature.py  (test suite)

Integration point:
- calculate() function: insert step between existing steps N and N+1
```

Identify data dependencies:
- What inputs does this step need from previous steps?
- What outputs does it produce for subsequent steps?
- What parameters/settings does it require?

## Step 5: Write Failing Tests (TDD)

Define expected physics behavior as tests:

```python
def test_basic_behavior():
    """Feature should produce correct result for simple case."""
    # Setup with known inputs
    # Execute
    # Assert against analytical solution

def test_edge_case():
    """Feature should handle boundary conditions."""
    # Setup with edge-case inputs
    # Execute
    # Assert no NaN/Inf, reasonable values

def test_integration():
    """Feature should integrate correctly with pipeline."""
    # Full pipeline test with feature enabled
    # Compare against baseline (feature disabled)
```

## Step 6: Implement the Feature

Follow the patterns identified in Step 3:
- Use `@cuda.jit` for GPU computation
- Use float32 throughout CUDA kernels
- Follow the settings dict convention for parameters
- Add clear comments referencing paper equations
- Use print() for progress reporting

## Step 7: Run Tests

```bash
uv run ruff format conehead/
uv run ruff check --fix conehead/
uv run rm -rf .mypy_cache && uv run mypy conehead/
uv run pytest tests/test_new_feature.py -v
```

Run in this exact order: format → lint → typecheck → test.

## Step 8: Request Physics Verification

After tests pass, flag the new code for physics verification:
- Follow `context/guides/verification-workflow.md`
- Or delegate to physics-verifier agent with the source file path

## Step 9: Update Milestone Tracker

Edit `context/lookup/milestone-tracker.md`:
- Change relevant milestone status
- Add notes about what was implemented

## Step 10: OpenSpec Verification (if applicable)

If the feature had an OpenSpec spec:
```bash
/opsx-verify {feature-name}
```
This checks that the implementation meets the spec's acceptance criteria.

## Common Implementation Pitfalls

1. **Forgetting bounds checks** in CUDA kernels — causes silent out-of-bounds writes
2. **Using float64 constants** in CUDA kernels — forces expensive casts, may not match grid dtype
3. **Not normalising direction vectors** before DDA raytracing
4. **Mixing cm and mm** — conehead uses cm internally, DICOM uses mm, block uses tenths-of-mm
5. **Forgetting to multiply by transmission factor** — blocked voxels must have zero fluence
