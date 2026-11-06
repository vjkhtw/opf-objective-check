# AC-OPF and DC-OPF check: Pyomo vs PYPOWER

Two notebooks compare the total generation cost and the solve time of a Pyomo formulation against PYPOWER on six IEEE cases from [PGLib-OPF](https://github.com/power-grid-lib/pglib-opf):

| Notebook | Problem | Pyomo solver | PYPOWER call |
|---|---|---|---|
| [`05_acopf_objective_check.ipynb`](notebooks/05_acopf_objective_check.ipynb) | AC-OPF (non-convex NLP) | IPOPT | `runopf` |
| [`06_dcopf_objective_check.ipynb`](notebooks/06_dcopf_objective_check.ipynb) | DC-OPF (LP) | HiGHS | `dcopf` |

Only the objective and the run time are compared. All solves converged (IPOPT/HiGHS: optimal, PYPOWER: success). All results below come from running the notebooks in a Kaggle notebook (CPU only, no accelerator).

## Earlier work

This builds on my earlier PJM 5-bus project:

- **[pjm5-opf](https://github.com/vjkhtw/pjm5-opf)** (GitHub repo): economic dispatch, DC-OPF and AC-OPF on the PJM 5-bus case in Pyomo, cross-checked against PYPOWER. The notebooks there are `01_dcopf_pyomo`, `02_dcopf_pypower_reference`, `03_acopf_pyomo` and `04_acopf_pypower_reference`; the notebooks here continue the numbering as `05` and `06`.
- **[Project write-up](https://vesuvius-khatiwada.super.site/economic-dispatch-dc-opf-and-ac-opf-on-the-pjm-5-bus-network)**: the full write-up of that project, with figures.
- **Theory notes:** [ED, DC-OPF, and AC-OPF](https://vesuvius-khatiwada.super.site/ed-dc-opf-and-ac-opf) (the three formulations, congestion, and reading duals as LMPs and line shadow prices) and [Optimization Foundations for Energy System Modelling](https://vesuvius-khatiwada.super.site/optimization-foundations-for-energy-system-modelling) (convexity, Lagrangians, KKT conditions).

How the two relate:
- The AC model in `05` is the formulation from `03_acopf_pyomo.ipynb`, and the DC model in `06` is the one from `01_dcopf_pyomo.ipynb`. Both are extended with transformer tap ratios and bus shunts so that they also run on the IEEE cases (see "Why taps and shunts are in the models" below).
- Each notebook begins with a 5-bus sanity check against the earlier reference costs: $17,551.89/h for AC-OPF and $17,479.90/h for DC-OPF.
- The scope here is narrower and wider at the same time. The earlier project compares dispatch, LMPs, shadow prices and losses on one 5-bus network. This one compares only the objective and the solve time, but across six IEEE cases.

## Results: objective

**AC-OPF.** Costs agree to at most 5.2e-8 relative (largest absolute difference $0.0047/h). IPOPT starts from a flat start and PYPOWER from the case-file setpoints. Agreement means both reach the same local optimum; AC-OPF is non-convex, so it is not a proof of a global optimum.

| Case | Pyomo/IPOPT ($/h) | PYPOWER ($/h) | Abs. diff | Rel. diff |
|---|---|---|---|---|
| IEEE 14 | 2,178.0804 | 2,178.0805 | 1.1e-4 | 5.2e-8 |
| IEEE 30 | 8,208.5154 | 8,208.5152 | 2.7e-4 | 3.3e-8 |
| IEEE 39 (EPRI) | 138,415.5613 | 138,415.5633 | 2.0e-3 | 1.4e-8 |
| IEEE 57 | 37,589.3381 | 37,589.3390 | 8.5e-4 | 2.3e-8 |
| IEEE 118 | 97,213.6065 | 97,213.6079 | 1.4e-3 | 1.4e-8 |
| IEEE 162 (DTC) | 108,075.6435 | 108,075.6482 | 4.7e-3 | 4.4e-8 |

**DC-OPF.** Costs agree to at most 9.0e-12 relative (largest absolute difference $1.8e-7/h). DC-OPF is a linear program, so both solvers find the global optimum and the differences are numerical tolerance.

| Case | Pyomo/HiGHS ($/h) | PYPOWER ($/h) | Abs. diff | Rel. diff |
|---|---|---|---|---|
| IEEE 14 | 2,051.5263 | 2,051.5263 | 1.9e-8 | 9.0e-12 |
| IEEE 30 | 7,504.4405 | 7,504.4405 | 1.8e-8 | 2.3e-12 |
| IEEE 39 (EPRI) | 136,816.1561 | 136,816.1561 | 1.3e-7 | 9.4e-13 |
| IEEE 57 | 34,772.9479 | 34,772.9479 | 1.2e-7 | 3.4e-12 |
| IEEE 118 | 93,132.6793 | 93,132.6793 | 6.8e-8 | 7.3e-13 |
| IEEE 162 (DTC) | 101,268.2940 | 101,268.2940 | 1.8e-7 | 1.8e-12 |

## Results: solve time

**These times were measured in a Kaggle notebook (CPU only, no accelerator), and they will be different on other machines.** Kaggle sessions run on shared, virtualised hardware, so absolute times also vary from one session or run to the next, and they will differ on your own computer, Colab, or a different Kaggle session. The ratios between the two workflows are more portable than the absolute values.

Each number is the median wall-clock time per run after one warm-up run (AC: 5 runs, DC: 20 runs), from the timing cell at the end of each notebook.

What is timed:
- **Pyomo total** = building the Pyomo model + the `solver.solve` call. The solve call includes passing the model to the solver and reading the result back, not only the solver's iterations.
- **PYPOWER total** = the whole `runopf` / `dcopf` call, including its own case conversion and set-up, not only its interior-point solve.

So the totals compare two complete workflows, not the bare solvers.

**AC-OPF (seconds)**

| Case | Buses | Pyomo/IPOPT total | PYPOWER total |
|---|---|---|---|
| IEEE 14 | 14 | 0.067 | 0.572 |
| IEEE 30 | 30 | 0.115 | 0.508 |
| IEEE 39 (EPRI) | 39 | 0.155 | 0.872 |
| IEEE 57 | 57 | 0.184 | 0.640 |
| IEEE 118 | 118 | 0.469 | 1.225 |
| IEEE 162 (DTC) | 162 | 0.774 | 1.410 |

**DC-OPF (milliseconds)**

| Case | Buses | Pyomo build | HiGHS solve | Pyomo total | PYPOWER total |
|---|---|---|---|---|---|
| IEEE 14 | 14 | 2.8 | 12.5 | 15.5 | 33.9 |
| IEEE 30 | 30 | 5.4 | 17.9 | 23.3 | 42.0 |
| IEEE 39 (EPRI) | 39 | 5.4 | 19.6 | 25.0 | 76.2 |
| IEEE 57 | 57 | 8.5 | 29.1 | 38.1 | 48.9 |
| IEEE 118 | 118 | 18.4 | 55.6 | 74.8 | 105.4 |
| IEEE 162 (DTC) | 162 | 32.8 | 73.4 | 106.0 | 223.4 |

Takeaways for these cases on this Kaggle run:
- Every solve takes under 1.5 s (AC) or under 225 ms (DC), with either workflow.
- The Pyomo workflow was faster than PYPOWER in every case: 1.8x to 8.5x for AC (the gap generally narrows as the network grows) and 1.3x to 3.0x for DC.
- The Pyomo DC solve is about 4x to 7x faster than the Pyomo AC solve, but those are different problem types and different solvers, so this is not a like-for-like solver comparison.
- Nothing here says how the two scale to much larger networks.

## How to run

Requires Python 3.10 or newer. From the repository root, install the dependencies listed in [`requirements.txt`](requirements.txt), install the IPOPT extension used by the AC-OPF notebook, and launch either notebook:

```bash
python -m pip install -r requirements.txt
idaes get-extensions
jupyter lab notebooks/05_acopf_objective_check.ipynb
```

For the DC-OPF notebook, launch `jupyter lab notebooks/06_dcopf_objective_check.ipynb` instead. In Jupyter, run all cells. The notebooks download case files from the pglib-opf GitHub repository at the pinned commit (`PGLIB_COMMIT` in each notebook), so they need internet access.

To add a case, append its pglib name (for example `"30_ieee"`) to the `CASES` list.

## Data pipeline: feeding the functions

```
pglib_opf_case<n>.m  ->  read_matpower()  ->  case dict  ->  solve_pyomo_acopf(case) / solve_pyomo_dcopf(case)
                                                         ->  solve_pypower(case)     / solve_pypower_dc(case)
```

All four functions take the same dictionary:

```python
case = {
    "baseMVA": 100.0,
    "bus":     np.array(...),   # one row per bus
    "gen":     np.array(...),   # one row per generator
    "gencost": np.array(...),   # one row per generator
    "branch":  np.array(...),   # one row per line/transformer
}
```

The arrays use MATPOWER column order, as in the `.m` case files. Units: powers in MW/MVAr, voltages and impedances in p.u. on `baseMVA`. `read_matpower()` builds the dictionary from a pglib `.m` file; to use your own data, build it the same way (for example from CSV files with `np.loadtxt`).

| Array | Column (0-based) | AC | DC |
|---|---|---|---|
| `bus` | 0 number, 1 type (3 = reference), 2 Pd | yes | yes |
| `bus` | 4 Gs | yes | yes (added to demand) |
| `bus` | 3 Qd, 5 Bs, 11 Vmax, 12 Vmin | yes | no |
| `gen` | 0 bus, 8 Pmax, 9 Pmin, 7 status (must be 1) | yes | yes |
| `gen` | 3 Qmax, 4 Qmin | yes | no |
| `gencost` | 5 linear cost in $/MWh; 4 quadratic and 6 constant terms (must be 0) | yes | yes |
| `branch` | 0 from bus, 1 to bus, 3 x, 5 rateA, 8 tap ratio, 11 angmin, 12 angmax (degrees) | yes | yes |
| `branch` | 2 r, 4 b | yes | no |
| `branch` | 9 phase shift (must be 0), 10 status (must be 1) | yes | yes |

The functions assert on the "must be" conditions, so an unsupported case fails loudly instead of giving a wrong number.

Not included: IEEE 24 and 73 (quadratic generator costs) and IEEE 300 (phase-shifting transformers). Including them would need a change to the objective or the branch model.

## Why taps and shunts are in the models

The IEEE cases contain transformer tap ratios and bus shunts, which the 5-bus case does not. They are part of the network equations, not extra constraints. Without them the Pyomo model would be solving a different network from PYPOWER.

| | AC-OPF | DC-OPF |
|---|---|---|
| Tap ratio τ (0 in the file means τ = 1) | `Y_ff = (y + j·b_c/2)/τ²`, `Y_ft = Y_tf = −y/τ`, `Y_tt = y + j·b_c/2` | branch flow `(θi − θj)/(x·τ)` |
| Bus shunt (Gs, Bs) | extra load `Gs·V²` in the active balance and `−Bs·V²` in the reactive balance | `Gs` added to the demand at the bus; `Bs` not used |

`Gs` is 0 in all six cases (their shunts are purely susceptance, `Bs`), so the DC `Gs` term matches PYPOWER's formulation but is not exercised by these cases.

| Case | Tapped branches | Buses with shunts |
|---|---|---|
| IEEE 14 | 3 | 1 |
| IEEE 30 | 7 | 2 |
| IEEE 39 | 11 | 0 |
| IEEE 57 | 17 | 3 |
| IEEE 118 | 11 | 14 |
| IEEE 162 | 91 | 34 |
