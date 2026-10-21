# ADNI AD/CN Classification: CNN Baseline and ConvNeXt

COMP3710 individual project — s4982104.

## Status

Initial project scaffold. Models, patient-level splits, training, and evaluation
have not yet been implemented. No experimental results are claimed.

## Project objective

Compare a simple CNN baseline with ConvNeXt for AD versus cognitively normal
classification. Evaluate accuracy, confidence reliability, referral of uncertain
cases for human review, GPU memory, and inference time.

## Data

Course data location: `/home/groups/comp3710/ADNI` on Rangpur.
The initial read-only audit found 30,520 JPEG slices, 1,526 image IDs, and
680 patients. The supplied train/test folders share 216 patients; create
patient-disjoint splits using `meta_data_with_label.json` before training.
Folder `NC` denotes the normal class referred to as `CN` in the task.
Do not commit images or model weights.

## File structure

- `modules.py`: CNN and ConvNeXt model components.
- `dataset.py`: data loading, preprocessing, augmentation, and patient splits.
- `train.py`: training, validation, testing, checkpoint saving, and curves.
- `predict.py`: load saved weights, predict, and visualize examples.
- `README.md`: report and reproduction instructions.

## Feasibility review

TODO: user needs, 3–5 acceptance criteria, model choice, preliminary evidence,
resource budget, risks, and fallback.

## Environment and usage

The data preparation stage uses Python 3.11 and Pillow 12.3.0.
Verified locally with Python 3.11.15; the recorded Rangpur selection run used
Python 3.11.13. Model-training dependencies will be added when implemented.

Run these commands from this project directory to create an isolated environment:

```bash
python3.11 -m venv .venv
.venv/bin/python -m pip install -r requirements.txt
.venv/bin/python -m unittest discover -s tests -v
```

Use `.venv/bin/python` for project scripts, or activate the environment first:

```bash
source .venv/bin/activate
python -m unittest discover -s tests -v
```

In VS Code, use **Python: Select Interpreter**, then choose this project's
`.venv/bin/python`. When opening the whole repository, it is located at
`recognition/ConvNeXt_ADNI_s4982104/.venv/bin/python`.
Selecting a different interpreter in an existing terminal requires activating
the environment or using the explicit executable path above.
Create a separate environment on Rangpur; do not copy the macOS `.venv` there.
The virtual environment is ignored by Git.

## Generate the sample manifest (B2)

Run from this project directory on Rangpur with Python 3 (standard library only):

```bash
python3 generate_samples.py \
  --data-root /home/groups/comp3710/ADNI \
  --output-dir "$HOME/comp3710-data/adni"
```

The script scans JPEG filenames in both original `train` and `test` directories
and joins them to `meta_data_with_label.json`. It does not decode image pixels,
modify source files, create new splits, or train a model.

The output columns are `file_path`, `original_split`, `image_id`, `patient_id`,
`scan_date`, and `label`. Paths are absolute cluster paths. `original_split`
records the source folder only. Image IDs come from JPEG filenames; patient IDs
come from the matching JSON `raw` path, whose image ID is cross-checked. Labels
are filled only when folder AD/NC agrees with JSON integer 2/0 respectively.
Missing or conflicting information is left blank and recorded in
`samples_issues.csv`; the image row is retained.

**Date policy:** the supplied JSON contains no documented acquisition-date
field. `scan_date` is therefore blank for every current image. A timestamp in
a filename is not sufficient evidence of the acquisition date.

Generated files remain in the personal cluster output directory:

- `samples.csv`: one row per JPEG.
- `samples_issues.csv`: field-level issues, including unavailable scan dates.
- `manifest_summary.json`: generation metadata and issue counts.

These outputs are ignored by Git. Do not commit or copy real patient-level
manifests into the repository. Existing outputs are not overwritten; to rerun,
choose a new personal output directory (outside the source data tree).

The current cluster copy of the script is at
`~/comp3710-code/ConvNeXt_ADNI_s4982104/generate_samples.py`.

Run the synthetic-fixture tests locally from this project directory:

```bash
python3 -m unittest discover -s tests -v
```

B2 verification covers the header and representative filename/JSON mappings.
Full dataset reconciliation and split validation belong to the next step.

## Check and select scans (B3)

Requires Python 3 and Pillow. Run on a SLURM compute node (CPU only):

```bash
python select_scans.py \
  --input "$HOME/comp3710-data/adni/samples.csv" \
  --output-dir "$HOME/comp3710-data/adni/selection_seed42" \
  --seed 42
```

The script verifies JPEG structure and fully decodes every slice with Pillow
(`LOAD_TRUNCATED_IMAGES=False`). A scan must contain 20 distinct slice indices,
20 readable JPEGs, one valid patient ID and a consistent AD/NC label. Duplicate
paths, filename/image-ID disagreement, and patient label conflicts exclude the
whole scan. Dimensions (width, height), modes and channel counts are reported;
size differences alone do not exclude scans. This checks file integrity, not
medical image quality. Source files remain unchanged.

`scan_checks.csv` has one row per image ID, explicit failure reasons, and
`patient_qualified_scan_count` for querying candidate availability. If any patient
has zero qualified scans, only checks and a blocked summary are written (exit 2).
Discuss those cases before selecting; never silently drop a patient.

For selection, sort patient IDs as strings and each patient's image IDs as
integers, initialize `random.Random(42)` once, then call `choice` once per patient.
Keep the entire chosen scan, not a mixture of randomly sampled slices. Both
models must use these same manifests. Dates remain blank.

Outputs: `scan_checks.csv`, `selected_scans.csv`, `selected_samples.csv`, and
`selection_summary.json`. The summary includes counts, input/check/output hashes,
Python/Pillow versions, script hash and Git provenance. It verifies record subset,
unique paths, 20 distinct slices, complete patient retention, and repeated and
shuffled-input selection with a fresh RNG. Fixed seed reproducibility depends on
the same candidate pool and implementation; changed inputs require new outputs.
Existing output directories are refused. Real outputs stay outside Git.

When copying the script alone to Rangpur, pass `--code-version-json` with the
local `code_version()` result; its script hash is validated before use. The base
commit describes the repository snapshot, not a commit of uncommitted changes.
No train/validation/test split is performed in B3.

## Stratified patient splits (B6)

Run with the project virtual environment; original JPEGs are not needed:

```bash
.venv/bin/python split_patients.py \
  --selection-dir /path/to/selection_seed42 \
  --output-dir /path/to/split_seed42_60_20_20
.venv/bin/python -m unittest discover -s tests -v
```

The selection summary must have `status=passed` and SHA-256 hashes matching both
input CSVs. Validation requires 680 unique patients (221 AD / 459 NC), one unique
scan per patient, 20 distinct slices per scan, unique paths, and exact patient,
image and label associations. Invalid input fails before outputs are created.
Acquisition dates remain blank and no scan is reselected.

Algorithm `stratified-fixed-quotas-v1`: sort patient ID strings separately for AD
and NC; initialize one independent `random.Random(42)`; shuffle AD then NC once
each. Assign successive patients to train/validation/test using AD quotas
133/44/44 and NC quotas 275/92/92. This gives 408/136/136 patients and
8160/2720/2720 slices. `original_split` is retained solely as source provenance;
future loaders must use the new `split` column. CNN and ConvNeXt share the outputs.
Do not change assignment or seed based on model performance.

`patient_splits.csv` is sorted by patient ID. `split_samples.csv` preserves all
six input fields and adds `split`, sorted by patient ID, image ID and file path.
`split_summary.json` contains only aggregate counts, all pairwise patient/image/
path intersections, repeat and shuffled-input validation, input/output hashes,
Python version, algorithm and code provenance. The committed-script flag compares
actual script bytes to HEAD; worktree dirtiness comes from Git status, not a
hardcoded value. A standalone cluster copy may use `--code-version-json` exported
from the local script's `code_version()`; its script hash is checked before use.
The base commit is not a commit of pending changes.

Use new output directories; existing directories and outputs inside the selection
source are refused. Canonical cluster outputs are in
`~/comp3710-data/adni/split_seed42_60_20_20/`. Local copies belong in
`final_report/adni_manifest_local/split_seed42_60_20_20/`, outside this repository.
Do not commit patient CSVs, virtual environments, images or model weights.
Training learns from train; validation chooses settings and confidence/referral
rules; test is reserved for final evaluation after the procedure is fixed.

## Environment and single-image check (C0a)

Run inside an allocated SLURM GPU job using the independent cluster environment:

```bash
~/comp3710-env/bin/python environment_probe.py \
  --split-dir ~/comp3710-data/adni/split_seed42_60_20_20 \
  --output-dir ~/comp3710-data/adni/c0a_environment_check_v2
```

Choose a new output directory; existing outputs are never overwritten. Run Python
without `-O` so integrity assertions are enabled. The script verifies the split
manifest hash, counts and patient separation, fully decodes the first training
JPEG, and checks a CUDA matrix multiplication. It saves an aggregate
`environment_check_summary.json`; it does not copy images or train a model.

The supplied C0a run on 2026-10-21 recorded Python 3.11.13, PyTorch
2.14.0+cu130 (CUDA build 13.0), Pillow 12.3.0 and an A100-PCIE-40GB.
The CUDA operation passed and one training image decoded as 256 x 240,
single-channel grayscale. Validation/test images were not read. These versions
are observations of the existing cluster environment, not a tested installation
recipe. The original installation command is unknown; a reproducible environment
specification remains to be documented.

The original evidence and image stay outside Git in
`final_report/adni_manifest_local/c0a_environment_check/`. Their recorded script
hash refers to the original probe. This committed copy adds CLI paths and guards;
it has not yet been rerun on the cluster GPU. Preserve historical evidence and
use a new output directory for subsequent runs.

## Evaluation and results

TODO: identical evaluation splits, per-class precision/recall, macro-F1,
AUROC, accuracy, confidence analysis, and resource comparison.

## Failure analysis and recommendation

TODO: analyze 3–5 real failure cases and summarize operational trade-offs.

## Artificial Intelligence Usage Disclosure

Codex assisted with repository setup, the initial scaffold, manifest and
scan-selection review, data/resource auditing, and local environment setup.
Validation included output-count and hash checks, an independent seed-42
selection recalculation, and ten synthetic-fixture tests. The image integrity
results were read from the provided cluster outputs; the local review did not
reopen the cluster images. Python 3.11.15 with Pillow 12.3.0 was configured in a
project-local virtual environment and all ten tests passed there. No model code
or model performance has yet been validated. Update this disclosure as work proceeds.

Additional Codex assistance covered patient-split verification (19 local tests
passed), review of the C0a cluster evidence, independent hash checks and decoding
of the provided local sample, and packaging the probe with configurable paths.
The packaged probe received local syntax/CLI checks; no new cluster GPU run was
performed during packaging. Dataset loading and training remain unimplemented.

## References

TODO: cite the course specification, model papers, and any adapted code.
