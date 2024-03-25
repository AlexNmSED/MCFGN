# MCFGN
Cyclic Translations Between Pathomics and Genomics Improve Automatic Cancer Diagnosis from Whole Slide Images

## Pre-requisites:
* Linux
* NVIDIA GPU (Test on P100 * 4)
* install requirements via pip3 install -r requirements.txt

## 0. Patch Extraction
This step is based on CLAM tissue segmentation and patching code.

The first step focuses on segmenting the tissue and excluding any holes. The segmentation of specific slides can be adjusted by tuning the individual parameters (e.g. dilated vessels appearing as holes may be important for certain sarcomas.) The digitized whole slide image data are stored under a folder named DATA_DIRECTORY

```
DATA_DIRECTORY/
	├── slide_1.svs
	├── slide_2.svs
	└── ...
```
