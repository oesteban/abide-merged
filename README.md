# ABIDE I+II Merged BIDS Dataset

A merged BIDS view of the Autism Brain Imaging Data Exchange
([ABIDE I](http://fcon_1000.projects.nitrc.org/indi/abide/abide_I.html) and
[ABIDE II](http://fcon_1000.projects.nitrc.org/indi/abide/abide_II.html)) datasets.
This dataset was programmatically assembled by `build_abide_both.py`, which
reuses git-annex keys from the original per-site DataLad subdatasets and
generates per-file JSON sidecars from site-level metadata templates.

## Contents

| Item | Count | Notes |
|------|------:|-------|
| Subjects | 2,194 | 1,112 ABIDE I + 1,082 ABIDE II |
| Sessions | 1--2 | All subjects have `ses-1`; 46 ABIDE II subjects also have `ses-2` |
| Datatypes | anat, func, dwi | No fieldmap (`fmap`) data |
| T1w images | 2,529 | Some subjects have multiple runs (up to 5) |
| BOLD runs | ~2,400+ | All `task-rest`; some with `acq-` labels |
| DWI directories | 284 | ABIDE II subset only; no JSON sidecars |
| FLAIR images | 49 | ABIDE II subset only; no JSON sidecars |

## Subject ID Scheme

Subject labels encode the source dataset, site, and original ID:

- **ABIDE I**: `sub-v1s<siteindex>x<orig>` (e.g., `sub-v1s0x0050642`)
- **ABIDE II**: `sub-v2s<siteindex>x<orig>` (e.g., `sub-v2s0x29006`)

where `siteindex` is the zero-based alphabetical index of the site within its
source dataset and `orig` is the original numeric subject identifier.
The mapping is recorded in `participants.tsv`.

## Participants Metadata

`participants.tsv` contains the following columns:

| Column | Description |
|--------|-------------|
| `participant_id` | BIDS subject label |
| `source_dataset` | `abide1` or `abide2` |
| `source_site` | Original site name (43 unique sites) |
| `site_index` | Zero-based alphabetical site index within dataset |
| `source_subject_id` | Original numeric subject ID |

## Acquisition Variants

Some ABIDE II sites use BIDS `acq-` labels to distinguish acquisition variants:

- `acq-pedj` / `acq-pedi` -- phase-encoding direction variants (site s11)
- `acq-rc8chan` / `acq-rc32chan` -- receive-coil variants (site s6)
- `acq-hires` -- high-resolution T1w (site s15, 148 files)

## Data Use Agreement

Users must comply with the ABIDE data use agreement.
See <http://fcon_1000.projects.nitrc.org/indi/abide/> for terms.

## BIDS Audit Log

The following audit was performed against BIDS specification v1.10.0
on 2026-02-19.

### Errors (will cause validation failures)

#### ~~String values where numeric is REQUIRED~~ (FIXED 2026-02-19)

Resolved. Philips "shortest" auto-selection strings were replaced with
actual numeric values from the
[ABIDE II scan parameter documents](http://fcon_1000.projects.nitrc.org/indi/abide/scan_params/):

| Site | Field | Old value | Fixed value | Files |
|------|-------|-----------|-------------|------:|
| BNI_1 (v2s0) | `RepetitionTime` | `"shortest"` | `0.0067` | 58 |
| BNI_1 (v2s0) | `EchoTime` | `"Shortest"` | `0.0031` | 58 |
| ETHZ_1 (v2s2) | `RepetitionTime` | `"0.0084"` | `0.0084` | 37 |
| ETHZ_1 (v2s2) | `EchoTime` | `"Shortest"` | `0.0039` | 37 |

#### ~~Misspelled BIDS metadata keys~~ (FIXED 2026-02-19)

Resolved. Misspelled keys were renamed across 2,575 JSON sidecars:

| Typo | Renamed to | Files |
|------|------------|------:|
| `MagneicFieldStrength` | `MagneticFieldStrength` | 2,237 |
| `MagneicFieldSrengh` | `MagneticFieldStrength` | 301 |
| `PlaneOrientationSequentialuence` | `PlaneOrientationSequence` | 2,575 |
| `PulseSequentialuenceType` | `PulseSequenceType` | 301 |

#### ~~Missing DWI JSON sidecars~~ (FIXED 2026-02-19)

Resolved. 360 DWI JSON sidecars created from upstream ABIDE II site
templates and [NITRC scan parameter documents](http://fcon_1000.projects.nitrc.org/indi/abide/scan_params/)
for 6 sites: BNI_1 (58), IP_1 (53), NYU_1 (114), NYU_2 (38),
SDSU_1 (57), TCD_1 (40).

#### Missing FLAIR JSON sidecars

49 FLAIR `.nii.gz` files have no corresponding `.json` sidecars.

#### Missing `participants.json`

No data dictionary for `participants.tsv`. BIDS RECOMMENDS a
`participants.json` describing each column.

### Warnings (best practice violations)

#### `dataset_description.json` gaps

- Missing `License` (RECOMMENDED) -- ABIDE has specific data use agreements.
- Missing `Authors` (RECOMMENDED).
- Missing `DatasetDOI` (RECOMMENDED).
- Missing `SourceDatasets` (RECOMMENDED for merged datasets) -- should
  reference ABIDE I and ABIDE II.
- Contains `GeneratedBy`, which is a BIDS-Derivatives field; may be
  misleading for `DatasetType: "raw"`.

#### No inheritance-level JSON sidecars

Every BOLD and T1w file has its own per-file sidecar (~5,100 JSONs), even
though subjects from the same site share identical acquisition parameters.
Using the BIDS inheritance principle with site-level or top-level sidecars
would:

- Reduce the number of JSON files by ~90%.
- Make site-wide metadata corrections trivial (fix one file, not hundreds).
- Reduce repository size.

For example, a top-level `task-rest_bold.json` with `{"TaskName": "rest"}`
would eliminate that key from every per-file sidecar.

#### Non-numeric values in numeric fields

Beyond `RepetitionTime`/`EchoTime`, several other fields use strings where
numbers or arrays are expected:

| Field | Example value | Expected type |
|-------|--------------|---------------|
| `NumberofMeasurements` | `"120"` | integer |
| `ParallelReductionFactorIn-plane` | `"2"` | number |
| `PixelBandwidth` | `"3280"` | number |
| `AcquisitionDuration` | `"8:06"` | seconds (number) |
| `PixelSpacing` | `"3x3"` | array `[3, 3]` |
| `AcquisitionMatrix` | `"64x64"` | array `[64, 64]` |
| `FieldofViewDimensions` | `"192x192"` | array `[192, 192]` |

#### Inconsistent `run` entity usage

- ABIDE I T1w: no `run` entity (e.g., `sub-v1s0x0050642_ses-1_T1w.nii.gz`).
- ABIDE II T1w: uses `run-1` even for single-run subjects.

Using `run-1` for single-run subjects is valid but inconsistent across the
two source datasets.

### Notes (structural observations)

- **No fieldmap data**: No `fmap/` directories exist. Susceptibility
  distortion correction will rely on fieldmap-less methods (e.g., SyN-SDC).
- **Multi-session subjects**: 46 ABIDE II subjects have `ses-2` with func
  data only (no anat).
- **`acq` entity in BOLD**: Some sites encode technical metadata
  (phase-encoding direction, receive coil) in the `acq-` label rather than
  in JSON sidecars.
- **No `CHANGES` file**: BIDS RECOMMENDS tracking dataset versions.
