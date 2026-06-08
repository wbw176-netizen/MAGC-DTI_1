# MAGC-DTI

MAGC-DTI is a drug-target interaction prediction project. The model combines drug and protein representations, applies AGICA-based cross-modal interaction, and predicts binary interaction labels.

The repository supports two input modes:

- Raw-input mode: SMILES are converted to molecular graphs and proteins are encoded as amino-acid indices.
- Pre-extracted feature mode: ChemBERTa drug features and ESM2 protein features are loaded from `.npy` files.

## Repository Structure

```text
MAGC-DTI-main/
  code/
    main.py                 # Main training entry
    model.py                # MAGC_DTI model and loss helpers
    model_cdan.py           # CDAN / cross-domain training
    5fold_validation.py     # Stratified K-fold validation
    dataloader.py           # Raw and pre-extracted feature datasets
    trainer.py              # Training and evaluation loop
    configs.py              # Default hyperparameters
    extractor.py            # PLM feature extraction utilities
    pre_data_extractor.py   # Variable-length feature extraction wrapper
  datasets/                 # Dataset directory
  Distributions/            # Dataset distribution analysis figures
  visualization/            # Feature visualization outputs
  environment.yaml          # Conda environment specification
```

## Environment Setup

Create the conda environment from `environment.yaml`:

```bash
conda env create -f environment.yaml
conda activate MAGC_DTI
```

The environment includes PyTorch, DGL, DGLLife, RDKit, Transformers, scikit-learn, and other required packages.

If CUDA or DGL installation fails, install the PyTorch and DGL versions that match your local CUDA driver.

## Downloads

The full datasets and generated split files are hosted externally because they are too large for this repository:

- [Download Full Datasets (Google Drive)](https://drive.google.com/drive/folders/1IpJ8g2GJPoX70LL9fgDRYGFlmS3e6Cob?usp=drive_link)

Download and unzip the dataset files into:

```text
datasets/
```

If the Google Drive folder also contains pre-extracted feature files, place them under a feature root such as:

```text
code/features/varlen/<dataset_name>/
```

The supported PLM checkpoints are:

| Modality | Model | Link |
| :--- | :--- | :--- |
| Drug | ChemBERTa | [DeepChem/ChemBERTa-77M-MLM](https://huggingface.co/DeepChem/ChemBERTa-77M-MLM) |
| Target | ESM-2 | [facebook/esm2_t36_3B_UR50D](https://huggingface.co/facebook/esm2_t36_3B_UR50D) |
| Target | ProtT5 | [Rostlab/prot_t5_xl_uniref50](https://huggingface.co/Rostlab/prot_t5_xl_uniref50) |

Place local PLM weights in a directory such as:

```text
plm_models/
  chemberta_model/
  esm2_3B_model/
  protT5_model/
```

## Data Preparation

The expected raw dataset format is:

```text
datasets/
  Drugbank/
    random2/
      train.csv
      val.csv
      test.csv
  BindingDB/
    random/
      train.csv
      val.csv
      test.csv
```

Each CSV file should contain at least these columns:

```text
SMILES,Protein,Y
```

For cluster-aware or cross-domain training, optional cluster columns can be provided:

```text
drug_cluster,target_cluster
```

Full datasets are not included in this repository because of file size. Use the Google Drive link above and place the downloaded files under `datasets/`.

## Pre-Extracted Feature Format

Pre-extracted feature training expects `.npy` files organized by split:

```text
features_root/
  train/
    train_smiles_features.npy
    train_protein_features_esm2.npy
    train_labels.npy
  val/
    val_smiles_features.npy
    val_protein_features_esm2.npy
    val_labels.npy
  test/
    test_smiles_features.npy
    test_protein_features_esm2.npy
    test_labels.npy
```

Expected array shapes:

```text
*_smiles_features.npy        [num_samples, drug_feature_dim]
*_protein_features_esm2.npy  [num_samples, protein_feature_dim]
*_labels.npy                 [num_samples]
```

Typical dimensions are:

```text
ChemBERTa: 384
ESM2:      1280
```

`model_cdan.py` also supports flat feature files with a strategy suffix, for example:

```text
features_root/
  train_smiles_mean_mean.npy
  train_protein_mean_mean.npy
  train_labels_mean_mean.npy
```

Use `--feature_strategy mean_mean` when using this naming style.

## Training

Run raw-input training:

```bash
cd code
python main.py --data Drugbank --split random2
```

Run training with pre-extracted ChemBERTa and ESM2 features:

```bash
cd code
python main.py \
  --use_pretrained_features \
  --feature_dir ../code/features/varlen/Drugbank \
  --run_name Drugbank_plm
```

Training outputs are written under:

```text
output/result/<dataset>/<split>/
```

The saved artifacts include:

```text
best_model.pth
model_architecture.txt
config.txt
valid_markdowntable.txt
test_markdowntable.txt
train_markdowntable.txt
result_metrics.pt
```

## CDAN / Cross-Domain Training

`code/model_cdan.py` trains MAGC-DTI with cluster-aware loss and optional CDAN-style domain adaptation. This entry uses pre-extracted features.

Default pre-extracted feature training:

```bash
cd code
python model_cdan.py \
  --mode default \
  --feature_dir ../code/features/varlen/Drugbank
```

Cross-domain training:

```bash
cd code
python model_cdan.py \
  --mode cross_domain \
  --feature_dir ../code/features/varlen/BindingDB \
  --source_split source_train \
  --target_train_split target_train \
  --target_test_split target_test
```

With flat feature files and a strategy suffix:

```bash
cd code
python model_cdan.py \
  --mode default \
  --feature_dir ../code/features/Drugbank \
  --feature_strategy mean_mean
```

Useful options:

```text
--lambda_cluster       Weight for cluster consistency loss
--lambda_domain        Weight for domain adaptation loss
--use_cluster_loss     Enable cluster consistency loss
--analyze_clusters     Report per-cluster test performance
```

## Five-Fold Validation

Run K-fold validation on raw CSV data:

```bash
cd code
python 5fold_validation.py \
  --data BindingDB \
  --split random \
  --n_splits 5
```

Run K-fold validation with pre-extracted features:

```bash
cd code
python 5fold_validation.py \
  --use_pretrained_features \
  --feature_dir ../code/features/varlen/Drugbank \
  --n_splits 5
```

The K-fold feature mode uses labels from:

```text
<feature_dir>/train/train_labels.npy
```

## Feature Extraction Notes

This repository contains feature extraction utilities under `code/extractor.py` and `code/pre_data_extractor.py`.

The current training code is most stable when features are prepared in the split-based `.npy` format described above. If you use `extractor.py`, check the generated file names and either:

- Rename files into the split-based format expected by `main.py` and `5fold_validation.py`, or
- Use `model_cdan.py --feature_strategy <strategy>` for flat strategy-suffix files.

`pre_data_extractor.py` expects a `TwoStageFeaturePipeline` implementation. If that class is not present in your local copy, use already generated `.npy` features or restore the missing extractor implementation before running that script.

## Model Overview

The main model is `MAGC_DTI` in `code/model.py`.

Raw-input mode:

```text
SMILES -> molecular graph -> MolecularGCN
Protein sequence -> amino-acid index sequence -> MSAA/MSAFI
```

Pre-extracted feature mode:

```text
ChemBERTa drug vector -> BiEncoderFeatureExtractor
ESM2 protein vector   -> BiEncoderFeatureExtractor
```

Fusion and prediction:

```text
Drug features + protein features -> AGICA cross-modal interaction
Drug/protein/shared pooled vectors -> MPRC classifier -> DTI logit
```

## Metrics

The training scripts report:

```text
AUROC
AUPRC
F1
Sensitivity
Specificity
Accuracy
Precision
MCC
Loss
```

## Troubleshooting

`ModuleNotFoundError: No module named 'dgllife'`

Install the conda environment from `environment.yaml` or install `dgllife` in the active environment.

`ModuleNotFoundError: No module named 'model'`

Run scripts from the `code/` directory:

```bash
cd code
python main.py ...
```

Feature file not found

Check that your feature directory follows the required split structure:

```text
train/train_smiles_features.npy
train/train_protein_features_esm2.npy
train/train_labels.npy
```

Dimension mismatch in pre-extracted feature mode

Make sure train, validation, and test splits use the same ChemBERTa and ESM2 feature dimensions.

## References

1. Chithrananda, S., Grand, G., & Ramsundar, B. ChemBERTa: Large-Scale Self-Supervised Pretraining for Molecular Property Prediction. arXiv:2010.09885.
2. Lin, Z., Akin, H., Rao, R., et al. Evolutionary-scale prediction of atomic-level protein structure with a language model. Science, 379(6637), 1123-1130.
3. Elnaggar, A., Heinzinger, M., Dallago, C., et al. ProtTrans: Toward Understanding the Language of Life Through Self-Supervised Learning. IEEE TPAMI, 44(10), 7112-7127.
4. McInnes, L., Healy, J., & Melville, J. UMAP: Uniform Manifold Approximation and Projection for Dimension Reduction. arXiv:1802.03426.
5. Wishart, D. S., et al. DrugBank 5.0: a major update to the DrugBank database for 2018. Nucleic Acids Research, 46(D1), D1074-D1082.
