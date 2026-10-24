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

## Fixed-manifest grayscale dataset (C0b)

`ADNISliceDataset` reads only the new `split` column. The old `original_split`
column remains provenance and never controls membership. On construction it
checks `split_samples.csv` against the SHA-256 in `split_summary.json`, validates
labels and patient/image consistency, and rejects duplicate paths. No original
images are needed merely to construct a dataset or inspect its length.

```python
from dataset import ADNISliceDataset
train = ADNISliceDataset("/path/to/split_samples.csv", "train")
validation = ADNISliceDataset("/path/to/split_samples.csv", "validation")
test = ADNISliceDataset("/path/to/split_samples.csv", "test")
sample = train[0]  # Actual cluster images must be accessible at manifest paths.
```

Each item is a dictionary: `image` (CPU float32 tensor), `label` (CPU scalar
int64 tensor), `patient_id`, `image_id`, and `file_path`. Model labels are
**NC → 0, AD → 1**. The original metadata's **AD=2** is not the model label.
Read/decode errors raise an exception containing the source path and cause;
there is no silent skipping or substitution.

For the simple CNN, all splits use **H=240, W=256**, with shape `[1,240,256]`.
Pillow fully decodes the image, converts it to grayscale `L`, and uses bilinear
resizing only if dimensions differ. The uint8 pixels are converted to float32
and divided by 255, giving `[0,1]`. No mean/std standardization, data-dependent
normalization, or random augmentation is applied. This rule was chosen from
previously audited image dimensions before accessing test pixels. ConvNeXt's
input conversion remains a separate future decision.

The fixed manifest contains 8160/2720/2720 samples for train/validation/test.
C0b validates real train/validation samples on the cluster; test pixels remain
unread and only test-manifest membership/counts are checked. A few sampled
reads do not establish full-dataset decoding. Batch validation and model training
belong to later steps.

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
performed during packaging. Dataset loading and the batch-check script are implemented; C0c real-batch validation is complete; model training remains pending.

C0b review verified dataset/split hashes and actual manifest lengths, ran all
26 local tests (including seven dataset tests), checked dependency consistency,
and loaded the existing real-image copy through the dataset using a temporary
local manifest. Provided cluster evidence records four unique train/validation
images and no test pixels; the reviewer did not rerun these cluster reads.
Preprocessing is fixed to grayscale float32 [1,240,256] scaled to [0,1].

C0c assistance reviewed the existing batch checker and added regression tests before fixing frozen-cohort count validation and failure-summary persistence. All 34 local synthetic tests passed. No real cluster images were read in this C0c implementation session.

## References

TODO: cite the course specification, model papers, and any adapted code.

## C0c: batch integrity check

Run this on Rangpur, where the manifest's image paths are accessible:

```bash
python check_dataloader.py \
  --manifest ~/comp3710-data/adni/split_seed42_60_20_20/split_samples.csv \
  --summary ~/comp3710-data/adni/split_seed42_60_20_20/split_summary.json \
  --output-dir ~/comp3710-data/adni/c0c_loader_check \
  --batch-size 4 --seed 42
```

This CPU check uses the existing ADNISliceDataset without training. Training
shuffles with a private seeded PyTorch generator; validation is sequential.
Both loaders use num_workers=0 and drop_last=False. Images are float32
[4,1,240,256] in [0,1]; int64 labels are [4], with NC=0 and AD=1.
Each collated position is checked against the manifest and its pre-collation
image tensor. The validation first batch is read again using a fresh iterator
and must match exactly. Same-patient slices and single-class batches are valid.

The default run reads 12 items (4 training and 8 validation), covering 8 unique
images. Test image reads are zero; the full manifest is checked for patient
separation. This is a batch spot check, not a full image audit. Results and
actual logs are written to loader_check_summary.json and loader_check.log
outside the repository. Existing evidence is never overwritten.

Run all synthetic tests with `.venv/bin/python -m unittest discover -s tests -v`.
Tests also verify that a final incomplete batch is retained and that metadata
or pixel misalignment fails explicitly.

### C0c cluster procedure and evidence

Sync the updated `check_dataloader.py`, `dataset.py`, tests and README to your
existing cluster project directory. Do not copy the macOS `.venv`.
On the login node, enter that project directory and activate the same cluster
Python environment used for C0b. Both locations are your existing paths; the
examples below assume that this has already been done.

Request a short compute session using the previously verified course setup:

```bash
srun --account=comp3710 --partition=a100-test --gres=gpu:1 --time=00:10:00 --pty bash
```

The checker itself uses CPU tensors. This session reuses the known course
execution route; it does not measure GPU training memory or speed. If the
shell changes environment or directory, reactivate your cluster environment
and return to the directory containing `dataset.py` and `check_dataloader.py`.

```bash
python -c 'import sys, torch, PIL; print(sys.executable); print(torch.__version__, PIL.__version__)'
C0C_RUN_DIR="$HOME/comp3710-data/adni/c0c_loader_check_$(date +%Y%m%d_%H%M%S)"
mkdir -p "$C0C_RUN_DIR"
python -m unittest discover -s tests -v > "$C0C_RUN_DIR/test_results.txt" 2>&1
cat "$C0C_RUN_DIR/test_results.txt"
```

Continue only if the tests end with `OK`. Then run:

```bash
python check_dataloader.py   --manifest "$HOME/comp3710-data/adni/split_seed42_60_20_20/split_samples.csv"   --summary "$HOME/comp3710-data/adni/split_seed42_60_20_20/split_summary.json"   --output-dir "$C0C_RUN_DIR" --batch-size 4 --seed 42
cat "$C0C_RUN_DIR/loader_check_summary.json"
printf '%s\n' "$C0C_RUN_DIR"
```

The input location follows the earlier project convention. If your actual
frozen files are elsewhere, change only those two input paths. Do not rerun
patient splitting. Each attempt uses a new output directory; the script refuses
to overwrite existing check logs or summaries. The CLI requires the exact frozen
8160/2720/2720 cohort lengths before reading images. A failed check exits nonzero
and saves failure evidence (provided the output directory is writable).

Send back `loader_check_summary.json`, `loader_check.log`, and
`test_results.txt`, plus the cluster project location and job ID (if available).
Copy these to the local `final_report/adni_manifest_local/c0c_loader_check/`
folder, with separate subfolders for repeat attempts. No MRI images or patient
CSVs need to be sent. Failure logs may contain identifying paths: retain those
privately, never commit them to the public repository. A default successful run
reports 12 read calls, 8 unique images, zero test reads, zero errors, correct
shapes/dtypes, and identical validation batches. C0c real-data acceptance is recorded below; no model training is performed.

### C0c acceptance (24 October 2026)

Saved cluster evidence for job 690651 reports COMPLETED, exit 0:0. Both batch
shapes are [4,1,240,256], float32 in [0,1], with [4] int64 labels and matching
metadata. Validation repetition is identical; 12 reads cover 8 distinct images,
with zero test reads or errors. Training contains NC=3/AD=1; validation NC=4/AD=0,
which is valid for a small batch. Both frozen input hashes and the dataset.py
hash match local files; cohort lengths were independently checked.

Version scope: the cluster ran the earlier checker (SHA-256
4cf2a05250f63cbca9910e8848129479fb44c229e6a381b7df7b479a94f00e04),
with 6 batch tests passing; the supplied earlier full-suite log shows 32 tests.
Reconstructing that source from the current version reproduces this exact hash:
the only changes are strict CLI cohort-count validation and saving input-hash
failures in the summary. The current version passed all 34 local tests, including
those two regression cases. The current complete CLI was not rerun on the cluster.
Use the newly committed version for subsequent deployments. Saved cluster
provenance remains unchanged (d2732da plus uncommitted changes).

Codex audited the saved evidence, hashes, source difference and manifest counts,
and reran local tests; it did not rerun remote image reads. This acceptance covers
sampled batch loading only, not full-image decoding or model/GPU training.

## Simple CNN smoke check (C1)

`modules.py` now defines SimpleCNN: Conv2d/ReLU/MaxPool blocks with channels
1→16→32→64, adaptive average pooling, then a 64→2 classifier (23,426 parameters).
It returns raw logits, ordered NC=0 and AD=1. CrossEntropyLoss consumes logits
and int64 labels directly; do not apply softmax before this loss.

`check_cnn.py` runs two Adam updates (lr=0.001, batch=4, seed=42), then one
validation batch under eval/no_grad. It checks finite outputs/loss/gradients,
nonzero gradients, changed finite parameters and checkpoint prediction agreement.
Validation does not update parameters. No accuracy, convergence or monotonic-loss
claim is made. `train.py` remains a scaffold: this is not formal training.

Sync modules.py, check_cnn.py, check_dataloader.py, dataset.py and all tests to
the existing cluster project directory. Use the previous C0b/C0c cluster Python
environment, not the Mac .venv. On the login node request a compute session:

```bash
srun --account=comp3710 --partition=a100-test --gres=gpu:1 --time=00:10:00 --pty bash
```

Inside that session, activate your existing cluster Python environment, then:

```bash
cd /home/Student/s4982104/comp3710-code/ConvNeXt_ADNI_s4982104
CNN_RUN_DIR="$HOME/comp3710-data/adni/cnn_smoke_check_$(date +%Y%m%d_%H%M%S)"
mkdir -p "$CNN_RUN_DIR"
python -m unittest discover -s tests -v > "$CNN_RUN_DIR/test_results.txt" 2>&1
cat "$CNN_RUN_DIR/test_results.txt"
```

Continue only when the full suite ends with `OK` (currently 41 tests).

```bash
python check_cnn.py \
  --manifest "$HOME/comp3710-data/adni/split_seed42_60_20_20/split_samples.csv" \
  --summary "$HOME/comp3710-data/adni/split_seed42_60_20_20/split_summary.json" \
  --output-dir "$CNN_RUN_DIR" \
  --device cuda --batch-size 4 --steps 2 --seed 42
cat "$CNN_RUN_DIR/cnn_check_summary.json"
printf '%s\n' "$CNN_RUN_DIR"
printf '%s\n' "${SLURM_JOB_ID:-unavailable}" > "$CNN_RUN_DIR/job_id.txt"
```

Default real-data counts are train=8, validation=4, test=0, unique_images=12.
The script fails if CUDA was requested but is unavailable; CPU fallback must
be explicitly selected with --device cpu and does not satisfy the GPU check.
Outputs are cnn_check_summary.json, cnn_check.log and smoke_checkpoint.pt.
The checkpoint is only for the smoke round-trip check, not a trained deliverable.
Do not commit it or send it for routine acceptance. Repeated runs require a new
output directory so that earlier evidence is retained.

Send back summary, log, test_results.txt and job_id.txt; add saved Slurm status
if available. Store local copies under final_report/adni_manifest_local/cnn_smoke_check/.
The summary records code/input hashes, environment, parameters, each step loss,
gradient norm, updates, validation loss, reload agreement, reads, synchronized
step times and peak CUDA allocated/reserved bytes. Timing includes diagnostics
and excludes loading/transfers; memory is captured before loading a second model.
These are smoke-run measurements, not formal throughput or full-run estimates.
A non-Git cluster directory is recorded honestly; source file hashes identify
this run without rewriting old provenance.

Local validation: 41 synthetic tests passed, including 7 CNN tests covering
batch-size one/four, gradients/updates, invalid input/labels, checkpoint reload,
end-to-end loading with an incomplete final batch and nonexistent test images,
and CLI failure evidence. Codex implemented this CNN/checker and tested locally;
The user ran the real MRI/CUDA check on the cluster; Codex audited the returned
evidence and reran the local suite. Formal training remains pending.


### C1 acceptance — 24 October 2026

Job 690695 completed on A100-PCIE-40GB with exit 0:0 (elapsed 1m43s).
The CUDA summary reports passed, zero errors and 23,426 model parameters.
Both Adam steps updated parameters with finite losses (0.698846, 0.685690)
and nonzero finite gradient norms (0.452714, 0.437852). Validation loss was
0.652301 with gradients disabled; checkpoint predictions agreed after reload.
Reads were train=8, validation=4, test=0, unique_images=12.
All four source hashes and both frozen-input hashes match the local files.
The cluster full suite passed 41 tests; the acceptance rerun locally also passed 41.

Peak CUDA allocated memory was 73.06 MiB and reserved memory 96 MiB.
Diagnostic step times were 4.97152s and 0.00534s; these include checks and exclude
loading/transfers, so they do not establish formal latency or a training budget.
Private evidence is retained in final_report/adni_manifest_local/cnn_smoke_check/.
C1 is accepted; next implement the epoch training/validation loop, validation-based
checkpoint selection and reproducible logs, then run a short pilot. Keep the test
set untouched during this development. No accuracy target has yet been evaluated.
