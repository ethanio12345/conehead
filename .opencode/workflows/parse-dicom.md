# Workflow: Parse DICOM

Parse, validate, or extend DICOM RT handling in conehead.

## When to Use
- Loading an RT Plan for dose calculation
- Importing CT images for patient geometry
- Extracting structure sets for ROI contours
- Extending DICOM parsing for IMRT/VMAT
- Validating DICOM file structure

## Prerequisites
- DICOM file(s) available (RT Plan, CT, Structure Set)
- pydicom installed
- Understanding of IEC 61217 coordinate conventions

## Steps

### Stage 1: Load and Validate
1. Load DICOM file with pydicom
2. Verify required tags exist for the IOD type
3. Check SOP Class UID matches expected type
4. Validate patient coordinate system
5. Report any missing optional tags (informational)

### Stage 2: Extract Data

For RT Plans:
1. Iterate BeamSequence
2. Extract BeamType (STATIC vs DYNAMIC)
3. Get BeamLimitingDeviceSequence for MLC boundaries
4. Get ControlPointSequence for positions
5. For DYNAMIC: iterate all control points, extract cumulative MUs

For CT Images:
1. Load all slices in series
2. Sort by ImagePositionPatient[2]
3. Extract pixel_array and apply RescaleSlope/Intercept
4. Determine grid origin, spacing, and size
5. Convert HU to electron density using conversion curve

For Structure Sets:
1. Extract ROI contours
2. Map contour data to grid coordinates
3. Create boolean masks for each structure

### Stage 3: Convert to Conehead Format
1. RT Plan → Block class (block_values array)
2. CT Images → phantom-compatible structure (origin, spacing, densities)
3. Structure Set → density override map
4. Ensure unit conversions (mm → cm → tenths-of-mm where needed)
5. Validate geometric relationships (SAD, field sizes)

### Stage 4: Integration
1. Create appropriate conehead objects (Source, Block, Phantom)
2. Set source gantry/collimator from plan
3. Configure block from MLC/jaw positions
4. Set up phantom from CT or use water phantom fallback
5. Configure settings dict with plan parameters

### Stage 5: Validate
1. Request test-author to write tests for new parsing
2. Request physics-verifier to check coordinate transforms
3. Visual inspection of block plane (if applicable)

## Context Dependencies
- context/errors/dicom-parsing-issues.md (known pitfalls)
- context/concepts/varian-linac-6mv.md (beam parameters)
- context/guides/codebase-dataflow.md (where DICOM data enters pipeline)

## Success Criteria
- DICOM data loaded without errors
- Coordinate system verified (IEC 61217)
- Conehead objects correctly initialized
- Tests written for the parsing
- Geometric validation passed
