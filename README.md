# CAC-QPI Experimental Reconstruction Pseudocode

This repository provides manuscript-consistent pseudocode and parameter definitions for the experimental reconstruction workflow of **closed-form aberration-corrected quantitative phase imaging (CAC-QPI)** described in:

> **Non-Iterative Quantitative Phase Imaging with Aberration Correction via Angular and Focal Modulation**

The repository is intended to make the reconstruction procedure, calibration requirements, and numerical settings transparent and reproducible at the algorithm-specification level. It contains detailed pseudocode rather than executable source code. Raw experimental data, forward simulations, synthetic-object generation, and simulation-only validation modules are not included.

## Measurement scheme

Each CAC-QPI dataset contains four experimentally acquired intensity images:

1. two on-axis images recorded at symmetric defocus positions $+\Delta z$ and $-\Delta z$;
2. two weighted multiplexed off-axis illumination (OAI) images.

Each OAI exposure contains two mutually incoherent point sources separated by $90^\circ$, with a calibrated optical-intensity ratio of $1:2$. The second OAI pattern is rotated by $45^\circ$ relative to the first. Under the reference angular convention,

$$
\mathcal S_1=\{0^\circ,90^\circ\},
\qquad
\mathcal S_2=\{45^\circ,135^\circ\}.
$$

A common rotation of all azimuths is permitted when a different angular origin is used for the calibrated illumination coordinate system. Reconstruction must use the calibrated illumination wavevectors and optical-intensity weights rather than nominal angle or drive-current values alone.

## Repository contents

| File | Description |
|---|---|
| [`Algorithm.md`](Algorithm.md) | Detailed end-to-end pseudocode for preprocessing, even- and odd-aberration recovery, corrected transfer-function construction, and final quantitative phase reconstruction |
| [`Reconstruction_parameters.md`](Reconstruction_parameters.md) | Acquisition parameters, calibration quantities, Zernike modes, numerical settings, and the minimum reproducibility record |
| [`CITATION.cff`](CITATION.cff) | Repository and manuscript citation metadata |
| [`LICENSE`](LICENSE) | MIT License for the documentation and pseudocode |

## Reconstruction outline

The documented workflow comprises the following stages:

1. dark correction, exposure/brightness normalization, cropping, and Fourier transformation of the four measured images;
2. recovery of the even-symmetric pupil aberration from the symmetric-defocus pair;
3. recovery of the defocus-derived common spectrum;
4. construction of geometry-defined OAI single-branch masks from the calibrated pupil support and illumination wavevectors;
5. reliability screening within the geometry-defined single-branch regions;
6. recovery of the odd-symmetric pupil aberration from the two multiplexed OAI measurements;
7. synthesis of the total pupil phase and aberration-corrected phase transfer functions;
8. final least-squares/Wiener hybrid phase reconstruction using the defocus and OAI measurements.

The geometry-defined single-branch regions are constructed directly from the calibrated binary pupil support and illumination wavevectors using the exact branch-count criterion $N_m(\mathbf u)=1$. No pupil erosion, circular-radius approximation, or empirically adjusted guard band is used in the documented workflow. Amplitude-, noise-, and sensitivity-based screening is applied only within these geometry-defined regions and does not alter their branch boundaries.

The single-branch masks are used for odd-aberration estimation. OAI frequencies outside these masks may still contribute to the final full-band phase reconstruction through the corrected multiplexed OAI transfer functions.

## Method conventions

- The pupil aberration is represented by 12 real Zernike modes in Noll ordering.
- The even-mode set is $\mathcal E=\{4,5,6,11,12,13,14,15\}$.
- The odd-mode set is $\mathcal O=\{7,8,9,10\}$.
- Noll modes 1–3 are excluded: piston cancels from the measurable relative phase, and pupil tilt lies in the null space of the odd-aberration finite-difference relation.
- Zernike coefficients are estimated by direct linear least squares after periodic-branch resolution.
- The reconstruction does not evaluate a complex logarithm and does not perform joint nonlinear optimization of the object and pupil.
- The final phase is recovered with the aberration-corrected least-squares/Wiener hybrid deconvolution specified in [`Algorithm.md`](Algorithm.md).

The associated manuscript is the authoritative source for the physical model, notation, and validation. This repository follows the manuscript when describing the experimental reconstruction workflow.

## Required calibration and metadata

At minimum, reconstruction requires the following calibrated or recorded quantities:

- illumination wavelength and bandwidth;
- objective numerical aperture;
- magnification and camera pixel size;
- exact symmetric-defocus positions;
- calibrated pupil support, center, and radius;
- OAI illumination wavevectors and optical-intensity weights;
- exposure times, brightness-normalization factors, and dark frames;
- image dimensions, crop, and Fourier-transform convention;
- numerical thresholds, regularization values, and phase-reference convention.

Dataset-specific values and reference numerical settings are listed in [`Reconstruction_parameters.md`](Reconstruction_parameters.md).

## Expected outputs

For reproducibility, an implementation should save at least:

- the eight even and four odd Zernike coefficients;
- the recovered even, odd, and total pupil-phase maps;
- the reliable-frequency mask used for the defocus-pair calculation;
- all OAI geometry and final single-branch masks;
- the aberration-corrected defocus and OAI phase transfer functions;
- the reconstructed quantitative phase spectrum and phase map;
- the calibration values and numerical parameters used for the dataset;
- the design-matrix rank and condition-number diagnostics.

## Citation

If you use these materials, please cite the associated manuscript and this repository. Citation metadata are provided in [`CITATION.cff`](CITATION.cff) and through GitHub's **Cite this repository** function.

Repository:

<https://github.com/XBEN-z/CAC-QPI-reconstruction-pseudocode>

## License

The documentation and pseudocode are distributed under the MIT License. See [`LICENSE`](LICENSE).
