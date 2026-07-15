---
description: Main command for conehead radiotherapy dose calculation project. Routes to the conehead-orchestrator agent which analyzes the request and dispatches to the appropriate specialist (physics-verifier, dose-engine-dev, cython-optimizer, dicom-specialist, or test-author).
---

# /conehead

The single entry point for all conehead project work.

## Usage

```
/conehead "{request}"
```

## Examples

### Verification
```
/conehead "verify geometry.py"
/conehead "verify the fluence calculation against Cho et al"
/conehead "check all CUDA kernels for correctness"
/conehead "show verification status"
```

### Implementation
```
/conehead "implement kernel tilting"
/conehead "add CT image import"
/conehead "implement IMRT DICOM parsing"
/conehead "add density overrides for structures"
```

### Performance
```
/conehead "benchmark the dose convolution kernel"
/conehead "optimize cuda_d_eff"
/conehead "profile the full pipeline"
```

### DICOM
```
/conehead "parse tests/RP.3DCRT.dcm"
/conehead "validate the RT plan structure"
/conehead "load a CT image series"
```

### Testing
```
/conehead "write tests for kernel.py"
/conehead "add regression tests for fluence"
/conehead "run all tests"
```

### Status
```
/conehead "show verification progress"
/conehead "what milestones are remaining"
/conehead "status"
```

## How It Works

The command routes your request to the conehead-orchestrator agent, which:

1. Analyzes the request to determine the task type
2. Routes to the appropriate specialist subagent
3. Manages state tracking and OpenSpec integration
4. Returns consolidated results

## Specialist Agents

| Agent | Handles |
|-------|---------|
| physics-verifier | Verification, validation, physics review |
| dose-engine-dev | Feature implementation, new milestones |
| cython-optimizer | Performance, benchmarking, optimization |
| dicom-specialist | DICOM parsing, CT images, RT plans |
| test-author | Test writing, regression suites, coverage |
