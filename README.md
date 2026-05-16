# Van der Pol Oscillator — Dataset & Classification

This repository contains code, data, and model results used to generate a feature dataset from simulations of the Van der Pol oscillator and to train/evaluate classifiers to predict oscillation regimes.

**Contributors**


BL.EN.U4AID23003- Amara Pranav
BL.EN.U4AID23006- B G Rajath Siddarth
BL.EN.U4AID23054- V. Sidharrth
BL.EN.U4AID23063- Paruchuri Sai


**Project summary**
- Generates time-series windows from a Simulink Van der Pol model (`vanderpol.slx`).
- Extracts time-domain features per window (RMS, entropy, crest factor, kurtosis, etc.).
- Saves a consolidated CSV dataset and prepares balanced train/test splits.
- Runs ML experiments across many classical classifiers and records best hyperparameters and metrics.

**Files**
- **vanderpol.slx**: Simulink model of the Van der Pol oscillator used to simulate trajectories for different `mu` values. Open in MATLAB/Simulink.
- **generate_dataset.txt**: MATLAB script (plain text) showing `generate_dataset.m` logic used to run `vanderpol.slx`, window signals, extract features, label regimes, and write `vanderpol_dataset_final.csv`. Key variables: `mu_values`, `window_size`, `stride`, and the feature list.
- **vanderpol_dataset_final.csv**: Full dataset produced by the MATLAB script. Columns:
  - `mu`, `std_y`, `rms_y`, `rms_dy`, `rms_ddy`, `energy_y`, `signal_power`, `peak_to_peak`, `crest_factor`, `zero_crossings`, `entropy_y`, `skewness_y`, `kurtosis_y`, `mean_abs_y`, `shape_factor`, `regime`
- **balanced_train_dataset (1).csv**: Balanced training split saved by the notebook (samples per `mu` equalized).
- **test_dataset (1).csv**: Test split containing held-out `mu` values for evaluation.
- **Code.ipynb**: Jupyter notebook (Python) that demonstrates dataset inspection, balancing, preprocessing (scaling, PCA), model training/search, and plotting. It contains sections to:
  - load `vanderpol_dataset_final.csv`, create balanced train/test splits, and save `balanced_train_dataset.csv` and `test_dataset.csv`;
  - visualize distributions, compute correlations, and run PCA;
  - set up many classifiers, run randomized search / CV, and save model results.
- **model_best_params.csv**: Exported best hyperparameters for each tried classifier (from the notebook's hyperparameter search).
- **model_comparison_metrics (1).csv**: Per-model comparison metrics (CV F1, accuracy, precision/recall, fit/predict times) produced by the experiments.
- **balanced_train_dataset (1).csv**, **test_dataset (1).csv**, and other CSVs: alternate saved dataset files (inspect for exact naming if duplicates exist).

**Datasets / Features**
- The dataset rows correspond to fixed-length windows extracted from simulated signals. Each row includes the `mu` parameter and a `regime` label with four classes: `Harmonic`, `Weak_Nonlinear`, `Strong_Nonlinear`, and `Relaxation`.
- Features are mostly time-domain statistics and derived measures (RMS, energy, crest factor, entropy, skewness, kurtosis, etc.). See `generate_dataset.txt` for the full extraction pipeline and formulae.

**How to run (quick)**
1. Open `Code.ipynb` in Jupyter/Colab. The notebook installs Python dependencies with `pip install numpy pandas scipy scikit-learn matplotlib seaborn`.
2. If you need to regenerate the dataset from simulation: open MATLAB, run `generate_dataset.m` (based on `generate_dataset.txt`) which requires `vanderpol.slx`.
3. Use the notebook cells to preprocess (`balanced_train_dataset.csv` / `test_dataset.csv`), run model searches, and reproduce plots and CSV exports.


**Analog circuit / Simulink model**
- Mathematical form: the Van der Pol oscillator is implemented in its standard second-order ODE form
  y'' - mu*(1 - y^2)*y' + y = 0
  which is rearranged for simulation as
  y'' = mu*(1 - y^2)*y' - y.
- State form used in the model: set x1 = y and x2 = y' so
  - x1' = x2
  - x2' = mu*(1 - x1^2)*x2 - x1
  The Simulink model implements this using two cascaded integrator subsystems (one to integrate x2 to produce x1, and another for x2), sums, gains, and multipliers.
- Subsystems and blocks (what to look for in `vanderpol.slx` and the diagram images):
  - Integrator subsystem: an op-amp-style integrator (capacitor feedback) block implementing continuous integration for state variables (see `Integrator_subsystem.png`).
  - Summer subsystem: summing amplifier block that combines scaled signals and feedback paths (see `Summer_subsystem.png`).
  - Nonlinear term: the (1 - x1^2) factor is implemented by squaring `x1` (multiplier or product block), subtracting from 1, then multiplying by `x2` to form (1 - x1^2)*x2.
  - mu injection: a tunable gain block labeled `mu/10` (visible in the top-level diagram) scales the `mu` value before additional scaling (there are other scalar blocks with labels like `8`, `1/5`, `-1/4`, `10/4` that adjust signal amplitudes to match the analog component scaling used in the model). These numeric gains are part of the circuit-level scaling — the code varies only the `mu` parameter (explained below) and the model multiplies and rescales it internally.
  - Outputs: the model writes three signals to the MATLAB base workspace (named `output_y`, `output_dy`, and `output_data` in the notebook/script). These are used by the MATLAB script for windowing and feature extraction.

**How the MATLAB script varies `mu` and generates samples**
- `mu` list: `generate_dataset.txt` contains the full `mu_values` array used for simulation. The script loops over these values (20 values in the provided script) covering four qualitative regimes (Harmonic, Weak_Nonlinear, Strong_Nonlinear, Relaxation).
- Passing `mu` to Simulink: inside the loop the script does `assignin('base','mu',mu);` then calls `sim('vanderpol');`. This assigns the scalar `mu` into the MATLAB base workspace so the Simulink model can read the value from a `From Workspace` or gain block (the top-level diagram shows a `mu/10` gain block wired into the model).
- Regime labeling:
  - `Harmonic` when `mu <= 1e-7`
  - `Weak_Nonlinear` when `mu > 1e-7 && mu < 1`
  - `Strong_Nonlinear` when `mu >= 1 && mu < 10`
  - `Relaxation` otherwise (mu >= 10)
- Simulation outputs: after `sim('vanderpol')` the script expects signals in `output_y`, `output_dy`, and `output_data` variables (column vectors). It does basic sanity checks (empty, NaN/Inf, or extremely large values) and skips invalid runs.

**Windowing and feature extraction **
- Transient removal: the script removes an initial transient by discarding the first 20% of samples via `transient_cut = floor(0.2 * length(y));` and working on the remainder.
- Windowing: uses `window_size = 100` and `stride = 50` (so windows overlap by 50 samples). For each valid window the script normalizes the window to zero mean and unit variance before feature computation.
- Features computed per window:
  - `std_y` (standard deviation)
  - `rms_y`, `rms_dy`, `rms_ddy` (RMS of y, y', y'')
  - `energy_y`, `signal_power`
  - `peak_to_peak`, `crest_factor`
  - `zero_crossings`
  - `entropy_y` (power-based entropy)
  - `skewness_y`, `kurtosis_y`
  - `mean_abs_y`, `shape_factor` (rms / mean_abs)
- Each window row saved includes the `mu` value and `regime` label. At the end the script assembles a table and calls `writetable(dataset_table,'vanderpol_dataset_final.csv');`.

**Train / test split used**
- We further select specific `mu` values for training and separate `mu` values for testing. Example sets used in the notebook:
  - `train_mu`: [0, 1e-10, 1e-9, 0.01, 0.05, 0.1, 2, 3, 4, 10, 12, 15]
  - `test_mu`: [1e-8, 1e-7, 0.2, 0.5, 5, 7, 18, 20]
- We balance the training set per `mu` by sampling each `mu`-group down to the minimum group count so that the classifier training is not biased by mu-rich regions.

**Where to look / reproduce**
- Visuals: `Analog_setup_Vanderpol.png`, `Summer_subsystem.png`, and `Integrator_subsystem.png` illustrate the top-level Simulink wiring and the key subsystems. Open `vanderpol.slx` in Simulink to inspect component-level values and exact block types.
- Script: `generate_dataset.txt` contains the full MATLAB code used to run `vanderpol.slx`, label regimes, perform windowing, extract features, tabulate results, and save `vanderpol_dataset_final.csv`.
