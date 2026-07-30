# Algorithm : CAC-QPI Aberration Recovery and Quantitative Phase Reconstruction from Experimental Images

## 1. Purpose

This document provides detailed end-to-end pseudocode for reconstructing the four experimentally acquired CAC-QPI intensity images. It intentionally excludes all forward-data generation and simulation-only procedures.

The manuscript is authoritative. The pseudocode therefore uses:

- two symmetric-defocus images;
- two weighted orthogonal multiplexed OAI images;
- Noll modes 4–15 for pupil-aberration representation;
- the manuscript's analytical aberration recovery;
- the manuscript's least-squares/Wiener hybrid deconvolution for the final phase.

No complex logarithm and no joint nonlinear object-pupil optimization are used.

---

## 2. Inputs

### 2.1 Experimental images

- $I_{\mathrm{def},+}(\mathbf x)$: intensity image at $+\Delta z$;
- $I_{\mathrm{def},-}(\mathbf x)$: intensity image at $-\Delta z$;
- $I_{\mathrm{tilt},1}(\mathbf x)$: first weighted orthogonal multiplexed OAI image;
- $I_{\mathrm{tilt},2}(\mathbf x)$: second weighted orthogonal multiplexed OAI image, rotated by $45^\circ$.

### 2.2 OAI patterns

For pattern $m\in\{1,2\}$, let $\mathcal S_m$ contain two mutually incoherent point sources. Each pair has a $90^\circ$ angular separation and calibrated intensity weights $A_{m,1}:A_{m,2}=1:2$.

A convenient azimuth convention is

$$
\mathcal S_1=\{0^\circ,90^\circ\},
\qquad
\mathcal S_2=\{45^\circ,135^\circ\}.
$$

The corresponding transverse illumination wavevectors are
$\mathbf u_{m,n}$, where $n\in\{1,2\}$.

### 2.3 Calibrated system parameters

- wavelength $\lambda$;
- objective numerical aperture $\mathrm{NA}_{\mathrm{obj}}$;
- OAI illumination numerical aperture $\mathrm{NA}_{\mathrm{ill}}$;
- magnification $M$;
- camera pixel size;
- symmetric defocus distance $\Delta z$;
- calibrated binary pupil-amplitude support $A_p(\mathbf u)$;
- calibrated illumination wavevectors $\mathbf u_{m,n}$;
- calibrated source intensity weights $A_{m,n}$;
- dark frames and exposure-normalization factors;
- numerical regularization and reliability parameters listed in `Reconstruction_parameters.md`.

### 2.4 Zernike basis

Use orthonormal real Zernike polynomials in Noll ordering.

Even-mode index set:

$$
\mathcal E=\{4,5,6,11,12,13,14,15\}.
$$

Odd-mode index set:

$$
\mathcal O=\{7,8,9,10\}.
$$

Noll modes 1–3 are excluded.

---

## 3. Outputs

- even-aberration coefficients $\widehat{\mathbf c}_e$;
- odd-aberration coefficients $\widehat{\mathbf c}_o$;
- even, odd, and total pupil phases
  $\widehat W_e(\mathbf u)$,
  $\widehat W_o(\mathbf u)$, and
  $\widehat W(\mathbf u)$;
- corrected defocus and OAI transfer functions;
- final quantitative phase spectrum $\widehat\phi(\mathbf u)$;
- final quantitative phase map $\widehat\phi(\mathbf x)$;
- reliable-frequency and single-branch masks used in the calculation.

---

## 4. Notation

- $\mathcal F\{\cdot\}$ and $\mathcal F^{-1}\{\cdot\}$: centered two-dimensional Fourier transform and inverse transform;
- $(\cdot)^*$: complex conjugate;
- $\operatorname{Arg}(\cdot)$: principal complex phase in $(-\pi,\pi]$;
- $\mathbf u$: object spatial-frequency coordinate;
- $k=2\pi/\lambda$: wavenumber in air;
- $u_c=\mathrm{NA}_{\mathrm{obj}}/\lambda$: objective cutoff frequency;
- $\boldsymbol\rho=\mathbf u/u_c$: normalized pupil coordinate;
- $\boldsymbol\rho_{m,n}=\mathbf u_{m,n}/u_c$: normalized illumination wavevector;
- $M_\Omega(\mathbf u)$: binary mask of a frequency set $\Omega$;
- $j=\sqrt{-1}$: imaginary unit;
- $s\in\{+1,-1\}$: positive or negative shifted-pupil branch.

The exact nonparaxial defocus phase is referenced to the pupil origin. In the notation of the manuscript,

$$
W_{\Delta z}(\mathbf u)
=
k\Delta z
\left[
\sqrt{1-\lambda^2\|\mathbf u\|^2}-1
\right],
$$

and $W_{-\Delta z}(\mathbf u)$ is computed for the negative axial displacement. For an ideal symmetric pair,

$$
W_{-\Delta z}(\mathbf u)=-W_{\Delta z}(\mathbf u).
$$

---

## 5. Detailed end-to-end pseudocode

```text
ALGORITHM CAC_QPI_EXPERIMENTAL_RECONSTRUCTION

INPUT:
    I_def_plus, I_def_minus
    I_tilt_1, I_tilt_2
    dark frames and exposure-normalization factors
    wavelength, objective NA, illumination NA
    magnification, camera pixel size
    symmetric defocus distance Delta_z
    calibrated pupil support A_p(u)
    OAI source patterns S_1 and S_2
    calibrated source weights A_mn
    numerical parameters

OUTPUT:
    c_even, c_odd
    W_even(u), W_odd(u), W_total(u)
    corrected transfer functions
    phase_spectrum(u), phase_map(x)
    all masks and reconstruction metadata


A. PREPROCESS THE FOUR EXPERIMENTAL IMAGES

A1. For each raw image:
        subtract the corresponding dark frame;
        divide by its calibrated exposure/brightness normalization factor;
        crop all images to the same field of view;
        verify identical pixel sampling and array dimensions.

A2. Compute centered Fourier spectra:
        I_def_plus_F  <- FFT2C(I_def_plus)
        I_def_minus_F <- FFT2C(I_def_minus)
        I_tilt_1_F    <- FFT2C(I_tilt_1)
        I_tilt_2_F    <- FFT2C(I_tilt_2)

A3. Remove the DC sample from the spectra used for aberration estimation:
        S_plus(0)  <- 0
        S_minus(0) <- 0
        S_tilt_m(0) <- 0

    For nonzero frequencies:
        S_plus(u)  <- I_def_plus_F(u)
        S_minus(u) <- I_def_minus_F(u)
        S_tilt_m(u) <- I_tilt_m_F(u)


B. BUILD THE FREQUENCY GRID AND KNOWN DEFOCUS PHASES

B1. Construct the physical spatial-frequency grid from:
        image dimensions,
        camera pixel size,
        magnification.

B2. Compute the objective cutoff:
        u_c <- NA_obj / wavelength

B3. Compute normalized pupil coordinates:
        rho <- u / u_c
        rho_mn <- u_mn / u_c

B4. Compute the exact nonparaxial, origin-referenced defocus phases:
        W_d(u)   <- exact_defocus_phase(+Delta_z, u)
        W_md(u)  <- exact_defocus_phase(-Delta_z, u)

    Do not replace this step with a paraxial quadratic phase unless such an
    approximation is explicitly being tested.


C. DEFINE THE RELIABLE DOMAIN FOR EVEN-ABERRATION RECOVERY

C1. Define the common geometric passband Omega_geom_e:
        frequencies supported by both defocus measurements,
        inside the calibrated native objective passband,
        excluding the DC point.

C2. Estimate or obtain the complex-noise amplitude of each defocus spectrum
    using the documented noise-estimation procedure for the dataset.

C3. Define Omega_amp_e:
        retain frequencies for which both |S_plus| and |S_minus|
        exceed their noise-derived thresholds.

C4. Define Omega_sens_e:
        exclude zeros and weak-response neighborhoods of the symmetric-
        defocus transfer relation, including frequencies where the known
        differential-defocus sensitivity is below its prescribed threshold.

C5. Form:
        Omega_e <- Omega_geom_e INTERSECT Omega_amp_e INTERSECT Omega_sens_e
        M_e(u)  <- indicator(Omega_e)

C6. Filter the two spectra:
        S_plus_filt  <- M_e * S_plus
        S_minus_filt <- M_e * S_minus


D. RECOVER THE EVEN-SYMMETRIC ABERRATION

D1. Evaluate the regularized spectral ratio only in Omega_e:

        gamma_hat(u) <-
            Re{
                S_plus_filt(u) * conj(S_minus_filt(u))
                /
                ( |S_minus_filt(u)|^2 + epsilon_gamma )
            }

D2. Recover the wrapped relative even-aberration phase:

        E_wrapped(u) <-
            atan2(
                sin(W_d(u)) - gamma_hat(u) * sin(W_md(u)),
                gamma_hat(u) * cos(W_md(u)) - cos(W_d(u))
            )

    E_wrapped is defined modulo pi.

D3. Convert the modulo-pi problem to a conventional modulo-2pi problem:

        doubled_phase(u) <- Arg{ exp( j * 2 * E_wrapped(u) ) }

D4. Split Omega_e into connected components.

D5. In each connected component:
        unwrapped_doubled <- two_dimensional_phase_unwrap(doubled_phase)
        E_component <- 0.5 * unwrapped_doubled

D6. Each disconnected component may retain an unknown integer multiple of pi.
    Select the component offsets jointly by minimizing the residual of one
    global even-Zernike model.

D7. For every retained frequency, construct the even-mode design row:

        A_even[u,k] <- Z_k(rho(u)) - Z_k(0)
        for k in E = {4,5,6,11,12,13,14,15}

D8. Solve the overdetermined linear least-squares problem:

        c_even <- pseudoinverse(A_even) * b_even

    where b_even contains the branch-consistent unwrapped samples.

D9. Synthesize:

        W_even(u) <- sum over k in E of c_even[k] * Z_k(rho(u))

D10. Adopt the measurable piston reference:

        E_hat(u) <- W_even(u) - W_even(0)


E. RECOVER THE DEFOCUS-DERIVED COMMON SPECTRUM

E1. Define:

        a_plus(u)  <- sin( W_d(u)  + E_hat(u) )
        a_minus(u) <- sin( W_md(u) + E_hat(u) )

E2. Recover the common spectrum Q(u) from both defocus equations by the
    regularized algebraic least-squares estimate:

        Q_hat(u) <-
            -[
                conj(a_plus(u))  * S_plus(u)
                +
                conj(a_minus(u)) * S_minus(u)
             ]
             /
             [
                2 * ( |a_plus(u)|^2 + |a_minus(u)|^2 )
                + epsilon_Q
             ]

E3. Retain Q_hat only in the native objective passband and where its
    magnitude and the joint defocus sensitivity are reliable.

    Q_hat contains the native-band object phase multiplied by the
    odd-symmetric pupil phase factor.


F. CONSTRUCT OAI SINGLE-BRANCH MASKS

F1. For each OAI pattern m in {1,2}:
        use the two calibrated sources in S_m;
        use their calibrated intensity weights A_mn.

F2. For every source n in pattern m and branch sign s in {+1,-1}, define
    the shifted-pupil support indicator:

        chi_mns(u) <- A_p(u_mn) * A_p( u_mn + s * u )

F3. For each OAI pattern, count all active shifted-pupil branches:

        N_m(u) <-
            sum over q in S_m and s' in {+1,-1} of chi_mqs'(u)

F4. Define the common geometry-only support of the two defocus modes:

        Omega_L <- Omega_def_geom excluding u=0

    where Omega_def_geom is the specimen-independent intersection of the
    calibrated geometry-only effective supports of the +Delta_z and
    -Delta_z defocus modes.

F5. Define the geometry-only single-branch candidate mask:

        M_geom_mns(u) <-
            indicator(Omega_L)
            * chi_mns(u)
            * indicator( N_m(u) = 1 )

        Omega_geom_mns <- {u : M_geom_mns(u) = 1}

    This step retains only frequencies at which exactly one branch from the
    complete multiplexed OAI pattern is active.

    Construct this geometry-only mask directly from the calibrated binary
    pupil support and calibrated illumination wavevectors. Do not erode the
    pupil, replace it with a circular-radius approximation, or add an
    empirically adjusted guard band.

F6. Within each geometry-only candidate support, apply three reliability
    tests:

        Omega_amp_Q:
            |Q_hat(u)| exceeds its noise-derived threshold;

        Omega_amp_OAI_m:
            |S_tilt_m(u)| exceeds its documented noise-derived threshold;

        Omega_sens_mns:
            the selected branch response is not at a zero or weak-response
            frequency.

F7. Form the final branch support:

        Omega_mns <-
            Omega_geom_mns
            INTERSECT Omega_amp_Q
            INTERSECT Omega_amp_OAI_m
            INTERSECT Omega_sens_mns

        M_mns(u) <- indicator(Omega_mns)

    These reliability tests only reject samples within the
    geometry-defined single-branch region; they do not change its branch
    boundary.

F8. Apply the same final mask to:
        the measured OAI spectrum S_tilt_m;
        the defocus-derived reference Q_hat.

    OAI frequencies outside these calibration masks are not used to estimate
    aberrations, but remain available for the final full-band phase
    reconstruction.


G. RECOVER THE ODD-SYMMETRIC ABERRATION

G1. For each retained source and positive branch, compute the known
    even-aberration contribution:

        Delta_W_even_mn_plus(u) <-
            W_even(u + u_mn) - W_even(u_mn)

G2. For each retained source and negative branch:

        Delta_W_even_mn_minus(u) <-
            W_even(u_mn) - W_even(u_mn - u)

    Equivalently, use the unified branch definition:

        Delta_W_even_mns(u) <-
            s * [ W_even(u_mn + s * u) - W_even(u_mn) ]

G3. For every branch s in {+1,-1}, remove:
        the calibrated source weight A_mn;
        the branch sign s;
        the factor j;
        the recovered even-aberration contribution.

    Define the corrected isolated branch:

        C_mns(u) <-
            [
                M_mns(u) * S_tilt_m(u)
                /
                ( s * j * A_mn )
            ]
            * exp( -j * Delta_W_even_mns(u) )

G4. Define the masked common-spectrum reference:

        Q_filt_mns(u) <- M_mns(u) * Q_hat(u)

G5. Obtain the wrapped odd-aberration measurement from the cross-spectrum
    phase:

        Delta_theta_mns(u) <-
            Arg{
                C_mns(u) * conj(Q_filt_mns(u))
            }

    This step does not evaluate a complex logarithm.

G6. For each connected component of every retained branch support:
        perform two-dimensional phase unwrapping with period 2*pi.

G7. Disconnected components may retain integer multiples of 2*pi.
    Select all residual component offsets jointly by minimizing the residual
    of one global odd-Zernike model.

G8. For every retained sample, construct the odd-mode design row:

        A_odd[m,n,s,u,k] <-
            s * [
                Z_k( rho_mn + s * rho(u) )
                - Z_k( rho_mn )
            ]
            - Z_k( rho(u) )

        for k in O = {7,8,9,10}

G9. Stack rows from:
        both OAI patterns;
        both sources in each pattern;
        positive and negative branches;
        all retained frequencies.

    Stack the branch-consistent unwrapped Delta_theta_mns samples into b_odd
    in exactly the same row order as A_odd.

G10. Solve the overdetermined linear least-squares system:

        c_odd <- pseudoinverse(A_odd) * b_odd

G11. Synthesize:

        W_odd(u) <- sum over k in O of c_odd[k] * Z_k(rho(u))


H. SYNTHESIZE THE ABERRATED PUPIL AND CORRECTED TRANSFER FUNCTIONS

H1. Form the estimated pupil phase:

        W_total(u) <- W_even(u) + W_odd(u)

H2. Form the aberrated pupil:

        P_hat(u) <- A_p(u) * exp( j * W_total(u) )

H3. Construct the corrected defocus phase transfer functions from P_hat
    and the exact nonparaxial propagation factors at +Delta_z and -Delta_z.

H4. Construct the transfer function corresponding to the measured
    defocus difference:

        H_def(u) <- H_def_plus(u) - H_def_minus(u)

        I_def(u) <- S_plus(u) - S_minus(u)

H5. For every individual OAI source, construct the corrected single-source
    OAI transfer function from the shifted aberrated pupil.

H6. For each measured multiplexed OAI pattern, form the incoherent
    intensity-weighted transfer function:

        H_tilt_m(u) <-
            sum over n in S_m of A_mn * H_tilt_mn(u)

    Use the same calibrated weights used during acquisition.

H7. Define real, nonnegative square-root weights from the corrected transfer
    sensitivities. Binary masks are the special case F(u) in {0,1}:

        F_def(u):
            emphasizes the reliable low-frequency defocus response;

        F_tilt_m(u):
            retains the reliable OAI response, including high-frequency
            components beyond the coherent objective cutoff.

    Apply the same selection to each image spectrum and its transfer
    function:

        I_def_eff      <- F_def * I_def
        H_def_eff      <- F_def * H_def
        I_tilt_m_eff   <- F_tilt_m * S_tilt_m
        H_tilt_m_eff   <- F_tilt_m * H_tilt_m


I. MANUSCRIPT-CONSISTENT HYBRID PHASE DECONVOLUTION

I1. Reconstruct the final phase spectrum by the aberration-corrected
    least-squares/Wiener solution:

        numerator(u) <-
            conj(H_def_eff(u)) * I_def_eff(u)
            +
            sum over m=1,2 of
                conj(H_tilt_m_eff(u)) * I_tilt_m_eff(u)

        denominator(u) <-
            |H_def_eff(u)|^2
            +
            sum over m=1,2 of |H_tilt_m_eff(u)|^2
            +
            alpha(u)

        phase_spectrum(u) <- numerator(u) / denominator(u)

    The sum over the two OAI frames is the explicit two-frame form of the
    compact OAI term used in Eq. (1) of the manuscript.

I2. Set frequencies outside the bright-field synthetic-aperture support
    to zero.

I3. Set the phase DC coefficient to the adopted reference value, normally
    zero, because the absolute phase offset is not observable.

I4. Transform to the spatial domain:

        phase_map(x) <- real{ IFFT2C(phase_spectrum) }

I5. Remove the spatial mean if a zero-mean phase reference is used.


J. SAVE REPRODUCIBILITY OUTPUTS

J1. Save:
        c_even and c_odd with Noll indices;
        W_even, W_odd, and W_total;
        Omega_e and M_e;
        all M_mns masks;
        corrected transfer functions;
        phase_spectrum and phase_map;
        all calibration parameters;
        all numerical thresholds and regularization values;
        design-matrix rank and condition-number diagnostics.

J2. Verify:
        the even design matrix has rank 8;
        the odd design matrix has rank 4;
        each retained OAI sample has exactly one active branch;
        no NaN or Inf values remain;
        the four input images use consistent normalization and sampling.

END ALGORITHM
```

---

## 6. Notes on implementation

### 6.1 Meaning of “closed-form aberration recovery”

The analytical part consists of common-factor cancellation, trigonometric inversion, cross-spectrum phase extraction, and direct linear Zernike fitting. Two-dimensional phase unwrapping only resolves periodic branches. It does not jointly optimize the object and pupil.

### 6.2 Piston and tilt

The even-aberration measurement is relative to the pupil center, so piston cancels. Pupil tilt lies in the null space of the odd finite-difference operator. Accordingly, Noll modes 1–3 are not estimated.

### 6.3 Multiplexed OAI data

The two point sources within one OAI exposure are mutually incoherent. Their intensities and transfer functions therefore add with calibrated intensity weights. The weights are measured optical intensity weights after exposure and source-brightness normalization, not merely nominal LED drive-current ratios.

### 6.4 Calibration support versus reconstruction support

Single-branch masks are required only for odd-aberration calibration. OAI frequencies outside those masks may still contribute to the final phase reconstruction through the full multiplexed OAI transfer functions.

### 6.5 Numerical linear algebra

Use QR or SVD-based least squares. Report the numerical rank of the original unregularized design matrix. For comparison with the manuscript's conditioning analysis, normalize every design-matrix column to unit Euclidean norm, compute the singular values of the column-normalized matrix, and report its condition number together with the normalization convention. Do not infer identifiability from a regularized matrix.
