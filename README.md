# Cerebrum-7T nnU-Net Pipeline

This repository provides an end-to-end pipeline for brain segmentation using nnU-Net (v2) on ultra-high field 7T MRI data, specifically utilizing the Cerebrum-7T dataset. The pipeline encompasses installation, preprocessing, training, inference, visualization, and basic statistical analysis. It also includes modifications (e.g., patching `blosc2.open`) to ensure compatibility in the Google Colab environment.

## Table of Contents

1. [Introduction](#introduction)
2. [Installation](#installation)
   - [Clone this Repository](#clone-this-repository)
   - [Install Dependencies](#install-dependencies)
   - [Google Drive Setup](#google-drive-setup)
3. [Dataset](#dataset)
   - [Dataset Source](#dataset-source)
   - [Folder Structure](#folder-structure)
4. [Environment Setup](#environment-setup)
5. [Preprocessing](#preprocessing)
6. [Training](#training)
7. [Inference](#inference)
8. [Visualization and Statistical Analysis](#visualization-and-statistical-analysis)
   - [Load & Visualize a Predicted Segmentation](#load--visualize-a-predicted-segmentation)
   - [Basic Statistical Analysis](#basic-statistical-analysis)
9. [Modifications and Customizations](#modifications-and-customizations)
   - [blosc2 Patch](#blosc2-patch)
   - [Environment Variables](#environment-variables)
   - [Dataset Loader](#dataset-loader)
   - [Training and Inference](#training-and-inference)
10. [Results](#results)
11. [Troubleshooting](#troubleshooting)
12. [Citation](#citation)
13. [License](#license)

## Introduction

The Cerebrum-7T project leverages nnU-Net to provide a fully automated, self-configuring segmentation pipeline for 7T brain MRI data. This pipeline:

- Automatically extracts dataset fingerprints and creates customized training plans.
- Trains both 3D full-resolution and 2D models using cross-validation.
- Supports inference and visualization of predicted segmentation masks.
- Enables basic statistical analysis (e.g., calculating volumes of segmented brain regions).

The pipeline is designed to run in Google Colab, with data stored on Google Drive. It includes modifications to ensure compatibility and memory efficiency.

## Installation

### Clone this Repository

```bash
git clone https://github.com/your_username/Cerebrum7T_nnUNet.git
cd Cerebrum7T_nnUNet

Install Dependencies:

In your Colab notebook or local environment, install the required packages:

python
Copy
!pip install nnunetv2
!pip install monai
!pip install nibabel
!pip install blosc2
Google Drive Setup:

Make sure your data and results folders are on your Google Drive. In Colab, mount Google Drive:

python
Copy
from google.colab import drive
drive.mount('/content/drive')
Dataset
Dataset Source
The dataset used in this pipeline is the Cerebrum-7T dataset, which is designed for brain segmentation from 7T MRI. The dataset contains:

High-resolution T1-weighted MRI scans (obtained via MP2RAGE sequence).

Multiple segmentation masks obtained using FreeSurfer and other tools.

Data in BIDS format.

Folder Structure
The dataset should follow the nnU-Net raw folder structure. For example:

bash
Copy
nnUNet_raw/
├── Dataset123_Cerebrum7T/
│   ├── imagesTr/        # Training images (e.g., sub-001.nii.gz, sub-002.nii.gz, …)
│   ├── labelsTr/        # Corresponding ground truth segmentation masks
│   └── imagesTs/        # Test images for inference (optional)
Preprocessed data (generated by nnU-Net planning and preprocessing) will be stored in:

Copy
nnUNet_preprocessed/
├── Dataset123_Cerebrum7T/
│   ├── splits_final.json
│   ├── nnUNetPlans.json
│   └── ... (other preprocessed files)
Environment Setup
Set the following environment variables to point to the appropriate folders on your Google Drive:

python
Copy
import os

os.environ["nnUNet_raw_data_base"] = "/content/drive/MyDrive/nnUNet_raw_data"
os.environ["nnUNet_preprocessed"] = "/content/drive/MyDrive/nnUNet_preprocessed"
os.environ["RESULTS_FOLDER"] = "/content/drive/MyDrive/nnUNet_results"

# Also define these additional variables required by nnU-Net:
os.environ["nnUNet_raw"] = "/content/drive/MyDrive/nnUNet_raw_data"
os.environ["nnUNet_results"] = "/content/drive/MyDrive/nnUNet_results"

# Create directories if they don't exist:
os.makedirs(os.environ["nnUNet_raw_data_base"], exist_ok=True)
os.makedirs(os.environ["nnUNet_preprocessed"], exist_ok=True)
os.makedirs(os.environ["RESULTS_FOLDER"], exist_ok=True)
Preprocessing
Run the following command in a Colab cell to plan and preprocess your dataset. This extracts dataset fingerprints and creates training plans automatically.

python
Copy
def plan_and_preprocess():
    command = f"nnUNetv2_plan_and_preprocess -d 123 --verify_dataset_integrity -np 3"
    print("Running experiment planning and preprocessing:")
    print(command)
    process = subprocess.Popen(command, shell=True, stdout=subprocess.PIPE, stderr=subprocess.STDOUT, text=True)
    for line in process.stdout:
        print(line, end='')
    process.wait()
    if process.returncode != 0:
        raise Exception("Experiment planning and preprocessing failed.")
    print("Planning and preprocessing completed.")

# Run the preprocessing function
plan_and_preprocess()
Training
The pipeline trains both 3D full-resolution and 2D models across 5 cross-validation folds. An example training cell is:

python
Copy
def train_model(configuration, fold):
    command = f"nnUNetv2_train {DATASET_NAME} {configuration} {fold} --npz"
    print(f"Training {configuration} model on fold {fold} with command:")
    print(command)
    subprocess.run(command, shell=True, check=True)
    print(f"Training for {configuration} fold {fold} completed.")

for fold in range(5):
    train_model("3d_fullres", fold)
    train_model("2d", fold)
Make sure to adjust the device settings if training on CPU/GPU.

Inference
After training, run inference on the test dataset using the trained model:

python
Copy
def run_inference(configuration):
    command = (
        f"nnUNetv2_predict -i {TEST_IMAGES_FOLDER} -o {OUTPUT_PREDICTIONS_FOLDER} "
        f"-d 123 -c {configuration} --save_probabilities"
    )
    print(f"Running inference for configuration {configuration} with command:")
    print(command)
    subprocess.run(command, shell=True, check=True)
    print(f"Inference for configuration {configuration} completed.")

# Run inference using the 3D full resolution model
run_inference("3d_fullres")
Visualization and Statistical Analysis
Load & Visualize a Predicted Segmentation
python
Copy
# Cell 7: Load & Visualize a Predicted Segmentation
predicted_file = os.path.join(OUTPUT_PREDICTIONS_FOLDER, "sub-001_0000.nii.gz")  # Adjust filename as needed
if os.path.exists(predicted_file):
    seg_data, seg_affine = load_nifti_image(predicted_file)
    visualize_slice(seg_data, title="Predicted Segmentation")
else:
    print(f"Predicted file {predicted_file} not found. Please check your inference output folder.")
Basic Statistical Analysis
python
Copy
# Cell 8: Basic Statistical Analysis Example
ground_truth_file = os.path.join(os.environ["nnUNet_raw_data_base"], "Dataset123_Cerebrum7T", "labelsTr", "sub-001.nii.gz")
predicted_file = os.path.join(OUTPUT_PREDICTIONS_FOLDER, "sub-001_0000.nii.gz")  # Adjust as needed

if os.path.exists(ground_truth_file) and os.path.exists(predicted_file):
    gt_data, _ = load_nifti_image(ground_truth_file)
    pred_data, _ = load_nifti_image(predicted_file)
    label = 1  # For example, label 1 may represent gray matter.
    gt_volume = compute_region_volume(gt_data, label)
    pred_volume = compute_region_volume(pred_data, label)
    print(f"Ground truth volume for label {label}: {gt_volume} voxels")
    print(f"Predicted volume for label {label}: {pred_volume} voxels")
else:
    print("Ground truth or predicted file not found. Please verify the file paths.")
Modifications and Customizations
blosc2 Patch:
To work around issues with memory mapping in the Colab environment, the script patches blosc2.open to force mmap_mode=None.

Environment Variables:
The code sets environment variables required by nnU-Net for raw data, preprocessed data, and results storage.

Dataset Loader:
Custom PreprocessedNnunetDataset and SliceDataset classes are provided to load your nnU-Net preprocessed data from Google Drive and convert 3D volumes into 2D slices for training with MONAI.

Training and Inference:
Standard nnU-Net commands are used for training and inference, while the MONAI example (provided in separate cells) demonstrates how to switch to an alternative architecture if lower memory consumption is desired.

Results
After training and inference, you should observe:

Preprocessed dataset fingerprints and training plans saved in the nnUNet_preprocessed folder.

Training logs indicating progress across folds.

Inference outputs (e.g., segmentation masks in NIfTI format) saved in the OUTPUT_PREDICTIONS_FOLDER.

Visualization of predicted segmentation overlays and basic statistical metrics (e.g., region volumes) printed in the notebook.

Example results (for visualization and volume computation) will be displayed in the notebook cells.

Troubleshooting
Killed Process / OOM:
If your process is killed due to insufficient memory, try reducing the number of data augmentation workers or use a GPU if available.

File Not Found Errors:
Ensure that all environment variables and paths are correctly set and that your dataset follows the expected folder structure.

Blosc2 Reshaping Errors:
If reshaping fails, verify the actual data shape stored in your blosc2 files and adjust the target shape in the loader accordingly.

Citation
If you use this pipeline or nnU-Net in your research, please cite:

Isensee, F., Jaeger, P. F., Kohl, S. A., Petersen, J., & Maier-Hein, K. H. (2021). nnU-Net: a self-configuring method for deep learning-based biomedical image segmentation. Nature Methods, 18(2), 203–211.

License
This project is licensed under the MIT License. See the LICENSE file for details.

