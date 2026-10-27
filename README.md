# Automated Hybrid Pipeline for Vitreous Hyperreflective Dot Detection in OCT B-Scans

A documentation-first overview of a hybrid computer-vision pipeline for **candidate vitreous hyperreflective dot (VHD) localization** in optical coherence tomography (OCT) B-scans. The pipeline combines a SegFormer-based retinal segmentation with an intensity-derived boundary estimate, fuses the two estimates to define a vitreous region of interest (ROI), and applies local contrast and robust-statistics-based candidate detection.

> **Research status:** This repository documents the pipeline and illustrative outputs. Candidate detections are not ground-truth labels or a clinical diagnosis. Quantitative performance should only be reported together with its metric, reference-annotation protocol, and evaluation set. The manuscript is in preparation / under internal review; publication details will be added when appropriate.
>
> **Data and release note:** Do not publish OCT examples, source code, model weights, or other project materials until you have confirmed that the relevant PI/institution permits their release. The sample image included here is for local README preparation only; replace it with an approved de-identified example or remove it before pushing to a remote repository if permission has not been obtained.

## Pipeline overview

![Overview of the hybrid OCT VHD candidate-detection pipeline](assets/pipeline_overview.svg)

The two boundary-estimation branches operate on the same grayscale B-scan. Their outputs are fused and regularized before the detector searches only within the resulting vitreous ROI.

## Representative output

![Illustrative OCT B-scan and candidate detections](assets/example_output.png)

*Illustrative output from the current pipeline. Markers represent algorithm-generated candidates, not independently confirmed VHDs. This image must not be publicly distributed without the required data-release approval.*

## Method

### A. SegFormer-based retinal localization

The grayscale B-scan is resized, while preserving its aspect ratio, and padded to **384 × 384** for the pretrained segmentation model. Predicted retinal classes **1–11** are combined into a binary retinal mask. For each column, the first foreground pixel is used to obtain an initial boundary profile, denoted by $d(x)$, after mapping the mask back to the original image dimensions.

### B. Intensity-based boundary estimate

A second boundary estimate is derived independently from the original grayscale scan. Non-local means denoising is followed by Otsu thresholding and extraction of the largest contour. The uppermost foreground pixel in each column forms the intensity-based profile $m(x)$. This estimate provides a complementary signal when the segmentation boundary is locally unreliable.

### C. Boundary fusion and envelope regularization

Missing profile values are interpolated. Let the global baseline be the median of the segmentation profile:

$$
 b = \operatorname{median}_{x} d(x).
$$

The current implementation uses the following deviation-gated fusion rule:

$$
 f(x)=\begin{cases}
 d(x), & b-m(x)>60,\\
 \min\bigl(d(x),m(x)\bigr), & \text{otherwise}.
 \end{cases}
$$

A quadratic curve is fitted to selected points in the central 70% of the fused profile. The final envelope is the pointwise minimum of the fused profile and the fitted curve, followed by a **15-column median filter**. This produces the envelope $e(x)$ used to constrain the ROI. The numerical rule and parameters above describe the current implementation and should be kept synchronized with the version evaluated in any reported experiment.

### D. Vitreous ROI

The detector is restricted to pixels above the final envelope, excluding a **20-pixel safety margin** adjacent to the boundary. The current configuration sets the top-artifact buffer to **0 pixels**. This setting should be reported if results depend on it, and must match the configuration used to generate example outputs.

### E. Local candidate detection

The scan is pre-smoothed with a **3 × 3 Gaussian filter**. The ROI is then traversed with **40 × 40-pixel windows** and a **20-pixel stride** (50% overlap). A Gaussian background estimate with $\sigma=7$ is subtracted from each window to obtain a high-pass residual:

$$
 H = I_{\mathrm{smooth}} - G_{\sigma=7}(I_{\mathrm{smooth}}).
$$

For the valid vitreous pixels in each window, let $H_V$ denote the corresponding residual values. The local threshold is:

$$
 T = \operatorname{median}(H_V) + 3.5\times1.4826\times
 \max\bigl(\operatorname{MAD}(H_V),3\bigr),
$$

where

$$
 \operatorname{MAD}(H_V)=\operatorname{median}\!\left(
 \left|H_V-\operatorname{median}(H_V)\right|\right).
$$

A pixel is a candidate only when its residual exceeds $T$, its original intensity exceeds **15**, and it lies within the vitreous mask. A window is retained only when candidate pixels make up **less than 5%** of its valid vitreous pixels; windows at or above that fraction are rejected. Overlapping candidate maps are combined by bitwise OR.

### F. Candidate refinement and logging

The combined binary map is processed with a **2 × 2 elliptical morphological opening**. External contours are then filtered using the current criteria:

| Filter | Acceptance criterion |
|---|---:|
| Contour area $A$ | $3 < A < 120$ pixels |
| Circularity | $> 0.40$ |
| Aspect ratio | $< 3.0$ |

Circularity is computed as

$$
 C=\frac{4\pi A}{P^2},
$$

where $A$ is contour area and $P$ is perimeter. The aspect ratio is the larger of the bounding-box width-to-height and height-to-width ratios. These are screening rules, not proof that a candidate is a true VHD.

The implementation records candidate coordinates, contour measurements, original-image intensity, acceptance/rejection verdicts, and rejection reasons in a CSV log. It also produces per-scan candidate counts and an overlay for visual inspection.

## Current implementation parameters

| Component | Current setting |
|---|---|
| Segmentation input | 384 × 384, aspect ratio preserved with padding |
| Retinal classes merged | 1–11 |
| Boundary deviation gate | 60 pixels |
| Central curve-fitting interval | 15%–85% of image width |
| Boundary smoothing | 15-column median filter |
| Vitreous safety margin | 20 pixels |
| Top-artifact buffer | 0 pixels in the current configuration |
| Detection window / stride | 40 × 40 / 20 pixels |
| Local background estimate | Gaussian blur, $\sigma=7$ |
| MAD multiplier / floor | 3.5 / 3.0 |
| Original-intensity floor | 15 |
| Window candidate-fraction gate | Reject at $\geq 5\%$ |
| Morphological opening | 2 × 2 elliptical kernel |
| Contour filters | $3 < A < 120$, circularity $>0.40$, aspect ratio $<3.0$ |

## Outputs

- A visual overlay of the original OCT B-scan and candidate locations.
- A per-scan count of retained candidates.
- A CSV log containing candidate positions, contour geometry, intensity, verdict, and rejection reason.

## Scope and limitations

- The output represents **candidate VHDs**, not a confirmed clinical label for each dot.
- Visual examples alone do not establish detection accuracy or clinical utility.
- Quantitative evaluation should specify the number and source of scans, how expert reference annotations were produced, the spatial matching tolerance, and the reported metrics.
- Generalization across scanners, image quality, and institutions remains a separate question and should not be inferred from one example or one site.
- Raw medical data, trained weights, and the full implementation are not included in this documentation-only snapshot. Their availability depends on project and institutional approval.

## Repository structure

```text
automated-hybrid-vhd-detection/
├── README.md
└── assets/
    ├── pipeline_overview.svg
    └── example_output.png
```

## Citation

Manuscript: *Automated Hybrid Pipeline for Vitreous Hyperreflective Dot Detection in OCT B-Scans* (in preparation / under internal review). Replace this note with the formal citation and DOI once the work is publicly available.
