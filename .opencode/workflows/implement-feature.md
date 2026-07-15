# Workflow: Implement Feature

Implement a new milestone or feature for the conehead project.

## When to Use
- Implementing a remaining milestone (kernel tilting, CT import, etc.)
- Adding new capabilities to the dose calculation pipeline
- Extending DICOM parsing (IMRT/VMAT)
- Adding density overrides or out-of-field dosimetry

## Prerequisites
- Feature clearly defined (milestone name or OpenSpec spec)
- Reference paper identified for relevant equations
- Existing tests passing

## Steps

### Stage 1: Design
1. Check for OpenSpec specification in openspec/changes/{feature}/
2. If spec exists, read it thoroughly — it is the implementation plan
3. If no spec, read reference paper and design from first principles
4. Identify files to modify and new files needed
5. List integration points in Conehead.calculate() pipeline

### Stage 2: Test Design (TDD)
1. Write failing tests that define expected behavior
2. Include physics validation where analytical solutions exist
3. Define acceptance criteria with tolerances
4. Use pytest fixtures following existing test patterns

### Stage 3: Implementation
1. Follow existing CUDA kernel patterns (thread idx, grid dims, device memory)
2. Use float32 consistently for GPU arrays
3. Add clear docstrings with equation references
4. Handle edge cases (boundary conditions, zero divisions)
5. Add progress print statements for long operations
6. Integrate into calculate() pipeline at the correct position

### Stage 4: Verification
1. Run tests — all must pass
2. Request physics-verifier review of new code
3. Request cython-optimizer review if performance-sensitive
4. Compare results against known data where available

### Stage 5: Update State
1. Update milestone-tracker.md with implementation status
2. If OpenSpec was used, mark change as ready for /opsx-verify
3. Update verification-dashboard.md with new module entries
4. Document any design decisions or deviations from spec

## Context Dependencies
- context/concepts/ (physics theory)
- context/guides/implementation-workflow.md (how-to)
- context/guides/codebase-dataflow.md (pipeline structure)
- context/lookup/milestone-tracker.md (state tracking)
- context/examples/dose-calculation-patterns.md (code patterns)

## Success Criteria
- Feature implemented and integrated into pipeline
- All tests pass including new physics validation tests
- Code reviewed by physics-verifier
- milestone-tracker.md updated
- OpenSpec change ready for verification (if applicable)
