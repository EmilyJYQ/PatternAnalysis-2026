# ADNI AD/CN Classification: CNN Baseline and ConvNeXt

COMP3710 individual project — s4982104.

## Status

Patient-disjoint splits, data loading and the SimpleCNN C1 GPU check are complete.
C2 training/validation, checkpoint selection and logging are implemented and pass
49 local tests; the three-epoch cluster pilot passed C2 engineering acceptance on 25 October 2026. ConvNeXt and final
evaluation remain pending. C3 validation inference and patient aggregation now pass
55 local tests and passed cluster acceptance on 25 October 2026. No final accuracy results are claimed.

## Project objective

Compare a simple CNN baseline with ConvNeXt for AD versus cognitively normal
classification. Evaluate accuracy, confidence reliability, referral of uncertain
cases for human review, GPU memory, and inference time.

## Data

Course data location: `/home/groups/comp3710/ADNI` on Rangpur.
The initial read-only audit found 30,520 JPEG slices, 1,526 image IDs, and
680 patients. The supplied train/test folders share 216 patients. The frozen replacement split selects one scan per patient
and separates patients into 60/20/20 train/validation/test groups.
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
claim is made. `train.py` now implements the separate C2 pilot described below; C1 itself is not formal training.

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
C1 is accepted. The C2 epoch loop, checkpoint selection and logs are implemented
and locally tested; the real-data C2 pilot passed engineering acceptance; see the results below. Keep the test
set untouched during this development. No accuracy target has yet been evaluated.


## C2: full-epoch CNN pilot (accepted 25 October 2026)

`train.py` trains SimpleCNN on the complete fixed training split and validates on
all validation slices every epoch. Defaults: 3 epochs, batch 16, Adam lr 0.001,
seed 42, no augmentation, weighting, scheduler or early stopping. Train shuffles
using a seeded generator; validation is sequential. Both keep incomplete batches.
The pilot is deliberately bounded to three epochs; it is not the final training
schedule. CLI enforces split lengths 8160/2720/2720. Test metadata is validated;
no test loader is created and no test pixels are read.

The best checkpoint minimizes validation cross-entropy; only strict improvement
replaces it (first epoch wins ties). Loss is sample-weighted, including the final
incomplete batch. Logs report **slice-level** accuracy, per-class precision/recall/F1,
macro-F1, confusion matrix (true rows/predicted columns, NC then AD), and AD AUROC.
Prediction uses P(AD)>=0.5. Undefined precision/recall/F1 use zero; AUROC is null
when only one class is present. These metrics do not constitute patient-level
results. Training metrics aggregate predictions made during optimization; validation
metrics use the final model for that epoch. No 80% threshold is used to pass C2.

Deterministic Torch algorithms and seeded shuffling are enabled; the runner sets
CUBLAS_WORKSPACE_CONFIG. Same-environment CPU reproducibility is tested; equality
across PyTorch versions or CPU/GPU is not assumed. Image data are not cached.
Read counts are collected with num_workers=0. Validation uses eval/no_grad and
checks all model state is unchanged; each training epoch must update parameters.
Best weights are reloaded and compared using a fixed synthetic probe. This does
not yet implement the separate `predict.py` deliverable.

### Submit the three-epoch pilot

Sync train.py, run_c2.slurm, modules.py, dataset.py, check_dataloader.py,
check_cnn.py and the complete tests directory to the existing cluster project.
Activate the same cluster environment used for C1, then on the login node:

```bash
cd /home/Student/s4982104/comp3710-code/ConvNeXt_ADNI_s4982104
python -c 'import sys; print(sys.executable)'
sbatch run_c2.slurm "$(python -c 'import sys; print(sys.executable)')"
```

The runner requests one GPU in comp3710 with account comp3710 and a one-hour
wall-time ceiling (not a predicted runtime). It runs all tests first and stops
on failure, then runs the pilot. It passes an absolute Python interpreter path;
do not upload or use the Mac .venv. Keep any environment modules required by
your existing cluster Python loaded before submission.

Replace JOB_ID below with the job number returned by sbatch:

```bash
squeue -j JOB_ID
sacct -j JOB_ID --format=JobID,State,ExitCode,Elapsed,MaxRSS   > "$HOME/comp3710-data/adni/c2_pilot_JOB_ID/slurm_status_JOB_ID.txt"
```

Run sacct after completion. Outputs:

```text
$HOME/comp3710-data/adni/c2_pilot_JOB_ID/
  job_id.txt
  test_results.txt
  slurm_status_JOB_ID.txt
  training/
    config.json
    epochs.jsonl
    training.log
    training_summary.json
    best.pt
```

The project directory also receives c2-slurm-JOB_ID.out; return this scheduler log.
A new output directory is mandatory. JSON/JSONL record configuration, source/input
hashes, environment, per-epoch metrics and timing, peak CUDA memory (before creating
the reload model), selected epoch, checkpoint hash and reload agreement. Each epoch
time includes data loading, transfers, optimization, validation and checks, but
excludes checkpoint writes. Overall elapsed time includes setup and checkpoint
verification. These measurements are not inference-latency benchmarks. Python
exceptions save a failed summary and traceback; scheduler termination can leave
only partial logs, which must also be retained.

### Return for acceptance

Copy the evidence directory **excluding best.pt** to
final_report/adni_manifest_local/c2_training_pilot/ and include the scheduler log.
Return seven evidence files plus the scheduler log: job_id.txt, test_results.txt,
slurm_status_JOB_ID.txt and training/{config.json,epochs.jsonl,training.log,
training_summary.json}. Keep best.pt privately on the cluster.

Acceptance: COMPLETED/0:0; 49 tests pass; summary passed/error_count=0 on CUDA;
three complete epochs with 8160 train and 2720 validation samples each; cumulative
reads train=24480, validation=8160, test=0; finite loss/metrics; training parameters
changed and validation unchanged; best_epoch matches minimum validation loss;
checkpoint reload agrees; four code and two input hashes match the audited files.
Review actual elapsed time and peak VRAM against the project budget/20GB target
before planning longer runs. Neither falling loss nor 80% accuracy is required
for this engineering acceptance. Unexpected errors are investigated before reruns.

AI disclosure (C2): Codex implemented the loop, metrics, runner and eight tests,
and updated the plan. All 49 local synthetic tests passed (including the existing
41); shell syntax and diff checks passed. Real MRI/CUDA C2 execution and acceptance
are pending. No new commit was made in this implementation turn.


### C2 cluster acceptance — 25 October 2026

Job 694112 completed (0:0) on A100-PCIE-40GB, wall time 1m07s. The training
program took 39.62s; full-epoch times were 14.37/11.99/11.31s. All three epochs
covered 8160 train and 2720 validation slices; cumulative reads were
24480/8160/0 for train/validation/test. Parameters changed during training and
remained unchanged during validation. The lowest validation loss selected epoch 3;
checkpoint reload agreed according to the cluster summary. Four code and two
input hashes match local files. Config and JSONL history match the summary.
Confusion-derived metrics were independently recomputed; individual predictions
and the private checkpoint were not returned, so loss/AUROC/reload verification
uses the saved cluster evidence. Cluster tests: 49 passed (19.674s); fresh local
acceptance tests: 49 passed (6.627s).

| Epoch | Validation loss | Slice accuracy | Macro-F1 | AD AUROC |
| --- | --- | --- | --- | --- |
| 1 | 0.616887 | 67.647% | 0.403509 | 0.625416 |
| 2 | 0.614117 | 67.574% | 0.403247 | 0.629774 |
| 3 | 0.611294 | 67.574% | 0.405419 | 0.631629 |

Best-epoch confusion matrix (NC/AD): [[1836,4],[878,2]]. AD recall is only
2/880 = 0.227%; always predicting NC gives 67.647% accuracy. C2 passes as a
pipeline check, not as a useful classifier or the 80% performance target.
Peak CUDA allocated/reserved memory was 277.51/518 MiB, below the 20GB target;
these are pilot measurements, not inference latency or a guarantee for other models.

The first submission was rejected for --mem=8G. The returned submission notes
report the partition's memory configuration and successful resubmission using
--mem=0. The runner now uses --mem=0 to reproduce that successful request
(all configured node memory, not zero memory). Only this submission directive
changed after the run; training-source hashes remain unchanged.

Evidence: final_report/adni_manifest_local/c2_training_pilot/, including original
submission notes. Codex audited returned evidence and reran local tests, but did
not execute remote training or retrieve patient images/weights. Next implement
C3 checkpoint-based prediction on validation data, including slice-to-patient
probability aggregation and diagnostic outputs; then plan the longer baseline
run and controlled experiments addressing low AD recall. Keep the test set sealed.


## C3: validation predictions and patient aggregation (accepted 25 October 2026)

`predict.py` loads the C2 best checkpoint and predicts **validation only**.
It verifies the checkpoint hash against training_summary.json, the selected epoch,
loss/configuration, input hashes and model/preprocessing source hashes. The CLI
requires the frozen 8160/2720/2720 cohort, exactly 20 slices per patient and 136
validation patients. Each patient must have one scan and one label; paths must be
unique. There is no split or threshold tuning option.

For each of 2720 slices, save P(AD) and the predicted class. For each of 136
patients, average the 20 probabilities and predict AD when the mean is >=0.5.
This is probability averaging, not majority voting. Report metrics separately
for slices and patients using the existing NC=0/AD=1 definitions and zero-division
policy. Patient-level results are validation results, not unseen-test estimates;
this cohort already selected the checkpoint.

Prediction uses eval/no_grad, checks model state is unchanged, and loads the same
checkpoint state into a new model for a second full pass. Identity/order and
classes must agree; probabilities must match within rtol=1e-5, atol=1e-6.
Only first-pass predictions are exported. Expected reads: train=0,
validation=5440, test=0. Peak memory and elapsed time include repeat checking,
so this is not a formal inference-latency benchmark. No training occurs.

### Cluster run

Sync predict.py, run_c3.slurm and the complete tests directory, and ensure train.py,
modules.py, dataset.py, check_dataloader.py and check_cnn.py are current. Preserve
the C2 best.pt and training_summary.json. Activate the C1/C2 cluster environment,
then submit from the login node:

```bash
cd /home/Student/s4982104/comp3710-code/ConvNeXt_ADNI_s4982104
sbatch run_c3.slurm "$(python -c 'import sys; print(sys.executable)')"
```

Default checkpoint source is
$HOME/comp3710-data/adni/c2_pilot_694112/training/. If relocated, pass that training
directory as the second script argument. The runner uses comp3710/account comp3710,
one GPU, --mem=0 and a 20-minute wall-time ceiling; it runs 55 tests before
prediction. Reuse the existing cluster environment, not the Mac .venv.

After completion replace JOB_ID and save scheduler status:

```bash
sacct -j JOB_ID --format=JobID,State,ExitCode,Elapsed,MaxRSS \
  > "$HOME/comp3710-data/adni/c3_validation_JOB_ID/slurm_status_JOB_ID.txt"
```

### Return for acceptance

Copy to local final_report/adni_manifest_local/c3_validation/:

```text
job_id.txt
slurm_status_JOB_ID.txt
test_results.txt
c3-slurm-JOB_ID.out
prediction/
  prediction_summary.json
  prediction.log
  slice_predictions.csv
  patient_predictions.csv
```

All except c3-slurm-JOB_ID.out are under
$HOME/comp3710-data/adni/c3_validation_JOB_ID/; the scheduler log is in the project
directory. The CSVs contain research IDs and paths: retain them as private local
acceptance evidence, do not commit or publish. No image or weight transfer is needed.
A new output directory is required; Python exceptions save a failed summary/log.

Acceptance: COMPLETED/0:0; 55 tests pass; summary passed with zero errors on CUDA;
2720 unique slice records and 136 patients (NC92, AD44), 20 slices each; correct
mean probabilities and threshold; metric recomputation agrees; unchanged model,
repeat predictions agree, reads 0/5440/0. Match five source hashes, frozen input
hashes, C2 training-summary and checkpoint hashes, and both exported CSV hashes.
Compare slice metrics with C2 epoch 3 (allowing numerical tolerance); any unexpected
difference is investigated. No accuracy floor is imposed for C3 engineering acceptance.

AI disclosure: Codex implemented C3 and six tests, including mean aggregation,
threshold ties, invalid groups, hash/provenance rejection, no test-image access,
repeat inference and output protection. Tests first failed on the missing API;
after implementation all 55 local tests passed. No real MRI/CUDA C3 run was
performed locally. C3 remains pending cluster acceptance; no commit in this turn.


### C3 acceptance — 25 October 2026

Job 694432 completed (0:0) on A100-PCIE-40GB in 34 seconds. Prediction including
two passes and checks took 7.32s; peak CUDA allocated/reserved was 155.93/202 MiB.
The returned cluster evidence reports unchanged parameters, repeat agreement and
maximum probability difference 0. Reads: train=0, validation=5440, test=0.

Independent local acceptance matched all 2720 unique slice paths and labels to
the frozen validation manifest, verified 136 unique patients (NC92/AD44), each
with exactly 20 slices from one scan, and recalculated every patient mean and
threshold decision. All per-class metrics, accuracy and macro-F1 were recomputed;
AUROC was independently calculated by positive-negative probability comparisons.
Ten local file hashes match (five code, two inputs, two CSVs, C2 summary); the
checkpoint hash agrees with the accepted C2 record. The checkpoint itself remains
on the cluster. Repeat/unchanged-state claims use the saved run evidence.

| Level | Accuracy | Macro-F1 | AD AUROC | AD recall |
| --- | --- | --- | --- | --- |
| Slice | 67.574% | 0.405419 | 0.631629 | 0.227% |
| Patient | 67.647% | 0.403509 | 0.689476 | 0% |

Slice metrics exactly match C2 epoch 3. Patient confusion matrix is [[92,0],[44,0]]:
all patients are predicted NC at threshold 0.5. This passes inference/aggregation
engineering checks, not classifier performance requirements. These are validation
results from the cohort used for checkpoint selection, not final test results.

Cluster: 55 tests passed (19.684s); fresh local acceptance: 55 passed (6.773s).
Shell syntax and diff checks passed. Private evidence remains under
final_report/adni_manifest_local/c3_validation/ and is excluded from Git.
Codex independently audited saved outputs and reran local tests; it did not rerun
remote predictions. Next plan a longer CNN baseline and controlled validation-only
experiments for low AD recall, retaining the existing split, C2/C3 evidence and
sealed test set. No additional training or threshold tuning was done in acceptance.


## E1 weighting and paired evidence; E2 one-epoch compatibility

E1 is implemented and passes 61 local tests (the previous 55 plus six new tests).
E2 real-data/CUDA compatibility passed on 25 October 2026; see acceptance below.

`train.py --class-weighting none|balanced` defaults to none. Balanced weights are
computed exclusively from unique training patients: N/(2*N_class), in [NC,AD]
order. The real split gives [408/550,408/266] = [0.74181818,1.53383459]. Training
must contain both classes and equal slice counts per patient for balanced mode.
The validation loss is always unweighted and selects the best checkpoint with
strict improvement (earlier epoch wins ties). No weighted sampler, augmentation,
early stopping, learning-rate schedule or threshold change is introduced.

Each epoch logs training loss numerator/denominator and unweighted_loss. For the
weighted objective, the denominator is the sum of target-class weights, including
the final partial batch. Unweighted mode retains the original forward/backward
calculation, verified against the old algorithm's loss and parameter updates.
Do not compare the weighted objective with A's ordinary loss as if they were the
same scale; use the shared unweighted diagnostic and validation metrics.

Config records class weights and patient/slice counts. The summary stores an
initial-state SHA-256; each epoch stores a SHA-256 of the ordered batch paths
including batch boundaries. Same-seed A/B must match both hashes. These hashes
record initialization and sample order, not equality of learned model weights.

After epoch 3, `first3/best.pt` freezes the best checkpoint among epochs 1–3,
not necessarily epoch 3. `first3/training_summary.json` preserves the three-epoch
history, corresponding best epoch/loss, input/source/config and checkpoint hashes,
plus reload verification. Its config retains the parent run's planned epoch count;
completed_epochs=3 and scope explicitly identify the prefix. Later improvements
only replace the parent best.pt. The C3 predictor accepts this checkpoint/summary
pair directly. For fewer than 3 epochs, first3_snapshot.status is not_reached and
no first3 directory is created. A failed parent run may retain a valid three-epoch
prefix; this never implies the full parent run passed.

### Run E2, not the 30-epoch experiment yet

Sync train.py, run_e2.py, run_e2.slurm, the entire tests directory and the current
predict.py, modules.py, dataset.py, check_dataloader.py, check_cnn.py. Activate the
same cluster Python environment, then on the login node:

```bash
cd /home/Student/s4982104/comp3710-code/ConvNeXt_ADNI_s4982104
sbatch run_e2.slurm "$(python -c 'import sys; print(sys.executable)')"
```

One job uses account/partition comp3710, one GPU, --mem=0 and a 20-minute ceiling.
After all 61 tests pass, it runs A=none for one epoch and its two-pass C3 prediction,
then B=balanced for one epoch and prediction. Both start fresh with seed42,
batch16 and Adam lr0.001. run_e2.py checks paired config (excluding weighting),
initialization, training order, source/inputs, unweighted validation and checkpoint
provenance, then saves e2_summary.json. Failures stop the job and retain available
evidence. Nothing overwrites C2/C3 outputs.

Outputs live under $HOME/comp3710-data/adni/e2_compatibility_JOB_ID/.
After completion replace JOB_ID and save:

```bash
sacct -j JOB_ID --format=JobID,State,ExitCode,Elapsed,MaxRSS \
  > "$HOME/comp3710-data/adni/e2_compatibility_JOB_ID/slurm_status_JOB_ID.txt"
```

Return the following tree to local final_report/adni_manifest_local/e2_compatibility/:

```text
job_id.txt
test_results.txt
slurm_status_JOB_ID.txt
e2-slurm-JOB_ID.out
pair/
  e2_summary.json
  A/
    training/{config.json,training_summary.json,epochs.jsonl,training.log}
    prediction/{prediction_summary.json,prediction.log,slice_predictions.csv,patient_predictions.csv}
  B/
    training/{config.json,training_summary.json,epochs.jsonl,training.log}
    prediction/{prediction_summary.json,prediction.log,slice_predictions.csv,patient_predictions.csv}
```

The scheduler log is in the cluster project directory. Keep all .pt files on the
cluster; CSVs remain private local evidence, not Git content. Include e2_error.log
and partial outputs if a step fails. There is intentionally no first3 snapshot in
a one-epoch E2 run: the snapshot logic is covered by the local four-epoch fixture,
and real-data snapshot evidence will be checked during E3.

Acceptance: job COMPLETED/0:0, 61 tests pass; both training and prediction summaries
and the pair summary pass. Each training arm reads 8160/2720/0 train/validation/test;
each prediction reads 0/5440/0 and exports 2720 slices/136 patients. Across both arms,
reads are train=16320, validation=16320, test=0. Verify NC275/AD133 training counts,
correct weights, equal initial-state/order hashes, unweighted validation, expected
best epoch 1, consistent source/input/checkpoint/output hashes and repeat predictions.
Low recall does not fail E2 compatibility and one epoch does not establish benefit.

AI disclosure: Codex implemented weighting, paired evidence, prefix snapshot,
run_e2.py/run_e2.slurm and six tests. New tests initially failed for the missing API;
all 61 local tests passed after implementation (8.835s). Tests cover the exact
weighted denominator and gradients, default-mode equivalence, paired initialization/
order, immutable first3 best selection with later improvement, C3 prefix reload,
and tampered pair rejection. Shell syntax/diff checks passed. No remote run or
commit was performed in this implementation turn.


### E2 acceptance — 25 October 2026

Job 694577 completed (0:0) on CUDA in 1m22s. Cluster tests: 61 passed (31.882s);
fresh local acceptance: 61 passed (8.654s). Shell syntax and diff checks passed.
Both arms trained one epoch and passed C3 validation prediction. Independent audit
matched runner/source/input/output and summary hashes, paired configuration,
initialization and ordered batch hashes. A has no weights; B uses
[0.7418181818181818,1.5338345864661653] from NC275/AD133 training patients.
Validation remains unweighted. Best epoch=1 in each arm; first3 is correctly
not_reached. Its real-data verification remains for E3, not this one-epoch run.

Each arm read train/validation/test = 8160/2720/0 during training, then 0/5440/0
for prediction. Combined totals: 16320/16320/0. Each exported 2720 slices and
136 patients. Independent recalculation confirmed exact validation-manifest
coverage, 20 slices/one scan/consistent label per patient, probability means,
threshold decisions, per-class metrics, accuracy, macro-F1 and pairwise AUROC.
Prediction slice metrics match the corresponding training validation metrics.
Both repeat passes report zero probability differences and unchanged model state.
Checkpoint hashes agree between source and prediction summaries; weights remain
on the cluster and were not locally reloaded during this acceptance.

| Patient-level metric | A: none | B: balanced |
| --- | --- | --- |
| Accuracy | 67.647% | 67.647% |
| AD recall | 0/44 = 0% | 21/44 = 47.727% |
| NC specificity | 92/92 = 100% | 71/92 = 77.174% |
| AD precision | 0 (zero-division policy) | 50% |
| Macro-F1 | 0.403509 | 0.625906 |
| AUROC | 0.685524 | 0.679101 |

Confusion matrices: A [[92,0],[44,0]], B [[71,21],[23,21]]. B identifies 21 AD
patients but also misclassifies 21 NC patients. This is an early tradeoff, not
proof that weighting is superior: one epoch/one seed does not settle the planned
30-epoch paired comparison. No threshold or protocol was changed after inspection.
A/B training elapsed was 17.91/12.79s, prediction 5.08/5.15s; allocated memory
277.42 MiB each, reserved 518 MiB each. These include checks, not formal latency.

Evidence is private under final_report/adni_manifest_local/e2_compatibility/.
Codex audited saved evidence and ran local tests, without remote retraining or
modifying historical outputs. Next E3: A42, 30 epochs from scratch, retain first3
snapshot, then predict both first3 and full-run best checkpoints before E4/B42.


## E3 A42: 30 epochs and automatic first3/full-best evaluation

E3 orchestration and the real A42 cluster run passed acceptance on 25 October 2026. Existing train.py and predict.py are reused unchanged. run_e3.py fixes
seed42, unweighted loss, 30 epochs, batch16, Adam lr0.001; no early stopping,
augmentation, schedule, threshold tuning or test-set evaluation. Training starts
from scratch. The public CLI does not expose protocol changes.

After training, the runner evaluates training/first3/best.pt with its own snapshot
summary, then training/best.pt with the full-run summary. Both are minimum
unweighted-validation-loss checkpoints (earlier epoch wins ties), not automatically
the weights at epochs 3 and 30. It checks prefix history/config/provenance against
the full run, recomputes best epochs, verifies physical checkpoint hashes, runs C3
repeat prediction for both, and writes comparison.csv and e3_summary.json.
The comparison contains patient-level accuracy, AD recall/precision, NC specificity,
macro-F1, balanced accuracy and AUROC. Full slice metrics stay in prediction summaries.
Identical first3/full best checkpoints are valid and are recorded as same_checkpoint=true.
Improvement is not an engineering acceptance requirement.

### Submit E3

Sync run_e3.py, run_e3.slurm, all tests, and the current train.py, predict.py,
modules.py, dataset.py, check_dataloader.py, check_cnn.py and run_e2.py. Activate
the existing cluster Python environment, then on the login node:

```bash
cd /home/Student/s4982104/comp3710-code/ConvNeXt_ADNI_s4982104
sbatch run_e3.slurm "$(python -c 'import sys; print(sys.executable)')"
```

This single job runs all 65 tests, then training and both prediction evaluations.
No separate C3 job is needed. It requests one GPU on comp3710, account comp3710,
--mem=0, a one-hour ceiling (not estimated duration). The earlier per-epoch timing
suggests several minutes, but actual Slurm usage determines the budget.
Evidence directory: $HOME/comp3710-data/adni/e3_A42_JOB_ID/.
After completion replace JOB_ID and save:

```bash
sacct -j JOB_ID --format=JobID,State,ExitCode,Elapsed,MaxRSS \
  > "$HOME/comp3710-data/adni/e3_A42_JOB_ID/slurm_status_JOB_ID.txt"
```

Copy the following tree, excluding all .pt files, to local
final_report/adni_manifest_local/e3_A42/:

```text
job_id.txt
test_results.txt
slurm_status_JOB_ID.txt
e3-slurm-JOB_ID.out
run/
  e3_summary.json
  comparison.csv
  training/
    config.json
    training_summary.json
    epochs.jsonl
    training.log
    first3/training_summary.json
  prediction_first3/
    prediction_summary.json
    prediction.log
    slice_predictions.csv
    patient_predictions.csv
  prediction_full/
    prediction_summary.json
    prediction.log
    slice_predictions.csv
    patient_predictions.csv
```

The scheduler log is in the project directory. Keep training/best.pt and
training/first3/best.pt on the cluster. CSVs stay private and are ignored by Git.
On failure return e3_error.log, scheduler output and partial evidence too; do not
overwrite the failed run. Python exceptions save failed summaries; scheduler kills
may leave partial files.

Acceptance: 65 tests pass, COMPLETED/0:0, CUDA and summary passed. Full 30 epochs
cover 8160 train/2720 validation slices each; training counts 244800/81600/0.
Each prediction reads 0/5440/0 and exports 2720 slices/136 patients (20 each).
Total actual reads are train=244800, validation=92480, test=0; first3 is a prefix
and is not counted again. Verify prefix history exactly equals full history[:3],
best epoch selection, checkpoint/summary/input/source/output hashes, repeat checks,
comparison metrics and resource use. Compare the first three training epochs with
C2/E2 as a reproducibility diagnostic, allowing stated environment/numerical differences.
A valid outcome may show no benefit or worse patient recall after longer training;
retain the evidence and continue the predefined E4/B42 comparison without tuning
this A42 run after inspecting its outcomes.

AI disclosure: Codex added the E3 runner, Slurm wrapper and four tests. New tests
first failed on the missing runner; after implementation and correcting an
inconsistent mock loss field, all 65 local tests passed (9.429s). Tests cover
four-epoch synthetic end-to-end evaluation, exact read accounting, tampered best
epoch rejection, saved failures and the case where both best checkpoints are the
same. Shell syntax and diff checks passed. No real 30-epoch/CUDA E3 run was performed;
no commit was made in this preparation turn.


### E3 acceptance — 25 October 2026

Job 694924 completed 30 epochs on CUDA (0:0), Slurm wall time 6m24s
(0.1067 GPU-hours). Training took 338.91s, the E3 training/evaluation sequence
348.97s; peak CUDA allocated/reserved was 277.51/518 MiB. Cluster tests: 65 passed
(29.259s); fresh local acceptance: 65 passed (9.761s). Shell syntax/diff checks passed.

Independent audit matched runner/source/input/summary/output hashes, all 30 epoch
records, the first3 prefix and comparison.csv. Both predictions exactly cover
2720 validation slices and 136 patients with 20 slices each. Patient means,
threshold decisions, all confusion-derived metrics and AUROC (via independent
positive-negative probability comparisons) were recomputed. Actual reads total
244800/92480/0 train/validation/test. Prefix losses and metrics exactly reproduce
C2's first three epochs. Repeat inference and unchanged parameters pass according
to saved cluster evidence. Checkpoint hashes link training and predictions; weights
remain on the cluster, so local acceptance did not reread their bytes.

| Patient metric | Best within first 3 | Best within all 30 |
| --- | --- | --- |
| Selected epoch | 3 | 29 |
| Unweighted validation loss | 0.611294 | 0.509784 |
| Accuracy | 67.647% | 72.059% |
| AD recall | 0/44 = 0% | 20/44 = 45.455% |
| NC specificity | 92/92 = 100% | 78/92 = 84.783% |
| AD precision | 0 (zero-division policy) | 58.824% |
| Macro-F1 | 0.403509 | 0.658472 |
| Balanced accuracy | 50% | 65.119% |
| AUROC | 0.689476 | 0.811759 |

Patient confusion changes from [[92,0],[44,0]] to [[78,14],[24,20]]. Twenty AD
patients become correctly identified at the cost of fourteen NC false positives;
accuracy rises 4.412 percentage points. Slice accuracy rises 67.574% to 72.647%,
macro-F1 0.405419 to 0.665148, AUROC 0.631629 to 0.788459. Epoch 30 was not selected:
its validation loss 0.513093 exceeds epoch 29's 0.509784. Selection remains based
on the predefined loss rule, not a retrospectively preferred metric.

This is improvement for one seed on a validation cohort used for selection, not
an unbiased test estimate or proof across seeds; the 80% target is not reached.
Do not compare E3 A42's 30-epoch result with E2 B42's one-epoch result as a fair
weighting comparison. Next E4: B42 balanced training for the same 30 epochs and
same selection/prediction protocol, then the paired seed43/44 runs. No E4 run or
protocol change was performed here. Private evidence is retained under
final_report/adni_manifest_local/e3_A42/. Codex audited saved evidence and reran
local tests; it did not perform remote training or alter original run outputs.


## E4 B42 weighted 30-epoch run and paired A42 comparison

E4 real B42 execution and paired A42 comparison passed acceptance on 25 October 2026. train.py,
predict.py and the accepted E3 runner are unchanged. run_e4.py fixes 30 epochs,
seed42, batch16, Adam lr0.001 and balanced class weights from the training cohort.
Validation/checkpoint selection remains unweighted, with first-epoch tie breaking.

Before training, read-only preflight checks the existing E3 A42 full training,
prediction and E3 summaries, CSV hashes, best epoch, fixed configuration, source
and input hashes. Default A42 source is the run directory from job 694924. The
A42 reference is never trained again or rewritten; it must remain unchanged during
E4. Its recorded code must match current train/predict dependencies. If preflight
fails, inspect evidence/synchronization rather than silently recomputing A42.

B42 saves first3 and full-best checkpoints and evaluates both with C3. The primary
comparison uses A42 full-best versus B42 full-best, not the one-epoch E2 model.
All 30 epoch order hashes and the initialization hash must match. Runtime versions,
GPU name, input/source hashes, prediction settings and other training settings must
match; only class_weighting/class_weight_details may differ. The comparison reports
patient accuracy, AD recall/precision, NC specificity, macro-F1, balanced accuracy,
AUROC, true AD detections and NC false positives. e4_summary.json includes B-minus-A
differences in raw metric units (multiply fractions by 100 for percentage points).
One seed cannot establish the planned multi-seed effectiveness conclusion.

### Cluster submission

Sync run_e4.py, run_e4.slurm and the entire tests directory. Keep the current
train.py, predict.py, modules.py, dataset.py, check_dataloader.py, check_cnn.py,
run_e2.py and run_e3.py available for the full suite. Activate the existing cluster
Python environment and submit on the login node:

```bash
cd /home/Student/s4982104/comp3710-code/ConvNeXt_ADNI_s4982104
sbatch run_e4.slurm "$(python -c 'import sys; print(sys.executable)')"
```

Default reference: $HOME/comp3710-data/adni/e3_A42_694924/run. If moved, pass its
actual run directory as the second argument (not its training directory).
The script runs 69 tests, then all E4 steps in one GPU job; no separate C3 job.
Account/partition comp3710, --mem=0, one-hour ceiling. Output:
$HOME/comp3710-data/adni/e4_B42_JOB_ID/. After completion replace JOB_ID and save:

```bash
sacct -j JOB_ID --format=JobID,State,ExitCode,Elapsed,MaxRSS \
  > "$HOME/comp3710-data/adni/e4_B42_JOB_ID/slurm_status_JOB_ID.txt"
```

Return to local final_report/adni_manifest_local/e4_B42/ (preserve structure):

```text
job_id.txt
test_results.txt
slurm_status_JOB_ID.txt
e4-slurm-JOB_ID.out
run/
  e4_summary.json
  comparison_A42_B42.csv
  training/
    config.json
    training_summary.json
    epochs.jsonl
    training.log
    first3/training_summary.json
  prediction_first3/
    prediction_summary.json
    prediction.log
    slice_predictions.csv
    patient_predictions.csv
  prediction_full/
    prediction_summary.json
    prediction.log
    slice_predictions.csv
    patient_predictions.csv
```

Scheduler output is in the project directory; other files are under e4_B42_JOB_ID.
Keep both .pt files on the cluster. Existing local E3 evidence is reused for
acceptance, so no second transfer of A42 is needed. Private CSVs are ignored by Git.
If failed, return e4_error.log and partial logs; preserve failed run directories.

Acceptance: COMPLETED/0:0, 69 tests pass, all B summaries and pair checks pass on
CUDA; complete 30 epochs, correct [408/550,408/266] weights, unweighted validation,
correct first3 and full-best selection, matching A/B initialization/all epoch order
hashes and other conditions. Each prediction exports 2720 slices/136 patients and
passes repeat/state checks. B-only reads total train244800/validation92480/test0;
A42 reference reading only reads metadata/CSVs, never MRI pixels. Recompute means,
metrics and comparison table against the accepted A42 epoch29 results. Evidence
integrity and complete execution determine engineering acceptance; superiority is
not required, and no threshold or experimental protocol changes are made here.

AI disclosure: Codex added run_e4.py/run_e4.slurm and four tests. After the missing
module failure, implementation passed all 69 local tests (11.916s). Tests cover
paired end-to-end execution, A42 file preservation, missing/tampered reference
rejection before training and training-order mismatch rejection. The actual saved
E3 evidence passed read-only preflight locally; no MRI/GPU run was performed.
Shell syntax/diff checks passed. No commit was made in this preparation turn.


### E4 acceptance — 25 October 2026

Job 695456 completed 30 epochs (0:0), wall time 6m19s (0.1053 GPU-hours).
Training took 329.09s; the complete E4 sequence took 337.40s. Peak CUDA allocated/
reserved memory was 277.52/518 MiB. Cluster tests: 69 passed (34.219s); fresh local
acceptance: 69 passed (11.683s). Shell syntax and diff checks passed.

Independent audit verified source/input/output/reference hashes, unchanged A42
reference links, identical initialization and all 30 training batch orders,
matching runtime/configuration except weighting, complete histories, prefix and
best selection, and total B reads 244800/92480/0. Both B prediction sets cover
2720 slices and 136 patients; all patient means, threshold classes, confusion-
derived metrics and AUROC were independently recomputed, alongside A42 full-best
outputs. Summary, comparison CSV and B-minus-A differences agree. Checkpoint hashes
link saved summaries; actual weights remain on the cluster and were not locally
reloaded. Repeat/state results rely on saved cluster checks.

| Patient-level metric | A42 ordinary loss | B42 weighted loss |
| --- | --- | --- |
| Selected epoch | 29 | 27 |
| Unweighted validation loss | 0.509784 | 0.525001 |
| Accuracy | 72.059% | 75.000% |
| AD recall | 20/44 = 45.455% | 19/44 = 43.182% |
| NC specificity | 78/92 = 84.783% | 83/92 = 90.217% |
| AD precision | 58.824% | 67.857% |
| Macro-F1 | 0.658472 | 0.678889 |
| Balanced accuracy | 65.119% | 66.700% |
| AUROC | 0.811759 | 0.798172 |

A confusion [[78,14],[24,20]]; B [[83,9],[25,19]]. B reduces NC false positives
by five but identifies one fewer AD patient; net four additional correct patients,
accuracy +2.941 percentage points. AD recall is -2.273 points, NC specificity
+5.435 points, macro-F1 +0.020417 and AUROC -0.013587. This is a mixed tradeoff:
it does not demonstrate improved AD recall or justify adopting weighting after
one seed. The 80% accuracy target remains unmet. No thresholds or selection rules
were changed. B epoch 27 is selected by validation loss, not by the largest F1.
B first3-best is epoch 3 (patient accuracy 65.441%, recall 47.727%, AUROC 0.690958).
B full-best slice accuracy is 73.015%, macro-F1 0.655219, AUROC 0.775120.

Next E5: paired A43/B43 and A44/B44 with the frozen 30-epoch protocol, then report
all three pairs and predeclared aggregate criteria. Do not select or discard seeds
based on these results. These are validation results used for model selection,
not final test estimates. Private evidence: final_report/adni_manifest_local/e4_B42/.
Codex audited returned evidence and reran local tests; no remote training or E5
execution occurred in this acceptance turn.


## E5 paired seeds 43/44 and three-seed aggregation

E5 passed cluster acceptance on 25 October 2026. run_e5.py runs A43, B43, A44, B44 in that
order, 30 epochs each from scratch, with batch16/Adam lr0.001 and the fixed split.
Only the training seed changes between pairs; the patient split stays seed42.
A uses none and B uses balanced class weighting; all validation losses remain
unweighted for checkpoint selection. No threshold/scheduler/augmentation changes.
Each new full-best checkpoint gets C3 repeat validation prediction. The first3
snapshot remains saved/checked but is not separately predicted in E5.

A42/B42 are read-only inputs from E3 job694924 and E4 job695456. Preflight checks
the accepted summaries, reference chains, CSVs, source and data before any new
training. Checks are repeated after all four runs. Each pair must have identical
initial parameters and all epoch batch orders; software/GPU, source/input hashes,
other settings and prediction configuration match across all six runs.

run_e4.validate_arm/compare_arms now accept an optional seed argument (default42)
to reuse the same checks for seeds43/44. Historical runner SHA-256 consequently
changes; original E4 evidence is retained unchanged and the new E5 summary records
both orchestration-source hashes. train.py/predict.py and their recorded dependencies
are unchanged; existing E4 behavior and tests still pass.

Outputs include six rows in runs.csv, three paired B-minus-A rows in
paired_differences.csv, and A/B/paired-difference mean and sample standard deviation
(ddof=1, n=3) in aggregate.csv. Patient-level metrics and counts are aggregated;
per-run training/prediction time and CUDA peaks are retained in runs.csv. Slice
metrics remain in each prediction summary. Undefined metrics remain null with
n_available recorded; no seed is silently dropped or replaced with zero.

The predeclared exploratory rule is true only if AD recall improves in at least
2/3 pairs, mean AD recall and balanced accuracy are higher, and mean macro-F1 is
not lower. candidate_for_further_validation=false is a valid experimental outcome,
not a failed job. This is not a statistical-significance or clinical-effectiveness
claim: all seeds share the same validation patients and checkpoint selection data.

### Run one E5 batch job

Sync run_e5.py, run_e5.slurm, the updated run_e4.py and all tests. Also ensure current
train.py, predict.py, modules.py, dataset.py, check_dataloader.py, check_cnn.py,
run_e2.py and run_e3.py are present. Activate the established cluster environment:

```bash
cd /home/Student/s4982104/comp3710-code/ConvNeXt_ADNI_s4982104
sbatch run_e5.slurm "$(python -c 'import sys; print(sys.executable)')"
```

Defaults read $HOME/comp3710-data/adni/e3_A42_694924/run and
$HOME/comp3710-data/adni/e4_B42_695456/run. If moved, pass these run directories as
the second and third script arguments. Do not point at their training subdirectories.
The job runs all 73 tests, then four training/prediction runs sequentially and
aggregates them with seed42. Account/partition comp3710, one GPU, --mem=0, one-hour
ceiling. Prior timings suggest roughly 20–30 minutes; actual occupancy determines
cost (roughly 0.4 GPU-hours is only an estimate). No separate C3 or seed42 job.
On failure it stops, retains finished runs and saves failure evidence; it does not
retry or overwrite runs automatically. Inspect partial results before resubmitting.

After completion replace JOB_ID and save:

```bash
sacct -j JOB_ID --format=JobID,State,ExitCode,Elapsed,MaxRSS \
  > "$HOME/comp3710-data/adni/e5_pairs_JOB_ID/slurm_status_JOB_ID.txt"
```

Return to local final_report/adni_manifest_local/e5_pairs/:

```text
job_id.txt
test_results.txt
slurm_status_JOB_ID.txt
e5-slurm-JOB_ID.out
run/
  e5_summary.json
  runs.csv
  paired_differences.csv
  aggregate.csv
  A43/  (same structure for B43, A44, B44)
    training/
      config.json
      training_summary.json
      epochs.jsonl
      training.log
      first3/training_summary.json
    prediction_full/
      prediction_summary.json
      prediction.log
      slice_predictions.csv
      patient_predictions.csv
```

Other than the scheduler log (project directory), files are in
$HOME/comp3710-data/adni/e5_pairs_JOB_ID/. All .pt files stay on the cluster;
private CSVs are ignored by Git. Existing local E3/E4 evidence supplies seed42,
so do not duplicate it. Return e5_error.log and partial outputs on failure.

Acceptance: COMPLETED/0:0 and 73 tests pass; four new complete 30-epoch runs, each
with 244800 train/81600 validation/0 test training reads, plus 0/5440/0 prediction
reads. Total NEW reads 979200/348160/0. Each prediction exports 2720 slices/136
patients, 20 per patient, with repeat/state checks passing. Best epochs/prefixes,
source/input/checkpoint/output/reference hashes and all three pairing checks must
match. Six run rows, three paired differences and sample statistics/decision must
be independently reproducible; no missing/duplicate seed and no seed-selection
based on outcomes. E6 interpretation follows acceptance; the 80% accuracy target
is not a prerequisite for valid execution.

AI disclosure: Codex implemented E5 orchestration/Slurm, generalized E4 validation
helpers without changing default42 behavior, and added four tests. Tests first
failed for the missing module; all 73 passed after implementation (14.241s),
including known mean/sample-SD/decision cases, complete four-run integration,
reference immutability and missing/tampered reference rejection. Actual saved
A42/B42 evidence passed read-only preflight. Shell syntax/diff checks passed;
no real E5 training or commit occurred in this preparation turn.


### E5 acceptance and three-seed findings — 25 October 2026

Job 695688 completed (0:0) in 22m14s (0.3706 GPU-hours); the E5 sequence took
1285.97s. All four new 30-epoch runs and best-checkpoint predictions passed.
Cluster tests: 73 passed (42.117s); fresh local tests: 73 passed (14.607s).
Shell syntax/diff checks passed. New reads total 979200 train/348160 validation/0
test; historical seed42 was reused without retraining. CUDA allocated peaks are
about 277.51–277.52 MiB, reserved 518 MiB per run.

Independent audit checked all six runs against the frozen manifest and source/
input/output/reference hashes; every prediction has 2720 unique slices, 136
patients and 20 slices per patient. All patient means, classes, per-class metrics,
accuracy, macro-F1 and AUROC were recomputed from CSVs; AUROC used independent
positive-negative probability comparisons. Every paired initialization and all
30 batch orders match, other conditions/runtime match, and prefix/best selection,
repeat/state checks, six rows, three deltas and sample statistics agree. Weight
bytes remain on the cluster; local audit checked their recorded provenance links,
not another physical checkpoint reload.

| Seed | A best epoch | B best epoch | A AD recall | B AD recall | B−A recall | NC false positives A→B |
| --- | --- | --- | --- | --- | --- | --- |
| 42 | 29 | 27 | 20/44 = 45.455% | 19/44 = 43.182% | −2.273 pp | 14→9 |
| 43 | 24 | 20 | 21/44 = 47.727% | 29/44 = 65.909% | +18.182 pp | 13→18 |
| 44 | 26 | 25 | 20/44 = 45.455% | 25/44 = 56.818% | +11.364 pp | 13→13 |

Patient-level means ± sample SD across the three seeds:

| Metric | A ordinary loss | B weighted loss |
| --- | --- | --- |
| Accuracy | 72.79 ± 0.74% | 75.74 ± 0.74% |
| AD recall | 46.21 ± 1.31% | 55.30 ± 11.44% |
| NC specificity | 85.51 ± 0.63% | 85.51 ± 4.90% |
| AD precision | 60.40 ± 1.48% | 65.12 ± 3.13% |
| Macro-F1 | 0.6666 ± 0.0091 | 0.7090 ± 0.0263 |
| Balanced accuracy | 65.86 ± 0.86% | 70.41 ± 3.34% |
| AUROC | 0.8069 ± 0.0218 | 0.8053 ± 0.0094 |

B improves AD recall in 2/3 pairs; mean paired gain is 9.09 percentage points
(sample SD 10.41 pp), corresponding to four additional AD detections per run on
average. Mean NC false positives remain 13.33, though individual seeds trade off
differently. Mean balanced accuracy increases 4.55 pp and macro-F1 0.04243.
Thus the predeclared candidate_for_further_validation rule is satisfied.

Conclusion: in this fixed validation cohort and three-seed protocol, class weighting
improves average AD recall and merits further validation. It does not improve every
seed; recall variability increases, and mean AUROC does not improve (−0.00165).
This is not statistical significance, an independent clinical sample, a final test
estimate or proof that weighting improves calibration. All accuracies remain below
80%. Do not cherry-pick B43 or retune thresholds after these observations.

E5 is accepted; E6 interpretation is recorded here. Next carry this result into
the CNN/ConvNeXt comparison and calibration/referral planning under the fixed split,
with a recorded model/checkpoint selection rule before any final test evaluation.
No new training, threshold change or test evaluation occurred in acceptance.
Evidence remains private in final_report/adni_manifest_local/e5_pairs/. Codex
independently audited returned outputs and reran local tests, not remote training.

## ConvNeXt preparation and small-batch check (N0–N2)

N0 weight preparation was accepted on 26 October 2026 from the returned
`convnext_pretrained_check` summary and terminal log: Python 3.11.13,
PyTorch 2.14.0+cu130, TorchVision 0.29.0+cu130, original 1000-class strict CPU
load passed, file size 114419221 bytes. Accepted full SHA256:
`983f1562536e84ff750a1576fb08e54de751dbf2e17c0d8a4a13704341fdcd3d`.
The weight binary was not returned locally; N2 independently rehashes it on the
cluster. The reported `zsh: read-only variable: status` was in the local output
collection wrapper after the remote summary was saved. Use `command_exit_code`
for shell exit-code variables; the Python JSON field `status` is valid.

### Fixed N1 protocol

- `modules.py` implements ConvNeXt-Tiny in PyTorch: stage depths 3/3/9/3,
  widths 96/192/384/768, 7x7 depthwise convolution, LayerNorm (epsilon 1e-6),
  channel MLP, layer scale 1e-6 and row-wise stochastic depth up to 0.1.
  This is an attributed adaptation of the architecture/state layout, not a claim
  that the architecture is original. See [third-party notices](THIRD_PARTY_NOTICES.md),
  [Liu et al., CVPR 2022](https://arxiv.org/abs/2201.03545), and the
  [TorchVision reference](https://github.com/pytorch/vision/blob/main/torchvision/models/convnext.py).
- Use local `ConvNeXt_Tiny_Weights.IMAGENET1K_V1` weights. First strictly load
  all 1000-class tensors and compare eval outputs with the official model on a
  validation smoke batch and a seeded random tensor (rtol 1e-5, atol 1e-6).
  Then replace only the final linear layer with a freshly initialized 768→2
  head. Fine-tune all layers. The script never downloads weights or silently
  falls back to random initialization.
- Convert the original grayscale PIL image to three identical RGB channels,
  bilinearly resize the short edge to 236 with antialias=True, center crop
  224×224, convert to float32 /255, normalize by mean
  [0.485,0.456,0.406] and std [0.229,0.224,0.225]. No intermediate CNN resize.
  Compare resulting tensors exactly against the installed official preset.
  Train/validation use the same deterministic processing, no random image
  augmentation. Stochastic depth remains part of the model's training behavior.
- Seed 42, batch 4, FP32, two updates, AdamW lr=1e-4 and weight_decay=0.01,
  deterministic algorithms enabled, TF32 disabled. Train-only patient-balanced
  CE weights (NC, AD) are [408/550,408/266]; validation CE is unweighted.
  Check finite logits/loss/gradients/parameters, nonzero total gradient norm,
  actual updates to both stem and head, unchanged validation state/no gradients,
  and identical saved/reloaded predictions within recorded tolerances.
- Same frozen 60/20/20 patient manifest; require 408/136/136 patients,
  8160/2720/2720 slices, 20 slices per patient and expected class counts.
  Test metadata is checked, but test image pixels are never opened.
- Select two training patients/class with an independent `random.Random(42)`
  over sorted IDs. Preview their first/middle/last lexicographic file paths
  (selection positions, **not verified anatomical slice order**). Save original,
  resize-with-crop-box and cropped panels plus a private JSON mapping.
  The script does not decide whether brain tissue was removed: visual review
  is required before accepting N2. An apparent crop problem triggers documented
  investigation and a new preprocessing version/short check, not silent changes.
- Record optimization timing with diagnostics, plus inference timing after two
  warm-ups and five repeats, synchronized on CUDA, excluding loading,
  preprocessing and transfers. These are smoke estimates, not the final fair
  latency benchmark. Report allocated/reserved CUDA peaks; enforce ≤20 GiB.
  Official-model comparison runs on CPU; peak GPU evidence covers the self model,
  optimizer/diagnostics, validation and reload, not two concurrent GPU models.

The comparison plan remains ConvNeXt with balanced loss versus CNN-B primarily,
with CNN-A retained as the loss ablation. Evaluation split, patient averaging
and 0.5 threshold remain fixed. Pretraining, input processing and optimizer
are explicit differences: this compares complete approaches, not architecture
alone. Full training/validation support in `train.py` and `predict.py` remains
N3 work; those entry points still run the CNN. N1 checkpoint is a smoke artifact,
not a selected model or an accuracy result.

### Run N2 on the cluster

Synchronize `modules.py`, `dataset.py`, `check_convnext.py`, `run_n2.slurm`,
`requirements.txt`, `THIRD_PARTY_NOTICES.md`, `README.md`, `.gitignore`, and the
complete `tests/` folder. Keep existing `train.py`, `check_dataloader.py` and
other helper files present. Do not copy the Mac `.venv` or install Mac wheels.
The verified cluster environment already has torch/torchvision; no reinstall
is required. Local requirements pin torch==2.14.0 and torchvision==0.29.0;
record platform-specific CUDA build suffixes separately.

On the login node:

```bash
cd /home/Student/s4982104/comp3710-code/ConvNeXt_ADNI_s4982104
sbatch run_n2.slurm /home/Student/s4982104/comp3710-env/bin/python
```

This submits one GPU, account/partition `comp3710`, `--mem=0`, 20-minute limit.
The script first runs the full offline test suite, then the real check on CUDA.
It uses the already prepared files at:

```text
/home/Student/s4982104/comp3710-data/pretrained/convnext_tiny-983f1562.pth
/home/Student/s4982104/comp3710-data/pretrained/convnext_tiny_weights_summary.json
```

Evidence is saved under
`$HOME/comp3710-data/adni/convnext_smoke_check_<job_id>/`.
Do not reuse a previous output directory. If the check fails, retain the failed
run and return its logs first. If a verified memory issue needs a smaller batch,
submit a new job with optional second argument `2`, then `1` if necessary;
do not simultaneously change resolution/precision.

After completion, replace `JOB_ID` with the actual numeric ID:

```bash
sacct -j JOB_ID --format=JobID,State,ExitCode,Elapsed,AllocTRES,MaxRSS
```

Return to local `final_report/adni_manifest_local/convnext_smoke_check/`:

1. `job_id.txt`, `exit_code.txt`, `test_results.txt` and saved `sacct` output.
2. `n2-slurm-<job_id>.out` and `.err` from the project directory.
3. `run/convnext_check_summary.json`, `run/convnext_check.log`, `run/config.json`.
4. `run/crop_previews.json` and the complete `run/crop_previews/` PNG folder.

Leave `run/smoke_checkpoint.pt` and pretrained `.pth` on the cluster. Do not
commit images, patient mappings or weights. The summary records the checkpoint
hash and reload result; a local reviewer without the binary cannot independently
reload it. N2 acceptance requires numerical checks, source/input hashes and
counts, real weight identity, GPU resources, successful Slurm exit, and visual
review of the crop examples. No accuracy or loss-decrease target applies to two
updates; 80% remains the eventual classification target.

### AI usage: N0 acceptance and N1 implementation

Codex inspected the returned weight summary/log, implemented and tested the
ConvNeXt adaptation, preprocessing and checker, and drafted these instructions.
The implementation references the official architecture and uses the official
model as a numerical oracle. Local tests use explicitly synthetic random
reference states and generated images, without downloading pretrained weights
or reading ADNI test images. Passing those tests is not a claim of completed
real-weight GPU validation; the user runs N2 and returns evidence for acceptance.

Local N1 verification on 26 October 2026: 81 tests passed in 62.453 seconds (8 new tests); shell syntax and whitespace checks passed. Real-data/GPU N2 and crop-content review are pending. No commit or push was performed in this preparation turn.

### N2 accepted — 27 October 2026

Job 700565 completed (0:0) in 2m57s, approximately 0.04917 GPU-hours including
tests. Cluster 81 tests passed in 148.552s; local acceptance reran all 81 in
61.494s. Source/input/output hashes matched; all read paths matched the frozen
split. Old directory names containing `test` are provenance, not the current
split assignment. Reads: train 8, validation 4, test 0, plus 12 training preview
reads. Original 1000-class official/self-model outputs and binary checkpoint
reload both had maximum difference zero. Both optimization steps updated stem
and head; validation did not change state or create gradients.

All 12 crop previews (four training patients, two per class) were independently
viewed again. No obvious additional clipping of visible brain tissue, new
stretching or blank output was found; removed borders were predominantly black
background. Existing irregular lower edges are visible in original images too.
Recropping the displayed original panels reproduced all 12 output panels exactly.
This is sampled preprocessing review, not a full-dataset quality guarantee.
Retain the official preset for the next pilot. The original run summary remains
unchanged; the later acceptance is recorded separately.

A100 peak allocated 868.59 MiB, reserved 932 MiB; 27,821,666 trainable parameters.
Batch-four inference averaged 8.904 ms after two warm-ups over five repeats,
excluding I/O/preprocessing/transfers. This smoke estimate is not a final latency
benchmark. First/second update times were 6.836/0.0936s; do not extrapolate startup
cost into epoch estimates. Losses were 0.701318/0.690366, validation loss 0.601142;
two updates do not establish convergence or classification performance.

Pretrained/checkpoint binaries remain on the cluster; local acceptance checked
returned evidence rather than independently reloading those binaries. Private
acceptance evidence is under `adni_manifest_local/convnext_smoke_check/` outside
the repository. N3 is next: add ConvNeXt full training/validation and prediction
support, retain the current smoke configuration for a three-epoch pilot, and
measure full-epoch cost before committing to long runs. AI assistance in this
acceptance comprised evidence/hash/manifest checks, visual inspection, tests,
documentation and the user-authorized commit; no remote training was performed.

## N3: three full ConvNeXt epochs

`run_n3.py` reuses the tested epoch loop in `train.py` and prediction/aggregation
functions in `predict.py`; their existing CNN entry points are unchanged.
Fixed protocol: pretrained ConvNeXt-Tiny, all layers trainable, stochastic depth
0.1, FP32, batch 4, seed 42, three epochs, AdamW lr=1e-4 and weight_decay=0.01.
The deterministic N2 preprocessing is unchanged. Training-only patient-balanced
weights are computed from the training manifest. Validation CE is unweighted;
the lowest validation loss selects best.pt, with the earliest epoch winning ties.
There is no augmentation, scheduler or early stopping.

Checkpoint config contains the model, preprocessing, optimizer and source weight
identity. Strict loading checks both the full expected config and fixed protocol,
then verifies the state dictionary and synthetic-probe logits without extra MRI
reads. Two independent model reloads predict the complete validation split;
paths/labels/patient/scan IDs must exactly match the frozen validation manifest.
Patient probabilities are means over 20 slices, threshold 0.5. Test pixels remain
unread. Full epoch and prediction-pass times include loading/transfers and checks;
they are not isolated inference-latency benchmarks. CUDA peaks are recorded
separately for training and prediction. Slurm elapsed time times the one allocated
GPU is the job GPU-hours, including tests and startup.

Synchronize run_n3.py, run_n3.slurm, README.md, .gitignore and tests/, checking
existing model/dataset/helper code hashes first. Keep the existing cluster venv.

```bash
sbatch run_n3.slurm /home/Student/s4982104/comp3710-env/bin/python
```

Outputs: ~/comp3710-data/adni/convnext_pilot_JOB_ID/run/{training,prediction}/
and n3_summary.json. Each training epoch reads 8160 train / 2720 validation;
training totals 24480 / 8160 / 0, prediction totals 0 / 5440 / 0, combined
24480 / 13600 / 0. The prediction summaries/CSVs and training config, summaries,
epochs.jsonl, logs, job/test/exit/Slurm records go to the private local directory
final_report/adni_manifest_local/convnext_pilot/. Keep best.pt on the cluster.
Output directories cannot be reused; failures retain summaries and n3_error.log.
Full suite: `.venv/bin/python -m unittest discover -s tests -v`. Synthetic N3
integration tests mock only the heavy pretrained model and do not claim real
ConvNeXt/GPU success; N2 supplies that separate evidence.


### N3 acceptance (27 October 2026)

Job 701651 completed with exit 0:0 on one A100 in 12m02s (0.201 GPU-hours).
All 87 tests passed locally and on the cluster. Independent acceptance verified
source/input/output hashes, all epoch batch-order hashes, exact validation
membership, 20-slice patient means, and both levels of metrics including AUROC.
The two checkpoint-reload predictions agreed exactly; test pixel reads were zero.

Minimum unweighted validation loss selected epoch 1. Patient accuracy was 79.41%,
AD recall 70.45%, AD precision 67.39%, NC specificity 83.70%, macro-F1 0.7675,
and AUROC 0.8649. Confusion matrix (true/predicted NC, AD) was
[[77, 15], [13, 31]]; slice accuracy was 76.25%. These are pilot validation results,
not final test results or evidence that the 80% target has been achieved.
Validation loss rose from 0.8194 to 0.8414 and 2.9240 while training loss fell.
N4 must investigate stability/overfitting before freezing long-run settings.

N3 retained the N2 preprocessing source/config. The same 12 N2 crop previews
were rechecked and their hashes were unchanged; no obvious additional tissue
clipping was observed. No new N3 preview sample was produced. Mean full-epoch
time was 178.11s; peak training allocated memory was 870.00 MiB. Thirty epochs
for three seeds would take about 4.45 GPU-hours before additional overhead,
requiring the initial three-hour ConvNeXt allocation to be revisited.

N3 evidence and independent acceptance are stored privately in
adni_manifest_local/convnext_pilot/; convnext_smoke_check/ retains N2 evidence.
Weight binaries stay on the cluster. AI assistance covered review of the existing
implementation and evidence, independent metric/hash checks, crop inspection,
local tests, documentation and the authorized commit; no remote training was
performed during acceptance.
