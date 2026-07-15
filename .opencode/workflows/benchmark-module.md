# Workflow: Benchmark Module

Profile and optimize performance of a conehead module.

## When to Use
- Performance feels slow
- Comparing before/after optimization
- Profiling CUDA kernel efficiency
- Memory usage analysis
- Before and after Cython optimization

## Prerequisites
- Target module/kernel identified
- Baseline benchmark established or need to establish one
- CUDA-capable GPU available (or Cython CPU path)

## Steps

### Stage 1: Baseline
1. Identify the target kernel/function to benchmark
2. Create a benchmark script with realistic input sizes
3. Run baseline timing (wall clock + CUDA events if possible)
4. Record: execution time, memory transfers, thread utilization
5. If available, run nvprof/nsight for detailed profiling

### Stage 2: Analysis
1. Identify bottlenecks:
   - Memory bandwidth limited? (data movement dominates)
   - Compute limited? (arithmetic dominates)
   - Occupancy limited? (not enough threads to hide latency)
2. Check for unnecessary device/host transfers
3. Analyze thread block dimensions for GPU occupancy
4. Review local array sizes in DDA (potential waste or overflow)
5. Check memory access patterns (coalesced vs scattered)

### Stage 3: Optimize
Apply in order of expected impact:
1. Eliminate redundant cuda.to_device/copy_to_host calls
2. Fuse sequential kernels where dependencies allow
3. Optimize thread block dimensions (check occupancy calculator)
4. Reduce local array sizes in DDA to actual needed length
5. Use shared memory for frequently accessed read-only data
6. Improve memory coalescing (reorder array dimensions if needed)
7. Consider float16 for non-critical calculations

### Stage 4: Validate
After each optimization:
1. Run benchmark again
2. Verify numerical results are identical (within float32 precision)
3. Record speedup factor
4. If results differ, investigate and either fix or document

### Stage 5: Report
Present as:
| Optimization | Time (ms) | Speedup | Numerical Impact |
|---|---|---|---|

## Context Dependencies
- context/examples/cython-optimization-patterns.md (optimization techniques)
- context/concepts/ (understanding what the code computes)
- context/guides/codebase-dataflow.md (data flow for identifying transfer bottlenecks)

## Success Criteria
- Before/after benchmarks recorded
- Numerical accuracy preserved
- Each optimization documented with rationale
- At least 1.5x speedup achieved (or explanation why not)
