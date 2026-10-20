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
