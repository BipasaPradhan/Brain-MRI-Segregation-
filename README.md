# Brain Tumor MRI Segmentation

## Introduction

Magnetic Resonance Imaging (MRI) is one of the main tools doctors use to detect and diagnose brain tumors. Before a scan can be analyzed, the tumor has to be found and separated from the rest of the image. Doing it by hand is slow and depends on the radiologist's experience. Raw scans also make it harder: they contain sensor noise, the skull and scalp show up as bright regions that look like tumor tissue, and some tumors have very low contrast against the healthy brain around them.

### Objective

This project builds an image-processing pipeline that automatically cleans an MRI scan, removes the skull, and segments the tumor region. It then creates heatmaps that highlight the tumor's boundaries and internal structure, so the results are easy to interpret.

The pipeline uses only **classical image-processing techniques**: wavelet transforms, thresholding, morphological operations and region growing.

## Overview

The pipeline processes each scan through these steps:

1. **Standardize:** convert the image to grayscale and resizing it to `224 × 224`.

2. **Denoise:** applyin wavelet denoising with tuned parameters.

3. **Strip the skull:** separate brain tissue from the skull using thresholding and morphological operations.

4. **Segment the tumor:** use seeded region growing to extract a binary tumor mask.

5. **Visualize:** produce gradient-magnitude, texture-density and distance-map heatmaps overlaid on the original scan.

The pipeline is demonstrated on two cases: a scan where the tumor is **clearly visible**, and a **low-contrast** scan where it's hard to make out.

## Dataset

[Brain MRI Images for Brain Tumor Detection](https://www.kaggle.com/datasets/navoneel/brain-mri-images-for-brain-tumor-detection) (Kaggle): downloaded automatically with `kagglehub`.

- `yes/`: MRI scans with a tumor (154 images processed)
- `no/`: MRI scans without a tumor

## Tech Stack

| Library | Used for |
|---|---|
| **OpenCV** | Image I/O, thresholding, morphology, contours, colormaps |
| **SimpleITK** | Region-growing segmentation, gradient magnitude, distance maps |
| **PyWavelets** | Discrete wavelet transform for denoising |
| **NumPy / SciPy** | Array operations |
| **Matplotlib** | Visualization |
| **kagglehub** | Dataset download |

## Pipeline Details

### 1. Preprocessing
Each image is converted to grayscale and resized to `224 × 224` so every scan in the dataset has the same input size.

### 2. Wavelet Denoising
The image goes through a single-level 2D discrete wavelet transform. The detail coefficients (LH, HL, HH) are **soft-thresholded** and then the image is reconstructed.

To pick the parameters, the notebook runs a grid search over 3 wavelets (`bior1.3`, `bior3.5`, `db2`) × 3 thresholds (10, 30, 60). It then inspects the **residual noise map**, which is the difference between the original and denoised images. If the tumor's outline shows up in the residual, the threshold is removing real structure rather than noise.

**Chosen setting:** `bior3.5` with threshold `30`. It removes noise without wiping out tumor edges.

### 3. Skull Stripping
* Binary thresholding (a fixed threshold for clear scans, **Otsu's method** for low-contrast scans)
* Morphological closing to fill gaps
* Largest-contour extraction to isolate the head region
* Gaussian smoothing, erosion, and **connected-component analysis** to break off and discard leftover skull fragments, followed by dilation to restore the brain's size
* Edge cropping to remove border artifacts

### 4. Tumor Segmentation
* **Seed selection:** the brightest pixel inside the brain, with a safety margin from the border. For low-contrast scans, the search is limited to the region of interest.
* **Region growing:** SimpleITK's `ConnectedThreshold` grows outward from the seed and includes neighboring pixels whose intensity falls in the tumor range (e.g. 150–255).

### 5. Visualization
* **Gradient magnitude** (recursive Gaussian, σ = 2): highlights the tumor's boundaries.
* **Texture-density heatmap:** local standard deviation inside the tumor, showing how varied its internal structure is.
* **Distance-map heatmap:** a signed Maurer distance transform showing how far each point is from the tumor's edge, which highlights its core.

Heatmaps are color-mapped and overlaid on the original MRI for easy reading.

## Getting Started

1. Open `Image_Project.ipynb` in `Google Colab` or a local `Jupyter environment`.

2. Install dependencies if needed:
   ```bash
   pip install opencv-python SimpleITK PyWavelets scipy matplotlib kagglehub
   ```
   
3. Run all cells. The dataset downloads automatically.

> **Running locally:** the notebook uses `google.colab.patches.cv2_imshow`. Outside Colab, replace it with `plt.imshow(...)` from Matplotlib.

## Limitations & Future Work

* Some parameters (intensity thresholds, seed region) are tuned per image. A fully automatic pipeline would need adaptive thresholds or automatic seed detection.

* Results are checked visually with no ground truth metrics.

* The pipeline could serve as preprocessing for a deep learning model (e.g. a U-Net) to compare classical and learned segmentation.