# CAC-QPI Reconstruction Parameters

This file defines the parameter conventions, reported system settings, and dataset-specific records required to implement the experimental CAC-QPI reconstruction described in `Algorithm.md`.

The manuscript is authoritative. Values identified as **reference implementation defaults** are numerical safeguards derived from the internal validation implementation and are not replacements for dataset-specific calibration.

---

## 1. Experimental acquisition

| Parameter | Value or rule | Status |
|---|---:|---|
| Number of raw images | 4 | Manuscript-fixed |
| Symmetric-defocus images | 2 | Manuscript-fixed |
| Weighted multiplexed OAI images | 2 | Manuscript-fixed |
| Objective | $100\times/0.8\ \mathrm{NA}$ dry objective | Reported system |
| Camera pixel size | $6.5\ \mu\mathrm m$ | Reported system |
| Nominal symmetric defocus | $+\!0.5\ \mu\mathrm m$ and $-\!0.5\ \mu\mathrm m$, referenced to their midpoint | Reported acquisition/simulation setting |
| OAI illumination NA | 0.7 | Reported setting |
| OAI pattern 1 | two sources separated by $90^\circ$ | Manuscript-fixed |
| OAI pattern 2 | pattern 1 rotated by $45^\circ$ | Manuscript-fixed |
| Source-weight ratio in each OAI pattern | $1:2$ | Manuscript-fixed |
| Reference azimuths | $(0^\circ,90^\circ)$ and $(45^\circ,135^\circ)$ | Coordinate convention |
| Resolution-target wavelength | 463 nm | Reported experiment |
| Dental-plaque wavelength | 463 nm | Reported experiment |
| Buccal-cell wavelength | 628 nm | Reported experiment |

A common rotation of all OAI azimuths is permitted if the LED array uses another angular origin. The calibrated wavevectors, not the nominal angle labels, must be used in reconstruction.

The $1:2$ values are relative optical intensity weights. Before reconstruction, correct them for exposure time and measured source brightness.

---

## 2. Dataset-specific values that must be recorded

The following fields must be filled for every experimental dataset:

| Field | Required entry |
|---|---|
| Image width and height | pixel count |
| Camera bit depth and data type | e.g. 16-bit unsigned integer |
| Magnification used for the dataset | measured or nominal |
| Effective object-plane pixel size | camera pixel size divided by magnification |
| Exact $+\Delta z$ and $-\Delta z$ positions | calibrated stage positions |
| Illumination wavelength and bandwidth | measured or filter specification |
| Objective NA | calibrated/nominal value |
| OAI source coordinates | transverse wavevectors or calibrated LED positions |
| OAI source weights | calibrated intensity weights after normalization |
| Exposure times | one value per image |
| Dark frame | one per acquisition setting or a documented shared frame |
| Crop/FOV | pixel coordinates |
| Calibrated binary pupil support and frequency-grid registration | full binary support plus its center and sampling in Fourier coordinates |
| Fourier-transform normalization convention | explicitly documented |
| Phase reference | normally zero DC / zero spatial mean |

---

## 3. Illumination patterns

Using the reference azimuth convention:

$$
\mathcal S_1=
\left\{
(\theta,A)=(0^\circ,1),(90^\circ,2)
\right\},
$$

$$
\mathcal S_2=
\left\{
(\theta,A)=(45^\circ,1),(135^\circ,2)
\right\}.
$$

For numerical use, normalize the two weights within each exposure if the images have already been normalized to equal total incident intensity:

$$
(1,2)\longrightarrow
\left(\frac13,\frac23\right).
$$

Do not normalize again if the calibrated weights are already expressed in the intensity scale used by the measured OAI spectrum.

---

## 4. Zernike basis

Use orthonormal real Zernike modes in Noll ordering.

### 4.1 Estimated even modes

| Noll index | Conventional mode |
|---:|---|
| 4 | Defocus |
| 5 | Oblique astigmatism |
| 6 | Vertical astigmatism |
| 11 | Primary spherical aberration |
| 12 | Secondary astigmatism component |
| 13 | Secondary astigmatism component |
| 14 | Quadrafoil component |
| 15 | Quadrafoil component |

Index set:

$$
\mathcal E=\{4,5,6,11,12,13,14,15\}.
$$

### 4.2 Estimated odd modes

| Noll index | Conventional mode |
|---:|---|
| 7 | Vertical coma |
| 8 | Horizontal coma |
| 9 | Oblique trefoil |
| 10 | Vertical trefoil |

Index set:

$$
\mathcal O=\{7,8,9,10\}.
$$

### 4.3 Excluded modes

| Noll index | Reason |
|---:|---|
| 1 | Piston cancels from the measurable relative even phase |
| 2–3 | Pupil tilt lies in the null space of the odd finite-difference operator |

Total estimated coefficients: 12.

Noll mode 4 represents the static defocus component of the system pupil. The calibrated $+\Delta z$ and $-\Delta z$ measurement diversity is fixed in the forward model and is not fitted as an unknown modal coefficient.

---

## 5. Image preprocessing

| Item | Rule |
|---|---|
| Dark correction | subtract measured dark frame before Fourier transformation |
| Exposure normalization | divide by exposure time and calibrated brightness factor |
| DC handling | exclude the DC coefficient from aberration estimation |
| Array consistency | all four images must share dimensions, crop, sampling, and Fourier convention |
| Registration | apply only when experimentally required; document the transform and interpolation |
| Intensity clipping | do not clip valid linear-camera data before normalization |

---

## 6. Reliable-frequency selection

In the reported workflow, reliability screening is applied within the geometry-defined supports using spectral magnitude, noise level, and transfer sensitivity.

### 6.1 Even-aberration domain

$$
\Omega_e
=
\Omega_{\mathrm{geom},e}
\cap
\Omega_{\mathrm{amp},e}
\cap
\Omega_{\mathrm{sens},e}.
$$

- $\Omega_{\mathrm{geom},e}$: common native passband, excluding DC.
- $\Omega_{\mathrm{amp},e}$: both defocus spectra exceed their noise-derived thresholds.
- $\Omega_{\mathrm{sens},e}$: transfer zeros and weak differential-defocus sensitivity are excluded.

### 6.2 Odd-aberration domain

For each OAI pattern $m$, source $n$, and branch $s$,

$$
\Omega_{m,n,s}
=
\Omega^{\mathrm{geom}}_{m,n,s}
\cap
\Omega_{\mathrm{amp}}^{Q}
\cap
\Omega_{\mathrm{amp},m}^{\mathrm{OAI}}
\cap
\Omega_{\mathrm{sens},m,n,s}^{o}.
$$

For the calibrated binary pupil support $A_p$, define the shifted-pupil branch indicators

$$
\chi_{m,n,s}(\mathbf u)
=
A_p(\mathbf u_{m,n})A_p(\mathbf u_{m,n}+s\mathbf u),
\qquad s\in\{+1,-1\},
$$

and the branch count

$$
N_m(\mathbf u)
=
\sum_{q\in\mathcal S_m}
\sum_{s'\in\{+1,-1\}}
\chi_{m,q,s'}(\mathbf u).
$$

The geometry-defined support $\Omega^{\mathrm{geom}}_{m,n,s}$ is the intersection of the common geometry-only support of the two defocus modes, the selected shifted-pupil branch $\chi_{m,n,s}=1$, and the exact branch-count condition $N_m(\mathbf u)=1$.

The geometry-defined single-branch regions are constructed directly from the calibrated binary pupil support and illumination wavevectors. No pupil erosion, circular-radius approximation, or empirically adjusted guard band is used in the documented workflow. Amplitude-, noise-, and sensitivity-based screening is applied only within these geometry-defined regions and does not alter their branch boundaries.

### 6.3 Reference numerical safeguards

The following values may be used as documented starting values when spectra are normalized consistently. Noise-derived thresholds remain preferred for experimental data.

| Parameter | Reference value | Role |
|---|---:|---|
| Normalized defocus-spectrum amplitude fallback | $2\times10^{-3}$ | fallback only |
| Normalized common-spectrum amplitude fallback | $2\times10^{-3}$ | fallback only |
| Normalized OAI-spectrum amplitude fallback | $2\times10^{-3}$ | fallback only |
| Minimum normalized defocus sensitivity | 0.06 | weak-response rejection |
| Robust common-spectrum scale percentile | 99.5% | avoids normalization by isolated spikes |

---

## 7. Regularization and linear solvers

### 7.1 Regularized spectral ratio

The even-aberration ratio uses

$$
\widehat\gamma(\mathbf u)
=
\operatorname{Re}
\left\{
\frac{
S_+^{\mathrm{filt}}(\mathbf u)
S_-^{\mathrm{filt}*}(\mathbf u)
}{
|S_-^{\mathrm{filt}}(\mathbf u)|^2+\epsilon_\gamma
}
\right\}.
$$

Report $\epsilon_\gamma$ in the intensity-spectrum scale used by the implementation. A scale-independent implementation should report the dimensionless multiplier used after normalizing the denominator energy.

### 7.2 Common-spectrum inversion

Report $\epsilon_Q$ in

$$
\widehat Q
=
-
\frac{
a_+^*S_+ + a_-^*S_-
}{
2(|a_+|^2+|a_-|^2)+\epsilon_Q
}.
$$

A reference normalized implementation uses a denominator floor of approximately $10^{-10}$.

### 7.3 Zernike fitting

The manuscript-consistent coefficient estimate is direct linear least squares:

$$
\widehat{\mathbf c}
=
\mathbf A^\dagger\mathbf b.
$$

Recommended solver:

- SVD or QR;
- float64 arithmetic for coefficient estimation;
- SVD relative cutoff: $10^{-12}$ as a reference value;
- no Tikhonov term in the reported manuscript-first pseudocode unless an additional coefficient regularizer is explicitly disclosed.

Report the numerical rank of the original unregularized design matrix. For direct comparison with the conditioning analysis in the manuscript, normalize every matrix column to unit Euclidean norm, compute the singular values of this column-normalized matrix, and report its condition number together with the normalization convention.

### 7.4 Final hybrid deconvolution

The final phase uses

$$
\widehat\phi(\mathbf u)
=
\frac{
H_{\mathrm{def}}^* I_{\mathrm{def}}
+
\sum_m H_{\mathrm{tilt},m}^*I_{\mathrm{tilt},m}
}{
|H_{\mathrm{def}}|^2
+
\sum_m|H_{\mathrm{tilt},m}|^2
+
\alpha(\mathbf u)
}.
$$

A reference energy-scaled choice is

$$
\alpha
=
10^{-12}
\max_{\mathbf u}
\left(
|H_{\mathrm{def}}|^2
+
\sum_m|H_{\mathrm{tilt},m}|^2
\right),
$$

but the value used for each experimental dataset must be reported.

---

## 8. Phase unwrapping and branch selection

| Quantity | Period | Procedure |
|---|---:|---|
| Even relative phase | $\pi$ | double the wrapped phase, apply 2-D $2\pi$ unwrapping in each connected component, then divide by two |
| Odd relative phase | $2\pi$ | apply 2-D unwrapping independently in every retained connected branch region |
| Disconnected-component offsets | integer multiples of $\pi$ or $2\pi$ | select jointly by minimizing the residual of the corresponding global Zernike model |

Record:

- the unwrapping algorithm and software version;
- connectivity convention;
- any minimum connected-component size;
- selected integer offsets;
- final Zernike residual.

---

## 9. Frequency selection for hybrid reconstruction

The corrected transfer functions determine the reliable contribution of each modality.

- Defocus data provide the principal low-frequency information.
- OAI data provide angular diversity and high-frequency information up to the bright-field synthetic-aperture limit.
- Frequency-selection masks or weights must be applied consistently to both the measured spectrum and its corresponding transfer function.
- For a nonbinary least-squares weight $w(\mathbf u)$, apply its square root $F(\mathbf u)=\sqrt{w(\mathbf u)}$ to both the measured spectrum and the corresponding transfer function; a binary mask is the special case $F\in\{0,1\}$.
- The final estimate must use the joint least-squares/Wiener expression in `Algorithm.md`.

Report the rule used to construct the frequency-selection weights. Do not substitute an undocumented image-space blend.

---

## 10. Minimum reproducibility record

For every published reconstruction, archive:

1. the four normalized raw images or their data-access statement;
2. the complete calibration table;
3. all reliable-frequency masks;
4. all single-branch masks;
5. the 12 estimated Zernike coefficients;
6. the corrected transfer functions;
7. $\epsilon_\gamma$, $\epsilon_Q$, and $\alpha$;
8. the least-squares solver and SVD cutoff;
9. the design-matrix rank and condition number;
10. the final quantitative phase map and its phase-reference convention.
