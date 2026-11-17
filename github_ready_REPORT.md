# RD-EAGA repository: verification and clean-up report

Environment: Linux, 2 CPU cores, Python 3.13, torch 2.10.0 (cu128 wheel installed system-wide; published results used 2.10.0+cpu), networkx 3.6.1 / numpy 2.3.5 / pandas 2.2.3 via PYTHONPATH pins; matplotlib 3.11.2, scipy 1.18.1, statsmodels 0.15.0 (system, newer than pins). All runs were done in scratch copies (`scratch_run/`), never in `github_ready/`. Originals untouched (original SHA256SUMS verified OK).

## Checks
| # | Check | Result | Command / detail | Time |
|---|---|---|---|---|
| 1 | Original SHA256SUMS | PASS | `sha256sum -c SHA256SUMS.txt` in the original | - |
| 2 | Imports + bundled topologies load (5 GML: 12/15, 22/36, 39/61, 39/86, 50/88 nodes/edges) | PASS | `load_topology(name,'bundled')` | 2 s |
| 3 | Pins exist on PyPI and resolve together | PASS | `pip index versions` per pin; `pip install --dry-run --ignore-installed -r requirements` (py3.13) | 4 s |
| 4 | Tiny end-to-end (train+benchmark+ablation) | PASS | `python src/eaga_reproduction.py --output X --quick --topology-source bundled --train-samples 20 --epochs 2 --repetitions 1 --ablation-repetitions 1 --loads 10 --workers 1` | 15 s |
| 5 | One seed (20260803), shipped checkpoint, 4 loads, vs results/benchmark_raw.csv | PASS | README single-seed command, `--workers 2`; 80/80 rows matched. Dijkstra/CGA/EAGA/HGA-ACO: max abs diff 0 in success/delay/throughput/generations, 3.6e-15 in cost. RD-EAGA differs (max |d success| 0.04, cost 1.26) = MC-dropout global RNG | 15 s |
| 6 | Full `src/reproduce_executed_profile.sh` (retrain 200/15, 10 seeds, 10-rep ablation, postprocess) | PASS after fix | `PYTHON=python3 bash src/reproduce_executed_profile.sh`. First attempt FAILED in postprocess (topology plot read `data/<name>.gml`); fixed; postprocess rerun OK. Merged 800 rows vs shipped: all 640 non-RD rows exact (0 diff in all metrics); RD-EAGA means success 0.4176 vs 0.4178, cost 20.735 vs 20.750, delay 44.776 vs 44.776; ablation_summary success means identical to 3 decimals | 3 min 53 s (+5 s postprocess) |
| 7 | Retrained checkpoint = shipped checkpoint | PASS | `scripts/retrain_checkpoint_check.py`: MAE 4.152749, MAPE 0.274467, loss 0.198768, max weight diff 6.3e-7 | 100 s |
| 8 | `src/postprocess_results.py` | PASS after fix | needs `runs/run1..10` (written by the .sh); regenerates results/*.csv + figures/. Legacy Wilcoxon-based tables | 5 s |
| 9 | `src/enhancement_checks.py` | PASS | run after postprocess. Holm/BH, row-level bootstrap (tables 10/11/15 legacy), calibration, random-compute robustness, janos ablation. Ablation/Table15: only runtime differs; calibration to 3e-6; RD-EAGA-dependent entries differ slightly (stochastic) | 17 s |
| 10 | `src/update_manuscript.py` | FAIL (by design) | FileNotFoundError: needs `EAGA_Extended_Reproducible_Study.docx`, not shipped; edits a stale draft; contains placeholder e-mail. REMOVED | - |
| 11 | `scripts/seed_cluster_analysis.py` | PASS | `--raw results/benchmark_raw.csv --out X`; all 4 tables identical (max abs diff 0.0) to `supp/seed_cluster_outputs` | 1 s |
| 12 | `scripts/train_size_sensitivity.py` | PASS | smoke `--n-seeds 1 --samples 50 --epochs 5`, plus one full seed (6 configs): val MAE/loss bit-identical to shipped rows for seed 20260803. Full 5 seeds not run | 19 s / 16 s |
| 13 | `scripts/generation_budget_sensitivity.py` | PASS | full run (10 seeds x 2 regimes x 4 budgets x 3 methods, 240 rows): CGA/EAGA identical to shipped CSV; RD-EAGA differs (stochastic). `--replay` smoke: Dijkstra/CGA/EAGA/HGA-ACO diff <=7e-15, RD-EAGA differs | 23 s; 8 s |
| 14 | `scripts/make_figures.py` | PASS after fix | 12 figures (PDF+PNG) | 5 s |
| 15 | Final zip: unzip, `sha256sum -c`, smoke test (#4) and seed_cluster | PASS | | ~20 s |
| 16 | Privacy grep (`/home`, `/tmp`, `Users`, `@`, e-mail regex, PDF/PNG metadata) | PASS | only matches: Python decorators, BibTeX (removed), `/tmp` in usage examples | - |
| 17 | File sizes | PASS | largest file < 1 MB; tree ~26 MB, zip 23.5 MB | - |

Not verified: Table 21 with all 5 seeds; GPU runs; other OS/Python versions; exact `torch+cpu` build (system torch is cu128); pinned matplotlib/scipy/sklearn/statsmodels versions were checked for resolvability, not executed (system had newer versions); the full-parameter profile (Appendix A.3) was not run; SHA256SUMS lists files, not a signed manifest.

## Code changes (scientific behaviour unchanged)
1. `src/eaga_reproduction.py`: replaced by the patched version (loader searches `data/` and `data/topologies/`, `--topology-source bundled`, surrogate warning). Same as `patches/load_topology.patch`.
2. `src/postprocess_results.py`: `DATA = data/topologies` (fallback `data/`) - fixed FileNotFoundError.
3. `src/reproduce_executed_profile.sh`: `--topology-source bundled` instead of `auto`; `PYTHON`/`WORKERS` env vars (defaults `python`, 4); warning header; comments.
4. `scripts/retrain_checkpoint_check.py`: rewritten (was hard-coded `../code/FINAL...` and `pins` paths, positional args): argparse (--repo/--out/--seed/--samples/--epochs), trains into a temp dir, loads the shipped checkpoint and reports max weight difference.
5. `scripts/make_figures.py`: removed hard-coded `../code/FINAL...`, `figs2/`, `ts_seeds_summary.csv`; added --repo/--out/--training-size-summary; no longer writes the manuscript-slot copies (`imageN.jpeg/png`); Fig. 10 now reads `results/seed_cluster/table19_seed_cluster_tests.csv` instead of hard-coded numbers (values identical).
6. `scripts/train_size_sensitivity.py`, `scripts/generation_budget_sensitivity.py`: `--repo`/`--out` now optional (defaults: script's repo, results/...); added `--n-seeds`, `--samples/--epochs` resp. `--budgets/--demands/--replay-only` (defaults reproduce the original protocol).
7. `scripts/seed_cluster_analysis.py`: unchanged.
8. `environment/requirements.txt`: pinned list; python-docx dropped (only the removed script used it); `pillow>=10` added (make_figures); comments on CPU torch. `scikit-learn` is pinned per the manuscript but is not imported anywhere (kept; owner may drop).

## Files added
README.md (rewritten), CHANGELOG.md, CITATION.cff (TODO authors), .gitignore, results/README.md, scripts/*, patches/load_topology.patch, results/seed_cluster/*, results/training_sensitivity_seeds*.csv, results/generation_sensitivity_seeds*.csv, results/benchmark_replay_check.csv, results/checkpoint_retrain_check.json, figures/* (24 files), regenerated SHA256SUMS.txt.

## Files removed / moved
- `paper/` (tex, bib, citation audit): earlier draft, contains no seed-cluster analysis, different Table 19, placeholder acknowledgement/conflict text. Removed (owner can restore from the original).
- `src/update_manuscript.py`: see check 10.
- `src/__pycache__/*.pyc` (cpython-313), `data/*.gml` duplicates (byte-identical to `data/topologies/`).
- Moved to `results/superseded/`: hierarchical_blocked_stats.csv, failure_aware_utility.csv, training_sensitivity.csv, generation_sensitivity.csv (README notes "do not cite"; the owner may delete them outright).
- Kept deliberately: `results/core/` (duplicates many top-level CSVs, 4.6 MB), `results/seed_runs/` (18 MB of PNGs), legacy row-level `table10/11_bootstrap_ci.csv`, `statistical_tests.csv` (documented as legacy in results/README.md; manuscript Tables 10/11/19 come from `results/seed_cluster/`).

## Owner TODO
- CITATION.cff: author names, ORCID, release date; add `repository-code` and `doi` after the Zenodo release; README: repository URL and DOI placeholders.
- LICENSE holder is "RD-EAGA authors" - replace with real name(s)/institution.
- Check SNDlib/TopoHub terms before redistributing `data/topologies/*.gml`.
- Decide whether to delete `results/superseded/` and legacy row-level tables; decide whether to drop scikit-learn pin.
- Manuscript Appendix A.2 says `bash reproduce_executed_profile.sh` and `pip install -r requirements.txt`; actual paths are `src/reproduce_executed_profile.sh` and `environment/requirements.txt`. Table F1 caption in the paper says one seed, not retained; the repository now has ten seeds (update text).
- Old statement in the paper "seed-cluster analysis script ... will be deposited" is now satisfied by `scripts/`.
- Create a GitHub release after pushing so Zenodo can mint a DOI.
