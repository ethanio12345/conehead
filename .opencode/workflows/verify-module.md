# Workflow: Verify Module

Deep verification of a conehead module against physics literature.

## When to Use
- Verifying an existing module for correctness
- Reviewing algorithm implementations against published papers
- Validating CUDA kernel logic
- After making changes that could affect physics accuracy

## Prerequisites
- Module file(s) identified for verification
- Reference paper(s) available (Cho et al 2012, Yang et al 2002, etc.)
- context/lookup/verification-dashboard.md accessible

## Steps

### Stage 1: Setup
1. Read the target source file(s) completely
2. Read related test file(s) in tests/
3. Load relevant context: concepts/ and lookup/ files
4. Check verification-dashboard.md for any prior verification
5. Update dashboard status to "In Progress"

### Stage 2: Equation Mapping
1. For each function/kernel, identify the physics equation being implemented
2. Map to specific equation numbers in reference papers
3. Document expected units for all inputs and outputs
4. Note any approximations or simplifications made

### Stage 3: Detailed Verification
1. **Equation fidelity**: Does the code faithfully implement the published equation?
2. **Unit consistency**: Are cm/mm/tenths-of-mm handled correctly throughout?
3. **Numerical precision**: Is float32 adequate? Are there accumulation errors?
4. **Edge cases**: Zero divisions, boundary voxels, array bounds, singularities
5. **CUDA correctness**: Thread indexing, atomic operations, device memory
6. **Algorithmic correctness**: DDA traversal, interpolation, kernel indexing

### Stage 4: Report
1. Structure findings by severity: Critical / Warning / Info
2. For each finding: file:line, expected behavior, actual behavior, reference, suggested fix
3. List all functions/equations that passed verification
4. Provide overall module assessment: PASS / ISSUES FOUND

### Stage 5: Update State
1. Update verification-dashboard.md with results
2. If Critical or Warning issues found:
   - Suggest `/opsx-propose` to create formal tracking
   - Include finding details as proposal content
3. If all PASS:
   - Mark module as verified in dashboard
   - Note the date and reference papers used

## Context Dependencies
- context/concepts/ (domain knowledge)
- context/lookup/verification-dashboard.md (state tracking)
- context/lookup/nist-photon-data.md (reference data)
- context/errors/common-physics-pitfalls.md (known issues)

## Success Criteria
- All functions/kernels in the module have been reviewed
- Findings are documented with severity and fix suggestions
- verification-dashboard.md is updated
- OpenSpec proposal created if issues found
