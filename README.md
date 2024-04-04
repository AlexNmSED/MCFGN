# MCFGN
Cyclic Translations Between Pathomics and Genomics Improve Automatic Cancer Diagnosis from Whole Slide Images
(Reconstructing)
![Alt text](/imgs/model.jpg "Overview of our proposed MCFGN model")
Overview of our proposed MCFGN model, comprising three modules: (a) self-supervised
pretraining module; (b) hierarchical WSI feature construction module; (c) cyclic feature generation across modalities module. Our model draws inspiration from bidirectional translation in natural language processing, aiming to harness joint representation developed in modality translation, showcasing potential applications in cancer diagnosis. The key advantage of our MCFGN model is its reliance solely on WSIs during the testing phase, simplifying the diagnostic process.

## Pre-requisites:
* Linux
* NVIDIA GPU (Test on P100 * 4)
* install requirements via pip3 install -r requirements.txt

## Datasets:
Two large cohorts of H&E stained WSIs from TCGA data are utilized in this study. The first cohort consists of 1133 TCGA breast cancer WSIs, each accompanied by clinical diagnoses and genetic information. The second cohort consists of 1053 TCGA lung caner WSIs, also accompanied by clinical diagnoses and genetic information. The evaluated tasks include: (1) Subtyping of invasive breast cancer (i.e., Invasive Ductal Carcinoma (IDC) vs Invasive Lobular Carcinoma (ILC)); (2) Prediction of biomarker status for breast cancer patients, including ER, PR, and HER2; (3) Subtyping of non-small cell lung cancer (i.e., Lung Adenocarcinoma (LUAD) vs Lung Squamous Cell Carcinoma (LUSC)); (4) Prediction of Tumor mutation burden (TMB) for lung cancer patients.

## 0.1 Pre-processing:  Genetic Data
The gene expression data of cancer patients were retrieved from the official website of TCGA database.
The pre-processing details are as follows:
* The log transformation, i.e., $x = log_2(x+1)$, is applied on each gene's expression value $x$ in the gene expression data, resulting in updated gene expression values.
* Genes with a missing expression rate exceeding the 10% threshold are eliminated from the dataset as they are not suitable for imputation. For the remaining missing entries in the gene expression data, a nearest neighbor imputation method is employed to fill in the values.
* The gene expression data undergo z-score normalization, calculated as follows: $z = (x - \mu ) / \sigma$, where $\mu$ is the mean, $\sigma$ is the standard deviation, and $x$ represents the original value normalized by the standard deviation units away from the mean.
* The Cox proportional hazards model is employed to identify genes highly correlated with patients' overall survival risks. The number of selected genes matches the number of WSI-level features, ensuring consistent cyclic translations across modalities.

## 0.2 Pre-processing: Patch Extraction
This step is based on CLAM tissue segmentation and patching code.

The first step focuses on segmenting the tissue and excluding any holes. The segmentation of specific slides can be adjusted by tuning the individual parameters (e.g. dilated vessels appearing as holes may be important for certain sarcomas.) The digitized whole slide image data are stored under a folder named DATA_DIRECTORY

```
DATA_DIRECTORY/
	├── slide_1.svs
	├── slide_2.svs
	└── ...
```
Then use the CLAM basic command :

```
python create_patches_fp.py --source DATA_DIRECTORY --save_dir RESULTS_DIRECTORY --patch_size 256 --seg --patch --stitch 
```
In our work, we will extract 256*256 image patches in non-overlapping. The above command will segment every slide in DATA_DIRECTORY using default parameters, extract all patches within the segemnted tissue regions, create a stitched reconstruction for each slide using its extracted patches (optional) and generate the following folder structure at the specified RESULTS_DIRECTORY:

```
RESULTS_DIRECTORY/
	├── masks
    		├── slide_1.png
    		├── slide_2.png
    		└── ...
	├── patches
    		├── slide_1.h5
    		├── slide_2.h5
    		└── ...
	├── stitches
    		├── slide_1.png
    		├── slide_2.png
    		└── ...
	└── process_list_autogen.csv
```
The masks folder contains the segmentation results (one image per slide). The patches folder contains arrays of extracted tissue patches from each slide (one .h5 file per slide, where each entry corresponds to the coordinates of the top-left corner of a patch) The stitches folder contains downsampled visualizations of stitched tissue patches (one image per slide) (Optional, not used for downstream tasks) The auto-generated csv file process_list_autogen.csv contains a list of all slides processed, along with their segmentation/patching parameters used.

## 1.0 Pathological MAE Pre-training
This work is built using [timm](https://github.com/rwightman/pytorch-image-models/tree/master/timm), [DeiT](https://github.com/facebookresearch/deit), [DINO](https://github.com/facebookresearch/dino
), [MoCo v3](https://github.com/facebookresearch/moco-v3), [BEiT](https://github.com/microsoft/unilm/tree/master/beit), [MAE-priv](https://github.com/BUPT-PRIV/MAE-priv), and [MAE](https://github.com/facebookresearch/mae) repositories.
Installation and preparation follow that repo.

## 2.0 Feature Extraction
Followed by using the publicly-available CLAM library:
```
CUDA_VISIBLE_DEVICES=0,1 python extract_features_fp.py --data_h5_dir DIR_TO_COORDS --data_slide_dir DATA_DIRECTORY --csv_path CSV_FILE_NAME --feat_dir FEATURES_DIRECTORY --batch_size 512 --slide_ext .svs
```
The above command expects the coordinates .h5 files to be stored under DIR_TO_COORDS and will use 2 GPUs (0 and 1) and to extract 384-dim features from each tissue patch for each slide and produce the following folder structure:
```
FEATURES_DIRECTORY/
    ├── h5_files
            ├── slide_1.h5
            ├── slide_2.h5
            └── ...
    └── pt_files
            ├── slide_1.pt
            ├── slide_2.pt
            └── ...
```
where each .h5 file contains an array of extracted features along with their patch coordinates (note for faster training, a .pt file for each slide is also created for each slide, containing just the patch features). Each *.pt file is a [M × 384]-sized Tensor containing extracted 384-dim embeddings for M patches in the WSI.

Also, we extracted features at 40× magnification for [4096 × 4096] image regions. Other settings are the same as before. But, each *.pt file is a [M × 256 × 384]-sized Tensor containing extracted 384-dim embeddings for M regions in the WSI, which each region represented as a 256-length sequence of [256 × 256] patch embeddings.

## 3.0 Weakly-Supervised Training
Following ViT-16/256 pretraining and pre-extracting instance-level [256 × 256] features using ViT-16, we extend the publicly-available CLAM scaffold code for running 5-fold cross-validation experiments as well as implement several of the current weakly-supervised baselines.
