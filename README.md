# Learning about the Long Run: Replication and Epoch-Sensitivity Extension

A MATLAB replication of the computational package for **Farmer, Nakamura & Steinsson (2024)**, together with the reproducibility engineering needed to run that package on a laptop, and an original extension testing whether the estimated long-run learning mechanism changes when the estimation sample excludes the high-volatility 1970s.

**Underlying paper:** Farmer, Leland E., Emi Nakamura, and Jón Steinsson. "Learning about the Long Run." *Journal of Political Economy* 132(10), October 2024. DOI: [10.1086/730207](https://doi.org/10.1086/730207)

The model, the original replication package, and its estimation code are the authors' work. This repository reproduces them, documents what it takes to run them, and adds the extension described below — see [Attribution and citation](#attribution-and-citation).

---

## Research question

The paper asks why professional forecasts look the way they do: biased, with autocorrelated errors that are predictable from past forecast revisions. Its answer is that forecasters do not know the model that generates the data. A Bayesian agent learning about a *hard-to-learn* feature of the world — a permanent component whose variance share, γ, is difficult to pin down — reproduces those anomalies, for both nominal-interest-rate forecasts and CBO GDP-growth forecasts.

This repository addresses two questions:

1. **Reproducibility.** Can the authors' estimation package be run end to end, and what does it take to do so on consumer hardware?
2. **Stability.** Does the estimated γ change when the information set excludes the high-volatility 1970s?

γ is the share of the series' variation attributed to the permanent component of the unobserved-components model. A high γ means most variation is permanent, and therefore slow and hard to learn about; a low γ means most variation is transitory. It is the parameter that governs how difficult the learning problem is, which is why it is the natural object for a stability check.

## What I did

| Work | Whose |
| --- | --- |
| Unobserved-components model, Gibbs/MCMC estimation code, Monte Carlo experiments | Farmer, Nakamura & Steinsson (2024) |
| Running both MCMC estimations end to end; regenerating the figures and tables | My replication work |
| `run_replication.m` — single-command orchestration, runtime patching, figure export, table logging | My reproducibility engineering |
| Diagnosis and documentation of deviations, runtime and RAM limits, figure/table mapping | My reproducibility engineering |
| Epoch-sensitivity extension (2 estimation scripts, 2 analysis scripts, 1 figure) | My original extension |

In detail:

- **Reproduced the package's main text.** Ran the interest-rate (T-bill) and GDP unobserved-components MCMC estimations, and regenerated the main-text figures and tables, including the Monte Carlo section from the simulation draws the authors ship with the package (those do not need re-estimating).
- **Wrote `run_replication.m`**, a single-click wrapper with a master control panel that executes the four code folders in order, applies the fixes described below at runtime, exports every figure to PNG, and logs table output to a text file.
- **Diagnosed the package's environment assumptions** and documented the deviations required to run it on a laptop.
- **Mapped the code's raw output onto the published exhibits**, since the scripts print unlabelled matrices and produce diagnostic figures that are not in the paper.
- **Built the epoch-sensitivity extension** described below.

## Replication and reproducibility

Everything runs from one file:

```matlab
% run_replication.m — master control panel
run_full_tbill_estimation = true;   % set false on subsequent runs
run_full_gdp_estimation   = true;   % set false on subsequent runs
run_full_mc_estimation    = false;  % pre-computed draws provided by the authors
```

On the first run set the two estimation toggles to `true`. Once the large `.mat` outputs exist, set them back to `false` to regenerate every figure and table in minutes without re-estimating.

**Observed runtimes** (Apple M2 Pro MacBook Pro):

| Stage | Time |
| --- | --- |
| T-bill MCMC estimation | ≈ 5 hours |
| GDP MCMC estimation | ≈ 3 hours |
| Regenerating figures and tables with the toggles off | minutes |
| Each epoch estimation in the extension | ≈ 1 minute |

The two full estimations are best left to run overnight.

**Why the estimations must be re-run.** The two largest generated outputs are excluded from the repository because of GitHub's file-size limits, and both are listed in `.gitignore`:

- `T-bill results/tblResults_SlopeCoefficients_Level_SmoothL.mat` (≈ 2 GB — the authors' own note puts it "about 2 GB")
- `GDP results/gdpResults_AllCoefficients_Levels5.mat` (≈ 570 MB)

The Monte Carlo simulation draws *are* included in the authors' package, which is why `run_full_mc_estimation` stays `false`.

**Fixes the wrapper applies at runtime.** The authors' code carries environment assumptions that do not hold on another machine. The wrapper patches them in memory and restores the originals immediately afterwards, so the committed sources remain as the authors wrote them:

| File | Issue | Fix |
| --- | --- | --- |
| `Data/figure_5.m` | Hardcoded path to an author's Windows desktop (`C:\Users\shara\...\UKConsoleRate.xlsx`) | Repointed to the local copy of the spreadsheet |
| `T-bill results/tblFigs.m` | Calls `gam2Params`, but the loaded workspace provides `gamParams` | Variable name corrected before the script runs |
| `GDP results/gdpFigs.m` | The date vector loaded from `Gdp_Regression_Stats.mat` is not plot-ready on current MATLAB releases | A conversion to numeric year + quarter is inserted after the load |
| `Monte_Carlo/MonteCarlo_Results.m` | Written for the downward-biased case only | Switched between the downward-biased, unbiased, and upward-biased draws to obtain Tables 6 and 7 for all three |

**Output mapping.** The authors' scripts print raw matrices to the command window and produce more figures than the paper uses. The correspondence used here:

| Exhibit | Generated during |
| --- | --- |
| Figures 1, 2, 3, 5; Tables 1, 2 | `Data` |
| Appendix Table A.1 | `Data/Individual_regressions` — **not run** by the wrapper |
| Figures 4, 6, 7, 8; Tables 3, 4 | `T-bill results` |
| Figure 9 | `T-bill results` — **not reproduced**, see below |
| Figures 10, 11, 12 | `GDP results` |
| Table 5 (CBO rows) | `Data` |
| Table 5 (UC model rows) | `GDP results` |
| Figures 13, 14, 15; Tables 6, 7 | `Monte_Carlo` |

Root-level PNGs are named `<prefix>_<number>.png`, where the number is the figure's **internal number in the authors' scripts**, not the paper's figure number. They coincide for the main exhibits (`Data_Fig_1.png`, `Tbill_Fig_4.png`, `GDP_Fig_10.png`), but the T-bill scripts also open diagnostic figures, so `Tbill_Fig_14.png` through `Tbill_Fig_20.png` are that section's diagnostic panels — not Figures 14–20 of the paper, which come from the Monte Carlo section (`MC_Fig13_1.png` … `MC_Fig15_1.png`).

**Scope of the replication.** The wrapper reproduces the main-text exhibits: Figures 1–8 and 10–15, and Tables 1–7. It does not reproduce three parts of the authors' package:

- **Figure 9**, which the wrapper skips for RAM reasons (below).
- **The interest-rate appendix** (Appendices E, F.1, F.2), which the authors produce with `tblPriorEstimation_Direct.m`, `tblPriorEstimation_Loose.m`, and `tblPriorEstimation_LookAhead.m`. Each needs its own multi-hour estimation, and each output is then loaded into `tblRegressions.m` and `tblFigs.m` by hand — the wrapper does not drive that.
- **Appendix Table A.1**, which lives in `Data/Individual_regressions/`.

Note that `Tables_Output.txt` is the raw console transcript of a full run (captured with MATLAB's `diary`), not a set of formatted published tables: the numbers appear as unlabelled matrices in execution order, interleaved with progress messages.

## Epoch sensitivity extension

**Question.** The paper's baseline estimation uses the full historical sample, in which the 1970s are a period of unusually high macroeconomic volatility. If the learning mechanism is a stable feature of the economy, the estimated γ should not move much when that decade is excluded. If it does move, that is evidence the estimate is sensitive to which historical regime is in the sample.

**Scripts** (all in `GDP results/`, all mine):

| File | Purpose |
| --- | --- |
| `gdp_GreatMod.m` | Re-estimates the GDP unobserved-components model on the Great Moderation, 1984–2007 |
| `gdp_ModernEra.m` | Re-estimates it on the Modern Era, 1984–2019 |
| `Epoch_Analysis_Plot.m` | Kernel-density comparison of the three posterior distributions of γ → `Epoch_Analysis_Gamma.png` |
| `Epoch_Analysis_Table.m` | Summary statistics (mean, median, standard deviation, 5th and 95th percentiles) |

**Method.** Both epoch scripts reuse the authors' estimator (`recursiveGibbsUC_GDP`), hyperparameters, and seed exactly as set in their baseline script; only the sample changes. Each restricts the data to its epoch, takes the fully revised (final-vintage) GDP growth series, and runs the sampler with the same 25,000 burn-in draws and 50,000 retained draws.

This is why the two epoch scripts finish quickly. In the baseline run, the outer loop walks forward through the real-time data vintages, calling the Gibbs sampler once every four quarters from the estimation start through 2019 — 45 estimation passes, which is where the ≈3 hours goes. In the epoch scripts the estimation start is set to the end of the epoch (`estimationStart = T_epoch`), so that outer loop executes a **single** pass: one Gibbs run of the same length, which takes about a minute.

**Execution order.** The plot and table scripts read the baseline output as well as the two epoch outputs, so the baseline `.mat` must exist first:

1. `gdpPriorEstimation.m` (baseline) — or reuse an existing `gdpResults_AllCoefficients_Levels5.mat`; no need to re-run it if it is already there.
2. `gdp_GreatMod.m`, then `gdp_ModernEra.m`
3. `Epoch_Analysis_Table.m`, then `Epoch_Analysis_Plot.m`

## Main findings

![Posterior distributions of the learning parameter gamma across the baseline sample, the Great Moderation, and the Modern Era](GDP%20results/Epoch_Analysis_Gamma.png)

*Posterior distributions of γ. The three distributions occupy the same region and overlap heavily; the differences between them are small relative to their spread.*

Posterior statistics for γ, computed with the same definitions used by `Epoch_Analysis_Table.m` (NaNs omitted):

| Sample | Mean γ | Median | Std. dev. | 90% interval | Draws |
| --- | --- | --- | --- | --- | --- |
| Baseline (1947–2019) | **0.579** | 0.580 | 0.053 | 0.490 – 0.664 | 8,849,884 |
| Great Moderation (1984–2007) | **0.544** | 0.544 | 0.054 | 0.455 – 0.633 | 49,978 |
| Modern Era (1984–2019) | **0.548** | 0.549 | 0.054 | 0.459 – 0.638 | 49,929 |

**What this says.** Excluding the high-volatility 1970s moves the point estimate of γ down by about 0.03 — less than one posterior standard deviation (0.054) — and the two epoch estimates are almost identical to each other (0.544 versus 0.548). The 90% intervals overlap almost completely with the baseline's. The substantive reading is therefore stability, not change: outside the 1970s the large permanent-variance share remains, and the estimate shifts only slightly. The data do not sharply separate the epochs, and the difference is small relative to posterior uncertainty.

**How far this can be pushed.** The comparison is indicative rather than decisive, and it should not be read as a test of whether the learning mechanism "changed":

- The three posteriors overlap substantially.
- The epoch samples are far shorter than the baseline (96 and 144 quarters, versus 292).
- The epoch estimates are deliberately *not* a like-for-like re-estimation of the baseline object, for two reasons visible in the code: the epoch scripts collapse the real-time vintage dimension (they use the fully revised series rather than the vintage panel), and the estimator's hardcoded starting index means each single Gibbs pass effectively uses observations 51 onward **within** its epoch — roughly 1996–2007 and 1996–2019 — rather than the full labelled window. The baseline statistic also averages the recursive quarterly posteriors, whereas each epoch statistic comes from a single final-quarter posterior.
- Nothing here speaks to causality. The exercise is a descriptive check on how stable an estimated parameter is across sample windows.

![Parameter densities for the GDP unobserved-components model](GDP_Fig_10.png)

*One of the replicated exhibits, `GDP_Fig_10.png`, exported at 300 dpi by the wrapper from the authors' `GDP results/gdpFigs.m`. The middle-left panel is γ.*

## Methods and technical stack

- **MATLAB** — this replication was run and verified on R2024b (macOS).
- **Bayesian estimation** of an unobserved-components model by Gibbs sampling: 25,000 burn-in draws + 50,000 retained draws, `stepSize = 4`, seed 15284, on the authors' optimal hyperparameters.
- **Toolboxes required by the authors' code:** Statistics and Machine Learning Toolbox (`ksdensity`, `prctile`, `betapdf`, `normpdf`), Econometrics Toolbox (`ssm`, `simsmooth`), and Optimization Toolbox (`fminunc`). A commented-out `particleswarm` call would additionally require Global Optimization Toolbox.
- **Data** ship with the replication package (CSV, MAT, XLSX); nothing is downloaded at run time.

## Repository structure

```
run_replication.m            # my orchestration wrapper (master control panel)
Paper.pdf                    # the published paper
Authors_Original_Readme..pdf # the authors' own package instructions (December 2023)

Data/                        # authors' code — Figures 1, 2, 3, 5; Tables 1, 2; App. Table A.1
T-bill results/              # authors' code — Figures 4, 6, 7, 8, 9; Tables 3, 4
GDP results/                 # authors' code — Figures 10, 11, 12; Table 5 (UC rows)
  gdp_GreatMod.m             #   } my epoch-sensitivity extension
  gdp_ModernEra.m            #   }
  Epoch_Analysis_Plot.m      #   }
  Epoch_Analysis_Table.m     #   }
  Epoch_Analysis_Gamma.png   #   }
Monte_Carlo/                 # authors' code — Figures 13, 14, 15; Tables 6, 7

Tables_Output.txt            # raw console transcript of a full run (mine)
Data_Fig_*.png, Tbill_Fig_*.png, GDP_Fig_*.png, MC_Fig*.png   # figures exported by my run
.gitignore
```

Everything under `Data/`, `T-bill results/`, `GDP results/`, and `Monte_Carlo/` other than the five epoch files listed above is the authors' original replication package, redistributed unmodified — including the dotfiles that shipped with it (`.cshrc`, `.xinitrc`, `.nx/`, `.bash_profile`, `.viminfo` inside `Monte_Carlo/`). The `.mat` files committed inside those folders are the small data and results files that came with the package; the two large estimation outputs are excluded, and so are the extension's own regenerable outputs (`gdpResults_GreatMod.mat`, `gdpResults_ModernEra.mat`).

## Reproducing the analysis

```matlab
% 1. Full replication (hours — leave it running overnight)
%    In run_replication.m set run_full_tbill_estimation = true and
%    run_full_gdp_estimation = true, then run the script. Figures and tables
%    are regenerated and the large .mat outputs are created.

% 2. After the estimations exist: set both estimation toggles back to false
%    and run run_replication.m again to regenerate every figure and table
%    in minutes.

% 3. Epoch-sensitivity extension (about a minute per epoch), from GDP results/:
gdp_GreatMod        % needs Gdp_Data.mat; writes gdpResults_GreatMod.mat
gdp_ModernEra       % needs Gdp_Data.mat; writes gdpResults_ModernEra.mat
Epoch_Analysis_Table
Epoch_Analysis_Plot
```

Requirements: MATLAB with the toolboxes listed above, plus enough disk space for the two large `.mat` outputs (≈ 2 GB and ≈ 570 MB).

**Figure 9 is not reproduced.** `T-bill results/tblPriorEstimation_Break.m` allocates a 275 × 120 × 75,000 double array — `NaN(T, maxHorizon, B)` with the T-bill data's `T = 275`, `maxHorizon = 4*30 = 120`, and `B = 75,000` — which is about **18.4 GiB (19.8 GB)** in a single allocation, well beyond the RAM of a laptop. The wrapper skips that script rather than crashing, and Figure 9 is consequently absent from the exported figures.

## Attribution and citation

**The model, the estimation code, and the original replication package are the work of Leland E. Farmer, Emi Nakamura, and Jón Steinsson.** They are redistributed here unmodified; the fixes described above are applied at runtime and never committed. The authors' own instructions for the package are included as `Authors_Original_Readme..pdf` and remain the authoritative guide to their code.

**My contributions to this repository:** `run_replication.m`; the root-level exported figures and `Tables_Output.txt`; the epoch-sensitivity extension (`GDP results/gdp_GreatMod.m`, `gdp_ModernEra.m`, `Epoch_Analysis_Plot.m`, `Epoch_Analysis_Table.m`, `Epoch_Analysis_Gamma.png`); and `README.md`.

To cite the underlying research:

```text
Farmer, Leland E., Emi Nakamura, and Jón Steinsson. 2024.
"Learning about the Long Run." Journal of Political Economy 132 (10).
DOI: 10.1086/730207
```

There is no license file here: the authors' code and the published paper are redistributed for academic replication purposes, and their terms remain theirs.

## Context

This project was completed as the term project for **ECN 726 Econometrics II** at Arizona State University. The replication model and the original replication package are attributable to Farmer, Nakamura & Steinsson (2024); the reproducibility work and the epoch-sensitivity extension described above are my contributions.
