Cross-study generalization (csg) with cell-line drug sensitivity datasets.

The workflow to generate CSG results includes three steps:
1. Generate data (features, response, data splits) in a generic foramt that can be used by ML/DL models.
2. Preprocess data from step 1 to conform the model's API, run training and testing, and save raw predictions in a consistent format.
3. Pass raw model predictions from step 2 to the API to generate csg tables.

## 1. Generate data
Generate data per drug sensitivity study (e.g., cancer and drug features, response values). The code below generates datasets for CCLE, CTRP, GDSC1, GDSC2, and gCSI.

```
python src/build_dfs_july2020.py
```

TODO: add code that downloads the data from ftp.

The data is saved in `ml.dfs/July2020/data.STUDY_NAME` (e.g., data.ccle, data.ctrp, etc). Each folder contains multiple files. For example, in data.GDSC1:
* `rsp_gdsc1.csv`: drug response samples with drug ids (DrugID), cell ids (CancID), and multiple measures of drug response. For drug response, we use the `AUC` (area under the dose-response curve).
* `ge_gdsc1.csv`: gene expression. The first column is CancID, and the remaining columns are gene names (GeneBank) prefixed with `ge_`.
* `mrd_gdsc1.csv`: mordred descriptors. The first column is DrugID, and the remaining columns are mordred descriptors prefixed with `mordred_`.
* `fps_gdsc1.csv`: ECFP2 morgan fingerprints (radius=2, nBits=512). The first column is DrugID, and the remaining columns are fingerprints prefixed with `ecfp2_`.
* `splits`: This folder contains multiple sets of train/test split ids. These ids are used use to index samples in the `rsp_{STUDY_NAME}.csv`. Each split comes with a pair of files:
    * Ids for training: split_{split_number}_tr_id
    * Ids for testing: split_{split_number}_te_id
The ML model should be able to train using training samples (the split_{split_number}_tr_id) and test using test samples (the split_{split_number}_te_id).

## 2. Preprocessing, training, inference
The ML model should be able to take data from data.STUDY_NAME, pre-process to conform the model's training/test API, train the model, and save predictions. These steps are model-depend. All models should save raw predictions while following the format and naming convention as required by the csg API. Check `train_all.py` for executing this step LightGBM.

```
python train_all.py
```

__Preprocessing__. Take data.STUDY_NAME and generate data structures to conform the model's train/test API. For example:
* LightGBM can take csv files
* GraphDRP requires PyTorch data loaders for train, val, and test sets

__Training__. Train the model using split ids (each split produces a trained model). The training dataset is the __source__ dataset. For example:
* Train LightGBM with ccle split 0 train ids --> model0
* Train LightGBM with ccle split 1 train ids --> model1

__Inference__. Using the trained models for each split, run inferene with all datasets. The inference datasets are the __target__ datasets. When __source__ and __target__ are the same, use test ids for inference. Alternatively, when __source__ and __target__ are different, run inference for all the available samples in the __target__ dataset.

A single file of model predictions should be created for a unique combination of (source dataset, target dataset, split) as follows: `SOURCE_TARGET_split_#.csv`.

For example, if `source` is data.ccle, there are 5 `target` datasets (data.ccle, data.ctrp, data.gcsi, data.gdsc1, data.gdsc2) and 10 data splits, the model should save raw predictions in the following files:
* target is ccle (test ids)     ccle_ccle_split_0.csv, ..., ccle_ccle_split_9.csv
* target is ctrp (all samples)  ccle_ctrp_split_0.csv, ..., ccle_ctrp_split_9.csv
* target is gcsi (all samples)  ccle_gcsi_split_0.csv, ..., ccle_gcsi_split_9.csv
* target is gdsc1 (all samples) ccle_gdsc1_split_0.csv, ..., ccle_gdsc1_split_9.csv
* target is gdsc2 (all samples) ccle_gdsc2_split_0.csv, ..., ccle_gdsc2_split_9.csv

Each such file must contain the following 4 columns: `DrugID`, `CancID`, `True`, and `Pred`. The DrugID and CancID are drug and cell line ids (used in data.STUDY_NAME files). The True and Pred are the ground truth and predictions, respetively. 

All these files should be stored in a single folder `results.MODEL_NAME` (e.g., results.LGBM, results.IGTD, results.UNO).

## Cross-study generalization API
The API will take raw model predictions from results.MODEL_NAME and generate tables for the model.

```
python csg_post_process.py --model_name LGBM
```
