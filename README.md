# Statistical Image Denoising Pipeline

A ten‑stage, **statistics‑only** pipeline that *identifies* the noise in a
grayscale image and *routes* it to the matched denoising method — instead of
assuming a single noise model. Identification is done with classical statistical
estimators and tests (order statistics, moments, robust scale estimators,
goodness‑of‑fit); no black‑box or learned components are used.

The guiding principle: **the intensity histogram is the scene, not the noise.**
Every decision is made on the *noise distribution* (the residual `image − local
estimate`) and on local statistics, not on the raw pixel histogram.

---

## Key idea

A fixed 10‑stage skeleton is applied to every image. Stages 1–7 **measure and
identify**; stages 8–10 **act**. Stage 7's identification selects which action
stages fire, so two images with different noise take opposite branches through
the *same* code:

| Noise identified | Stage 8 (outlier removal) | Stage 10 (filter) | Extra |
|---|---|---|---|
| Additive Gaussian | skipped | **NL‑means** (matched) | — |
| Impulse (salt & pepper) | **decision‑median** (active) | skipped | — |
| Pure Poisson | skipped | NL‑means in VST domain | **Anscombe** transform |
| Speckle (multiplicative) | skipped | NL‑means in log domain | log transform |

Applying the wrong filter to the wrong noise (e.g. NL‑means on salt‑and‑pepper,
or a median on Gaussian) degrades the result — the identification step exists to
prevent exactly that.

---

## The ten stages

| # | Stage | Role | Primary statistical methods |
|---|---|---|---|
| 1 | Load / acquisition | action | `cv2.imread`, float/1‑D copies |
| 2 | Scale & dynamic range | measure | `min`, `max`, `ptp`, `percentile` (order statistics) |
| 3 | Central tendency | measure | `mean`, `median`, `trim_mean`, `mode` |
| 4 | Spread & dispersion | measure | `std`, `iqr`, `median_abs_deviation` (MAD) |
| 5 | Outlier / impulse identification | mask | `zscore` / deviation‑from‑local‑median, `ndimage.label` (cluster geometry) |
| 6 | Distribution shape + **noise distribution** | measure | `skew`, `kurtosis`, `skewtest`, `kurtosistest`, `shannon_entropy`; **residual kurtosis** |
| 7 | **Noise‑model identification** | decision | `estimate_sigma`, local mean–variance regression, `corrcoef`, `normaltest`, `kstest`, `cramervonmises` |
| 8 | Outlier removal | action | decision‑based median (impulse only) |
| 9 | Normalisation | action | min–max affine (applied to the *denoised* image) |
| 10 | Filtering | action | `denoise_nl_means` (Gaussian/VST domains) |

**Decisive identifiers at Stage 6–7**

- **Additive Gaussian:** residual kurtosis ≈ 0; `estimate_sigma` > 0; mean–variance slope ≈ 0.
- **Impulse:** isolated single‑pixel outliers; residual kurtosis ≫ 0 (spike + heavy tails); `estimate_sigma` ≈ 0 in flat regions.
- **Speckle:** variance scales with `μ²` (significant quadratic term).

---

## Requirements

Exact versions the pipeline was developed and validated against:

| Package | Version | Role |
|---|---|---|
| Python | 3.12.3 | runtime |
| NumPy | 2.4.4 | arrays, descriptive statistics |
| SciPy | 1.17.1 | statistical tests, `ndimage` filters |
| scikit‑image | 0.26.0 | `estimate_sigma`, `denoise_nl_means`, metrics |
| OpenCV (cv2) | 4.13.0 | image I/O, `calcHist` |
| Pillow | 12.1.1 | image I/O |
| PyWavelets | 1.8.0 | **required by `estimate_sigma`** |
| Matplotlib | 3.10.8 | figures |

Install:

```bash
pip install numpy scipy scikit-image opencv-python pillow PyWavelets matplotlib
```

---

## Usage

Edit the three paths at the top of the script, then run it:

```python
CLEAN = "org.jpg"      # clean reference (optional; enables full-reference metrics)
SRC   = "corr.png"      # image to process
OUT   = "C:/pipeline_out/"  # output folder (auto-created)
```

```bash
python adaptive_pipeline.py
```

- If `CLEAN` is provided and matches `SRC` in size → **full‑reference** metrics
  (MSE, PSNR, SSIM, ROI‑SSIM).
- If no clean reference exists (a genuinely noisy real image) → **no‑reference**
  metrics (residual σ, impulse specks removed, region preservation).

The scripts print a per‑stage log (`STAGE 1 … STAGE 10`), save per‑stage images,
write the cleaned result, and render the diagnostic flow figure.

---

## Outputs

| File | Contents |
|---|---|
| `s01_input.png` | loaded input |
| `s05_impulse_mask.png` / outlier mask | Stage‑5 flagged pixels |
| `s08_denoised.png` / `cleaned_result.png` | **the cleaned image** |
| `s09_normalised.png` | normalised (denoised) image |
| `*_flow.png` | 3×6 per‑stage figure (image + graph, lettered panels) |
| `*_table.png` | manuscript‑format metrics table |

---

## Metrics

- **Full‑reference (requires clean image):** MSE, PSNR, SSIM, and an ROI‑SSIM
  (e.g. crack/edge region). Computed on the **native intensity scale**; the
  filter is tuned only from the noisy image (no reference leakage); values are
  reported as mean ± SD over multiple noise realisations.
- **No‑reference (real images):** residual noise σ, impulse specks before/after,
  fraction of pixels modified, clean‑region preservation, and a
  variance‑stabilisation consistency check (post‑Anscombe σ → 1).

> **PSNR is a monotone function of MSE** — they are one axis, not two. The
> independent quality signals are {MSE/PSNR} and SSIM.

---

## Statistical methods (audit)

Every **identification, measurement, and denoising** decision is made with a
statistical estimator or test: order statistics (percentiles, median, IQR),
moments (mean, std, skewness, kurtosis), robust estimators (trimmed mean, MAD,
`estimate_sigma`), Shannon entropy, connected‑component cluster statistics,
least‑squares mean–variance regression, and goodness‑of‑fit tests
(`normaltest`, `kstest`, `cramervonmises`). The median filter used for impulse
removal is the local‑median order statistic.

Non‑statistical elements are strictly supporting infrastructure: file I/O, dtype
casting, array reshaping, the affine min–max rescale, the gradient operator used
only to define an ROI mask, plotting, and (in controlled experiments) the
noise‑generation step.

---

## Important caveats (read before reporting results)

1. **Add‑noise‑then‑remove validates the *method*, not real‑world performance.**
   Adding a known noise and recovering it confirms the estimator works, because
   you can only measure recovery against known truth. It is *not* a claim about
   deployment on unknown‑noise images. Keep the two separate.
2. **Full‑reference metrics need a clean reference.** Real noisy images have no
   ground truth, so PSNR/SSIM cannot be computed on them — use the no‑reference
   measures instead.
3. **PSNR gains are not comparable across noise types.** Impulse noise (few
   pixels, extreme) admits near‑perfect recovery (large PSNR gain); Gaussian
   (every pixel perturbed) has a lower ceiling. Larger gain ≠ better filter.
4. **Lossy‑JPEG references bias metrics** slightly toward smoothing filters.
5. **The one non‑adaptive constant** is the impulse detector threshold
   (`|image − median| > 60`); a MAD‑based adaptive cutoff (`median ± k·MAD`)
   makes detection fully data‑driven.

---

## Extending to medical (X‑ray / CT / MRI)

The same skeleton applies; Stage 7 identification changes the routing:
Poisson/Poisson–Gaussian (X‑ray/CT) triggers the (generalized) Anscombe branch
and an edge‑preserving filter (never a plain blur on a fracture); MRI magnitude
noise is Rician; ultrasound is multiplicative speckle. Real clinical images have
no clean reference, so validation is no‑reference or multi‑frame.

---

## File overview

- `adaptive_pipeline.py` — full identify‑and‑route pipeline (all noise types).
- `full_pipeline.py` — additive‑Gaussian case with per‑stage flow figure.
- `crack_sp_pipeline_final.py` — salt‑and‑pepper case (corrected flow).
- `noise_id_full_pipeline.py` — Poisson/Poisson–Gaussian identification + VST routing.
- `noise_id_simulation.py` — Monte‑Carlo benchmark (confusion matrix, parameter recovery).
- `photon_noise_estimation.py` — Poisson–Gaussian gain/read‑noise estimator.

---

## License / attribution

Add your preferred license here. If you use the public datasets referenced in
development (e.g. Özgenel concrete‑crack images), cite them per their terms.
