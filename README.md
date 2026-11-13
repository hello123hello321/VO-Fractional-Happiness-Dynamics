# Variable-Order Fractional Happiness Dynamics

Research code, digitized data, and archived results for parameter identification and adaptive state and fractional-order estimation in a variable-order fractional happiness model, applied to depressive-symptom trajectories from individuals with bipolar disorder.

The repository contains the data before and after interpolation, the interpolation script, the main computational notebook, and results from several calibration lengths. **The primary two-stage experiment uses the first 60 samples for calibration, initializes the observer at sample index 59, and updates it from index 60 onward.** All notebook sample indices are zero-based.

## Repository contents

The filenames below are preserved exactly as uploaded, including the `CVS` spelling.

| File | Contents and purpose |
| --- | --- |
| [`CVS_Initial sampling.zip`](CVS_Initial%20sampling.zip) | Points digitized from the published curves, before interpolation. Contains `Data1.csv`–`Data6.csv` and the corresponding `D1.txt`–`D6.txt` files. |
| [`code_for_interpolate.zip`](code_for_interpolate.zip) | Contains `code_for_interpolate/Interpolate.txt`, a Python script for linear interpolation of the digitized points. |
| [`CVS_after_interpolate.zip`](CVS_after_interpolate.zip) | Six interpolated sequences, stored as `Interpolated_Data1.csv`–`Interpolated_Data6.csv`. |
| [`main_code.ipynb`](main_code.ipynb) | Main Python notebook. Its current default is 60-sample Stage I calibration followed by the Stage II adaptive observer. The six interpolated sequences are embedded in its `DATA` dictionary. |
| [`combined_online_observer_output_full_data.zip`](combined_online_observer_output_full_data.zip) | Retrospective Stage I calibration using the complete available sequence for each series. Despite its filename, this archive contains Stage I outputs only. |
| [`combined_output_prefix32.zip`](combined_output_prefix32.zip) | Stage I calibration using the first 32 samples of each sequence. No Stage II output is included. |
| [`combined_output_prefix40.zip`](combined_output_prefix40.zip) | Stage I calibration using the first 40 samples of each sequence. No Stage II output is included. |
| [`combined_output_prefix50.zip`](combined_output_prefix50.zip) | Stage I calibration using the first 50 samples of each sequence. No Stage II output is included. |
| [`combined_prefix60_observer_start59_results.zip`](combined_prefix60_observer_start59_results.zip) | Primary experiment: Stage I calibration on the first 60 samples and Stage II observation of subsequent samples. Observer outputs are in `online_observer_start59/` inside the archive. |

Extract each results archive into a separate directory: different experiments reuse the same internal filenames.

## Data source and preprocessing

The data were reconstructed by point-by-point digitization of the six QIDS-SR16 trajectories in **Figure 4(a) of Bonsall et al. (2015)**, followed by linear interpolation. QIDS-SR16 denotes the 16-item Quick Inventory of Depressive Symptomatology, Self-Report.

These files contain figure-derived reconstructions, rather than the source study's original participant-level database. Interpolation does not create additional independent clinical observations.

### File formats and sequence lengths

- `Data1.csv`–`Data6.csv`: no header; the first two columns contain the digitized horizontal coordinate and symptom-score value.
- `Interpolated_Data1.csv`–`Interpolated_Data6.csv`: header `X,Y`; `X` is the integer interpolation coordinate and `Y` is the interpolated score.
- `series1`–`series6` in the notebook correspond to `Interpolated_Data1.csv`–`Interpolated_Data6.csv`, respectively. The notebook uses the `Y` values in their existing order and assigns zero-based row indices.

| Series | Digitized CSV rows | Interpolated samples | Interpolated `X` range | Stage II updates after the 60-sample prefix |
| --- | ---: | ---: | --- | ---: |
| S1 | 118 | 196 | 1–196 | 136 |
| S2 | 103 | 217 | 1–217 | 157 |
| S3 | 112 | 218 | 1–218 | 158 |
| S4 | 106 | 159 | 2–160 | 99 |
| S5 | 114 | 189 | 1–189 | 129 |
| S6 | 125 | 233 | 1–233 | 173 |
| Total | 678 | 1,212 | — | 852 |

The CSV `X` coordinate, notebook `sample` index, and dimensionless `model_time` are different quantities. In particular, S4 begins at CSV `X = 2`, while its first notebook sample is still index 0. The fitted model-time step is not a measured clinical sampling interval.

### Interpolation procedure

The script:

1. Reads the first two columns of one digitized CSV file.
2. Sorts points by their horizontal coordinate.
3. Averages the score values at duplicate horizontal coordinates.
4. Constructs integer coordinates from `ceil(min(X))` through `floor(max(X))`, inclusive.
5. Uses `scipy.interpolate.interp1d(..., kind="linear")` and exports a CSV with columns `X,Y`.

It performs interpolation within the digitized domain; it does not extrapolate beyond the endpoints or round the reconstructed scores to integers.

The supplied script is notebook-oriented and currently selects the first CSV found under `/kaggle/input/`. To process a chosen series locally, paste `Interpolate.txt` into a notebook cell and replace its `file_path` assignment with an explicit path, for example:

```python
file_path = "CVS_Initial sampling/Data1.csv"
```

Change the export filename from `Interpolated_Data.csv` to `Interpolated_Data1.csv`. Repeat with the matching input and output number for all six series. The script processes one file per execution; its original file-discovery expression does not iterate over all six files. In a standalone `.py` script, also import `display` from `IPython.display`, or replace `display(result)` with `print(result)`.

## Computational workflow

The three-coordinate nonlinear field is

$$
{}^{C}_{0}D^{q(t)}\mathbf{X}(t)
=\begin{pmatrix}y\\z\\-x-b(1-x^2)y-az\end{pmatrix},
\qquad \mathbf{X}=(x,y,z)^{\mathsf T}.
$$

The implementation uses the current-order Caputo convention and a full-history implicit L1 discretization. Normalized QIDS scores are assigned to the second coordinate, `y`, without sign inversion. Thus, higher observed values represent greater depressive-symptom severity. The unmeasured coordinates `x` and `z` are model pseudo-states.

### Stage I: prefix calibration

For each sequence, Stage I fits the first `N_CALIBRATION_POINTS` samples without measurement correction during the simulated trajectory. Prefix mean and population standard deviation define the normalization.

The variable-order fit has **53 optimized quantities**: `a`, `b`, the initial hidden coordinates `x0` and `z0`, `log(h)`, and 48 cubic B-spline coefficients for the offline order profile. It is not a two-parameter fit.

Default settings include:

| Setting | Value |
| --- | --- |
| Calibration samples | 60 per series |
| Order representation | 48 clamped cubic B-spline coefficients |
| Optimization grids | 2 and 4 substeps per sample interval |
| Multistart seeds | 1–100 |
| Maximum function evaluations | 1,000 per optimization start |
| Order regularization weight | 1.0 |
| Bounds on `a`, `b` | `[-10, 10]` |
| Bounds on `x0`, `z0` | `[-3, 3]` |
| Bounds on dimensionless `h` | `[0.03, 3]` |
| Bounds on spline coefficients | `[0.03, 0.98]` |

The objective combines reconstruction errors on both grids, agreement between the grids, and an order-smoothness penalty. The selected fit has the lowest objective among valid starts. Optimizer termination status and subsequent mesh checks are reported separately.

### Stage II: adaptive observation

Only `a`, `b`, `h`, and the prefix normalization are transferred from Stage I. They remain fixed. The fitted hidden initial conditions and offline order spline are not transferred.

For the primary experiment:

- Calibration uses indices **0–59**.
- Observer initialization is at index **59**, with `x_hat = z_hat = 0`, the normalized observation as `y_hat_normalized`, and `q = 0.505`.
- The first measurement update is at index **60**.
- Earlier calibration samples are not replayed through the observer. Its full L1 history begins at index 59 and is retained thereafter.
- The observer uses one implicit L1 step per incoming sample and estimates the states and current fractional order.

The order remains in `[0.03, 0.98]`. Nominal observer poles are `[-1, -2, -3]`; the adaptation gain is `0.05 / 0.12`, the order-rate limit is `1 / 21.48`, and the sensitivity regularizer is `1e-8`. The Newton stopping tolerance is `1e-10`, with up to 35 iterations and backtracking.

Stage II output errors measure **filtered tracking using the current observation**, not an observation-free forecast. The causal-update checks are conditional on fixed prefix calibration and the already reconstructed input sequences.

## Running the main notebook

### Python environment

The notebook imports NumPy, SciPy, pandas, Matplotlib, and IPython. Use a Python 3 Jupyter environment with SciPy's `BSpline.design_matrix` available. A basic installation is:

```bash
python -m pip install numpy scipy pandas matplotlib ipython jupyterlab
jupyter lab main_code.ipynb
```

Each results archive contains `execution_manifest.json` with its recorded package versions and execution settings. The notebook records the current environment when run. This repository does not currently include a dependency lockfile; the command above installs available package versions rather than recreating a pinned environment.

### Primary 60-sample experiment

1. Open `main_code.ipynb` in a fresh kernel.
2. Keep `N_CALIBRATION_POINTS = 60` and the default fit/observer settings.
3. Run the notebook in order, including the observer execution and final audit cells.
4. Inspect the summaries and diagnostic flags after completion.

The data are already embedded, so the default notebook does not need to load or extract the CSV archives. Stage I uses 100 optimization starts per sequence and can take substantial time.

The effective output directory is:

```text
combined_online_observer_output_prefix60/
```

Stage II outputs are written to its `online_observer_start59/` subdirectory. The final cell creates or refreshes `combined_online_observer_output_prefix60.zip`, including both stages. This generated ZIP name differs from the uploaded primary-results archive name.

### Other calibration lengths

Set `N_CALIBRATION_POINTS` to **32**, **40**, or **50**, restart the kernel, and run from the beginning. To obtain only Stage I outputs, finish the Stage I export/manifest cell and stop before the cell that imports `place_poles` and defines the observer settings. Running all cells will additionally execute Stage II for the chosen prefix; those extra outputs are not part of the uploaded prefix32/40/50 archives.

The full-data archive uses each sequence's own complete length. The current notebook is configured for a common integer prefix and does **not** expose a full-record mode through `N_CALIBRATION_POINTS = None` or one common full-length value. The full-data archive documents that separate retrospective calibration experiment.

### Checkpoints and reruns

`RESUME = True` allows reuse of compatible Stage I JSON checkpoints. Compatibility checks include data, settings, code, and environment; a mismatch stops the run instead of silently loading incompatible fits.

For a separate experiment, use a new output directory in the prefix-configuration cell, where `OUT` is reassigned. Changing only the initial `OUT` assignment in the first cell is insufficient. Keep downloaded result archives separate from a new run's output directory. Stage II JSON files record calibration, metadata, and summaries; the supplied notebook recomputes Stage II rather than restoring its entire history from those JSON files.

## Reading the archived results

### Stage I files

| File pattern | Meaning |
| --- | --- |
| `fit_summary_new_bounds.csv` | Selected coefficients, initial hidden coordinates, reconstruction RMSE, objective value, offline order statistics, validity, optimizer success, and runtime. |
| `fit_diagnostics_new_bounds.csv` | Fitted `h`, sample/parameter counts, optimizer messages, mesh results, residual diagnostics, and unresolved-status flags. |
| `multistart_attempts_new_bounds.csv` | Per-start records for all six series; the uploaded archives contain 600 rows. |
| `seriesN_fit_new_bounds.json` | Detailed fit, parameter vector, normalization, optimization attempts, evaluation, and checkpoint metadata. |
| `seriesN_reconstruction_new_bounds.csv` | Calibration-interval `sample`, `observed`, `reconstruction`, offline `q`, and fitted-grid reconstruction. A verification-grid reconstruction is included when available. |
| `reconstruction_and_q_new_bounds.png` | Stage I reconstruction and offline-order plots. |
| `execution_manifest.json`, `protocol_checks.json` | Execution provenance, configuration, and recorded checks. |

### Stage II files

Inside `online_observer_start59/`, each series has an observer CSV and a JSON metadata/summary file. The folder also contains `online_observer_summary.csv`, `online_protocol_checks.json`, and `online_observer_diagnostics.png`.

Key CSV columns are:

| Column | Meaning |
| --- | --- |
| `sample` | Zero-based index in the complete interpolated sequence. |
| `observer_sample` | Local observer index; 0 corresponds to global sample 59. |
| `observed_y`, `y_hat` | Reconstructed observation and filtered output in QIDS score units. |
| `y_hat_normalized` | Estimated second coordinate in the normalized model scale. |
| `x_hat`, `z_hat` | Estimated hidden pseudo-states in normalized model coordinates. |
| `q_used` | Fractional order used in the current implicit state solve. |
| `q_after` | Order after the current adaptation update, used for the next state solve. |
| `innovation_normalized`, `local_sensitivity` | Current normalized output error and local output sensitivity to order. |
| `raw_rate`, `rate_limited`, `boundary_limited` | Unclipped adaptation rate and its rate/boundary limit indicators. |
| `newton_residual`, `newton_iterations` | Numerical solve diagnostics. |
| `model_time` | `(sample - 59) * h` for the primary observer, beginning at initialization. |
| `calibration_prefix` | True for samples 0–59. |
| `observer_initialization` | True at sample 59. |
| `computed` | Whether an observer state was produced, including initialization. |

Observer CSVs preserve all original row indices. Blank observer values at indices 0–58 are intentional: the observer did not run there. Initialization at index 59 is not an evaluated measurement update. Use **`sample >= 60` and `computed == True`** for post-calibration metrics.

The notebook's diagnostic order plot uses `q_after`. For the order entering each state solve, use `q_used`, as in the manuscript's order trajectories. The archived summary's `q_used` mean and standard deviation include initialization; recompute them on `sample >= 60` for post-calibration-only statistics.

### Numerical-status interpretation

`valid`, optimizer `success`, and `mesh_pass` answer different questions. A selected valid trajectory can have `success = False`, and a completed calculation can have an unresolved mesh check. The archived Stage I diagnostics report:

| Experiment | Optimizer unresolved | Mesh unresolved |
| --- | --- | --- |
| Full data | None | S5 |
| Prefix 32 | None | S1, S2, S5 |
| Prefix 40 | S4 | S2, S3, S4, S5 |
| Prefix 50 | S5 | S5 |
| Prefix 60 | S5 | None |

The unresolved optimizer cases reached the maximum number of function evaluations. See the per-series diagnostic messages for the mesh-check outcomes. These statuses are preserved in the archives. All six archived Stage II runs in the primary 60-sample experiment completed.

## Primary archived tracking results

Using only samples from index 60 onward:

| Series | Evaluated samples | RMSE in QIDS score units |
| --- | ---: | ---: |
| S1 | 136 | 0.17486 |
| S2 | 157 | 0.38244 |
| S3 | 158 | 0.21602 |
| S4 | 99 | 0.22674 |
| S5 | 129 | 1.02832 |
| S6 | 173 | 0.07290 |
| Pooled | 852 | 0.45568 |

The pooled MAE is **0.23972**. Pooled errors combine all evaluated samples; they are not the unweighted mean of the six per-series RMSE values. The hidden coordinates and adaptive order are model-based estimates without measured ground-truth trajectories in these data.

## Data-source citation

Bonsall, M. B., Geddes, J. R., Goodwin, G. M., and Holmes, E. A. (2015). Bipolar disorder dynamics: affective instabilities, relaxation oscillations and noise. *Journal of the Royal Society Interface*, 12(112), 20150670. [https://doi.org/10.1098/rsif.2015.0670](https://doi.org/10.1098/rsif.2015.0670).

Please acknowledge this repository for the reconstruction/code and cite the source article when using the figure-derived data.
