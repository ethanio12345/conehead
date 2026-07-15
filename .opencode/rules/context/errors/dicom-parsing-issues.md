# DICOM RT Parsing Issues

Common issues when parsing DICOM RT data for conehead and how to handle them.

## 1. Missing Tags

Not all DICOM files include all optional tags. Robust parsing requires defensive access:

```python
# Wrong — crashes on missing tag
beam_name = ds.BeamName

# Correct — safe access with default
beam_name = getattr(ds, 'BeamName', 'Unknown')
# Or check existence first
if hasattr(ds, 'BeamName'):
    beam_name = str(ds.BeamName)
```

Common missing tags:
- `InstitutionName`, `Manufacturer` — identification (optional)
- `ReferencedDoseSequence` — may not exist in plan files
- `PatientSetupSequence` — may have zero or multiple entries

## 2. Beam Type Validation

Current code only handles `BeamType = 'STATIC'`:

```python
beam_type = str(ds.BeamType)
if beam_type != 'STATIC':
    raise ValueError(f"Unsupported beam type: {beam_type}")
```

**Must extend for:**
- `DYNAMIC` — IMRT beams with moving MLC leaves per control point
- Each control point defines a separate MLC configuration
- Control points are interpolated (MU-based) for VMAT

The `ControlPointSequence` for DYNAMIC beams contains:
- `CumulativeMetersetWeight` — fraction of total MU at this control point
- `BeamLimitingDevicePositionSequence` — jaw/MLC positions at this control point
- `GantryAngle` — may vary for VMAT (arc therapy)

## 3. MLC Bank Identification

MLC leaf positions are stored in `BeamLimitingDevicePositionSequence`:

```python
# RT Plan structure for MLC positions
for device in beam_limiting_device_positions:
    rt_beam_limiting_device_type = str(device.RTBeamLimitingDeviceType)
    if rt_beam_limiting_device_type == 'MLCX':
        # LeafJawPositions: [A1, A2, ..., A60, B1, B2, ..., B60]
        positions = device.LeafJawPositions
        n_leaves = len(positions) // 2
        bank_a = positions[:n_leaves]   # First half = Bank A
        bank_b = positions[n_leaves:]   # Second half = Bank B
```

**Gotchas:**
- Leaf count varies by MLC model (60, 80, 120 pairs)
- Some systems store [A1, B1, A2, B2, ...] instead of [A..., B...]
- Negative positions indicate retracted leaves (fully open)

## 4. Coordinate System Verification

DICOM RT uses IEC 61217, but some vendors deviate:

```python
# Always verify coordinate system by checking known geometry
# Example: gantry 0° source should be at (0, -SAD, 0) in IEC 61217
source_pos = compute_source_position(gantry_angle=0)
assert source_pos[1] == -100.0  # Y = -SAD for gantry 0°
```

**Vendor-specific quirks:**
- Varian: standard IEC 61217
- Elekta: may use different collimator angle convention
- Siemens: may use different patient orientation

## 5. Unit Conversion

DICOM positions are in mm; conehead uses cm internally:

```python
# DICOM → conehead conversion
isocenter_cm = np.array(ds.IsocenterPosition, dtype=np.float32) / 10.0
# mm → cm

# Block positions additionally converted to tenths-of-mm for indexing
block_position_tmm = dicom_position_mm * 10  # mm → tenths-of-mm
```

**Watch for:**
- `ImagePositionPatient` in mm
- `GridFrameOffsetVector` in mm
- `LeafJawPositions` in mm
- `IsocenterPosition` in mm

## 6. Leaf Transmission

Hardcoded edge transmission values:

```python
# Current implementation
edge_transmission = [0.20, 0.50, 0.75]  # First 3 rows
```

**Issues:**
- These values are MLC-model-specific (Millennium vs HD-MLC)
- Should be configurable per machine
- Some MLC models have interleaf leakage that varies with leaf position

## 7. Jaw vs MLC Ordering

The block calculation applies jaws and MLC in a specific order:

```
1. MLC transmission → block_plane (leaf-by-leaf)
2. Jaw transmission → overlay on block_plane
3. Combined transmission = min(MLC_transmission, jaw_transmission)
```

**Correct ordering matters** because:
- Jaws have lower transmission than MLC
- Overlapping jaw + MLC regions should use the more attenuating value
- Wrong order can overestimate transmission at field edges

## 8. CT Slice Ordering

DICOM CT files are not guaranteed to be in spatial order:

```python
# Sort slices by Z position (ImagePositionPatient[2])
slices = sorted(dicom_files, key=lambda s: float(s.ImagePositionPatient[2]))
```

**Without sorting:**
- Reconstructed 3D volume has scrambled Z-axis
- Density grid is wrong → d_eff is wrong → dose is wrong
- Error may not be visually obvious in the dose distribution

## 9. HU to Density Conversion

CT Hounsfield Units (HU) must be converted to electron density:

```python
# Linear conversion (simplified)
density = 1.0 + HU / 1000.0  # g/cm³

# More accurate: piecewise conversion curve
# Different CT scanners may use different calibration curves
```

**Required information:**
- CT-to-density curve from scanner calibration
- May be stored in DICOM as `CTScalingCurve` or externally
- Bone (HU > 1000) requires special handling

## 10. RT Plan Multi-Beam References

A single RT Plan may reference multiple beams:

```python
# Iterate over all beams
for beam in ds.BeamSequence:
    beam_number = beam.BeamNumber
    # Each beam has its own:
    # - ControlPointSequence
    # - BeamLimitingDevicePositionSequence
    # - GantryAngle, CollimatorAngle
    # - IsocenterPosition
```

**Ensure correct beam selection:**
- Match `BeamNumber` between RT Plan and RT Plan references
- A single fraction may deliver multiple beams
- Each beam requires independent dose calculation, then summation
