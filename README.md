# Thyroidiomics: An Automated Pipeline for Segmentation and Classification of Thyroid Pathologies from Scintigraphy Images


## Introduction
This codebase is related to our submission to EUVIP 2024:<br>
> Anonymous authors, _Thyroidiomics: An Automated Pipeline for Segmentation and Classification of Thyroid Pathologies from Scintigraphy Images_.

<p align="center">
<img src="./assets/flowchart.png" alt="Figure" height="285" />
</p>
<p align="justify">
<sub><i>
    Figure 1: <i>Thyroidiomics</i>: the proposed two-step pipeline to classify thyroid pathologies into three classes, namely, MNG, TH and DG. Scenario 1 represents the pipeline dependent on physician's delineated ROIs as input to the classifier, while scenario 2 represents the fully automated pipeline operating on segmentation predicted by ResUNet.
</i></sub>
</p>

<p align="justify">
The objective of this study was to develop an automated pipeline that enhances <b>thyroid disease classification</b> using thyroid <b>scintigraphy</b> images, aiming to decrease assessment time and increase diagnostic accuracy. Anterior thyroid scintigraphy images from <b>2,643 patients</b> from <b>nine centers</b> were collected and categorized into <b>multinodal goiter (MNG)</b>, <b>thyroiditis (TH)</b>, and <b>diffuse goiter (DG)</b>, based on clinical reports, and then segmented by an expert. A <b>Residual UNet (ResUNet)</b> model was trained to perform auto-segmentation [1]. <b>Radiomics</b> features were extracted from both physician's <b>(scenario 1)</b> and ResUNet segmentations <b>(scenario 2)</b>, followed by omitting highly correlated features using Spearman's correlation, and feature selection using <b>Recursive Feature Elimination</b> with <b>eXtreme Gradient Boosting (XGBoost)</b> as the core [2]. All models were trained under <b>leave-one-center-out cross-validation (LOCOCV)</b> scheme, where nine instances of algorithms was iteratively trained and validated on data from eight centers and tested on the ninth for both scenarios separately. 
</p>

### Segmentation and classification performance
<p align="center">
<img src="./assets/segmentation.png" alt="Figure" width="45%" />
<img src="./assets/classification_classwise.png" alt="Figure"  width="50%" />
</p>
<p align="justify">
<sub><i>
    Figure 2: (Left) (a) Distribution of center-level mean DSC over 9 centers for the classes, MNG, TH and DG. (b)-(d), (e)-(g), and (h)-(j) show some representative images from each class with the ground truth (red) and ResUNet predicted (yellow) segmentation of thyroid. The DSC between ground truth and predicted masks is shown in the bottom-right of each figure. (Right) Various class-wise metrics for classification were used to evaluate model performance in two scenarios: features extracted from the physician's delineated ROIs and those from ResUNet predicted ROIs. The boxplots show the distribution of metrics over the nine centers as test sets for the three thyroid pathology classes, MNG, TH and DG.
</i></sub>
</p>

## How to get started?
<p align="justify">
Follow the intructions given below to set up the necessary conda environment, install packages, preprocess dataset in the correct format so it can be accepted as inputs by the code, train model and perform anomaly detection on test set using the trained models. 
</p>

- **Clone the repository, create conda environment and install necessary packages.** The first step is to clone this GitHub codebase in your local machine, create a conda environment, and install all the necessary packages. This code base was developed primarily using python=3.9.19, pandas=2.2.2, numpy=1.26.4, SimpleITK=2.3.1, PyTorch=1.11.0, monai=1.3.1, pyradiomics=3.1.0 and CUDA 11.4 on a Microsoft Azure virtual machine with Ubuntu 20.04, so the codebase has been tested only with these configurations. The virtual machine had one GPU with 16 GiB of RAM and 6 vCPUs with 112 GB of RAM. We hope this codebase will run in other suitable combinations of different versions of these libraries, but we cannot guarantee that. Proceed with caution and feel free to modify wherever necessary!

    ```
    git clone 'https://github.com/igcondapet/thyroidiomics.git'
    cd thyroidiomics
    conda env create --file environment.yml
    ```
    The last step above creates a conda environment named `thyroidiomics_env`. Make sure you have conda installed. Next, activate the conda environment
    ```
    conda activate thyroidiomics_env
    ```

- **Define dataset location and create datainfo.csv file.** Go to [config.py](config.py) and set path to data folders for your (possibly multi-institutional) scintigraphy datasets. 
    ```
    THYROIDIOMICS_FOLDER = '' # path to the directory containing `data` and `results` (this will be created by the pipeline) folders.
    DATA_FOLDER = os.path.join(THYROIDIOMICS_FOLDER, 'data', 'nifti') # place your data in this location
    ```
    The directory structure within `THYROIDIOMICS_FOLDER` should be as shown below. The folders `images` and `labels` under `THYROIDIOMICS_FOLDER/data/nifti` must contain all the 2D scintigraphy images and ground truth segmentation labels from all the multi-institutional datasets in `.nii.gz` format with each image-label pair given the same filenames. Other folders containing the results of segmentation (`THYROIDIOMICS_FOLDER/segmentation_results`) and classification (`THYROIDIOMICS_FOLDER/classification_results`) steps will be created in the subsequent steps below.

        └───THYROIDIOMICS_FOLDER/
            ├──data/nifti/
            │  ├── images
            │  │   ├── Patient0001.nii.gz
            │  │   ├── Patient0002.nii.gz
            │  │   ├── ...
            │  ├── labels
            │  │   ├── Patient0001.nii.gz
            │  │   ├── Patient0002.nii.gz 
            │  │   ├── ...
            ├──segmentation_results
            └──classification_results
        

    Next, create a file named `datainfo.csv` containing information about `PatientID` (corresponding to image filenames), `CenterID` and `Class` as shown below. For this work, we had 3 classes corresponding to 3 thyroid pathologies: MNG (label=0), TH (label=1) and DG (label=2).
    ```
    PatientID,CenterID,Class
    Patient0001,A,0
    Patient0002,B,1
    Patient0003,D,2
    Patient0004,C,2
    Patient0005,I,1
    ...
    ...
    ...
    ```
    Place the `datainfo.csv` file in this location: `thyroidiomics/data_analysis/datainfo.csv`.
    
- **Run segmentation training.** The file [./segmentation/train.py](./segmentation/train.py) runs training on the 2D dataset via PyTorch's `DistributedDataParallel`. To run training, do the following (an example bash script is given in [./segmentation/train.sh](./segmentation/train.sh)). 
    ```
    cd thyroidiomics/segmentation
    CUDA_VISIBLE_DEVICES=0 torchrun --standalone --nproc_per_node=1 train.py --network-name='unet1' --leave-one-center-out='A' --epochs=300 --input-patch-size=64 --inference-patch-size=128 --train-bs=32 --num_workers=4 --lr=2e-4 --wd=1e-5 --val-interval=2 --sw-bs=4 --cache-rate=1
    ```
    A unique experimentID will be created using the `network_name` and `leave_one_center_out`. For example, if you used `unet1` and choose leave-one-center-out (loco) center as `A` for testing, this experiment will be referenced as `unet1_locoA` under the results folders. Set `--nproc_per_node` as the number of GPU nodes available for parallel training. The data is cached using MONAI's `CacheDataset`, so if you are running out of memory, consider lowering the value of `cache_rate`. During training, the training loss and validation DSC are saved under `THYROIDIOMICS_FOLDER/segmentation_results/logs/trainlog_gpu{rank}.csv` and `THYROIDIOMICS_FOLDER/segmentation_results/logs/validlog_gpu{rank}.csv` where `{rank}` is the GPU rank and updated every epoch. The checkpoints are saved every `val_interval` epochs under `THYROIDIOMICS_FOLDER/segmentation_results/models/model_ep{epoch_number}.pth`. There are many networks defined under the `get_model()` method in the file [./segmentation/initialize_train.py](./segmentation/initialize_train.py), but in this work, we used the network `unet1` as that had the best performance. 

- **Run segmentation evaluation on test set.** After the training is finished (for a given experimentID), [./segmentation/predict.py](./segmentation/predict.py) can be used to run evaluation on the test set (which consists of all the data left out in LOCOCV scheme, here `CenterID=A` from the above example) and save the predictions, testmetrics and visualization of predicted results. To run test evaluation, do the following (an example bash script is given in [./segmentation/predict.sh](./segmentation/predict.sh)).
    ```
    cd thyroidiomics/segmentation
    python predict.py --network-name='unet1' --leave-one-center-out='A' --inference-patch-size=128 --num_workers=2  --sw-bs=2 --val-interval=2
    ```
    [./segmentation/predict.py](./segmentation/predict.py) uses the model with the highest DSC on the validation set for test evaluation. CAUTION: set `--val-interval` to the same value that was used during training.

- **Run classification training and evaluation.** In this step, we will use [./classification/classification.py](./classification/classification.py) file to train a classfication model, and perform testing in two ways. The training will run on all the centers except the one defined by `--leave-one-center-out`, while testing will be performed only on the center defined by `--leave-one-center-out`. In scenario 1 of testing, we extract features from the test images using the physician's thyroid annotation, while in scenario 2, we extract them from the segmentation model's predicted annotations (see Fig. 1 above). To run classification training and evaluation, do the following (an example bash script is given in [./classification/classification_train_predict.sh](./classification/classification_train_predict.sh)). Remember, these experiments are also referenced using the same experimentID as for the segmentation step and the results are save accordingly.
    ```
    cd thyroidiomics/classification
    python classification.py --network-name='unet1' --leave-one-center-out='A'
    ```

- **Saved results from segmenation and classification.** The segmentation training logs, trained models, predictions, testmetrics and segmentation visualization are stored under folders `logs`, `models`, `predictions`, `testmetrics` and `visualization` created under the location `THYROIDIOMICS_FOLDER/segmentation_results` referenced by their unique experimentIDs. For the classification step, the extracted features and predicted metrics are stored under the folders `feature_extraction` and `prediction_and_metrics` created under the location `THYROIDIOMICS_FOLDER/classification_results` again referenced by the same experimentIDs as the previous step. 

# References

<a id="1">[1]</a> 
Ahamed, S., et al., "Comprehensive Evaluation and Insights into the Use of Deep Neural Networks to Detect and Quantify Lymphoma Lesions in PET/CT Images", arXiv:2311.09614 (2023). 

<a id="2">[2]</a> 
Sabouri, M., et al., "Myocardial Perfusion SPECT Imaging Radiomic Features and Machine Learning Algorithms for Cardiac Contractile Pattern Recognition", Journal of Digital Imaging, v36, p497-509 (2022). 