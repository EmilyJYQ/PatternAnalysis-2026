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

TODO: exact dependency versions and verified commands after implementation.

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

## Evaluation and results

TODO: identical evaluation splits, per-class precision/recall, macro-F1,
AUROC, accuracy, confidence analysis, and resource comparison.

## Failure analysis and recommendation

TODO: analyze 3–5 real failure cases and summarize operational trade-offs.

## Artificial Intelligence Usage Disclosure

Codex assisted with repository setup and this initial file scaffold, and with
a read-only data/resource audit. Repository paths, branch configuration, and
required files were checked. No model code or model performance has yet been
validated. Update this disclosure as development proceeds.

## References

TODO: cite the course specification, model papers, and any adapted code.
