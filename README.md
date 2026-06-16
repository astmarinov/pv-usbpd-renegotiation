# Direct Photovoltaic-to-USB-PD Regulation by Curve-Free Supervisory Current-Limit Renegotiation

Supplementary data and trained models for the paper:

> **Direct Photovoltaic-to-USB-PD Regulation by Curve-Free Supervisory Current-Limit Renegotiation**
> A. Marinov, N. Hinov

This repository contains the trained decision-tree controller, the held-out validation panel models, and the complete validation run data behind the results reported in the paper. It allows independent inspection of the controller logic and verification of the reported metrics.

---

## What is here

| Path | Contents |
|------|----------|
| `Trees/` | The two trained decision trees that constitute the supervisory controller, as human-readable rule files. |
| `Test Panels/` | Photovoltaic I–V lookup tables for the ten held-out validation panels (MATLAB `.mat`). |
| `Results/` | Full time-series of all 360 validation runs, split into one ZIP per panel (`Panel 1.zip` … `Panel 10.zip`); each archive holds that panel's 36 runs (4 contracts × 9 scenarios), one CSV per run. |
| `per_run_metrics.json` | Per-run summary metrics for all 360 runs (the data behind Tables 7 and 8 in the paper). |
| `LICENSE` | License for the data and models (CC BY 4.0). |

The deployed controller is fully specified by the two rule files in `Trees/` together with the feature definitions and supervisory guard logic in **Section 2 of the paper** (the control law and guard parameters are in Sections 2.3–2.4 and Table 3).

---

## The controller in brief

The system regulates a battery-free photovoltaic source delivering a fixed-voltage USB Power Delivery (USB-PD) contract through a four-switch buck–boost converter. Rather than tracking the maximum power point, the controller holds the negotiated output voltage and **renegotiates the contracted current** up or down to keep the operating point within the feasible region as irradiance varies.

Two decision trees act on normalised, panel-agnostic features (curve-free: only open-circuit voltage `Voc`, short-circuit current `Isc`, and live terminal measurements are used):

- **Tree 1** (`tree1_mc_rules.txt`) — a regressor that decides decisive single-step current **reductions**.
- **Tree 2** (`tree2_mc_rules.txt`) — a binary classifier that permits conservative current **increases** (recovery).

Both are wrapped in priority-ordered deterministic guards (Section 2.4 of the paper). The trees were trained on a corpus of datasheet-reconstructed panel models and evaluated on the ten **held-out** panels in `Test Panels/`, none of which appeared in training.

### Reading and applying the trees

The rule files in `Trees/` are exported in scikit-learn `export_text` format
(`|---` lines are branches; `value: [x]` lines are leaf outputs). They act on
dimensionless features computed from the panel constants (`Voc`, `Isc`) and the
live terminal measurements (`Vpv`, `Ipv`, `Vout`, contracted current `Ineg`).
In all validation runs the curve-free knee estimate `Vknee = 0.78 · Voc` is used.

| Feature | Definition |
|---------|------------|
| `vpv_norm` | `Vpv / Voc` |
| `ipv_norm` | `Ipv / Isc` |
| `i_neg_norm` | `Ineg / Isc` |
| `ratio` | `Vout / Vpv` (conversion ratio ρ = D/(1−D)) |
| `duty` | `Vout / (Vout + Vpv)` |
| `dvpv_dt_norm` | normalised dVpv/dt (Tree 1 only) |
| `p_slack_norm` | `(Vpv·Ipv − Vout·Ineg) / (Voc·Isc)` (Tree 2 only) |

**Note that `margin_norm` denotes a different quantity in each tree:**
- In **Tree 1** it is a ratio-derived stress proxy, `clamp(a·ratio + b, −1.0, 0.1)` with `a = −0.02114`, `b = 0.01468`.
- In **Tree 2** it is the voltage headroom above the knee, `(Vpv − Vknee) / Voc`.

- **Tree 1** (`tree1_mc_rules.txt`) — features in file order: `ratio`,
  `dvpv_dt_norm`, `i_neg_norm`, `margin_norm` *(stress-proxy form)*. Output is the
  signed current adjustment in units of `Isc`; a negative output triggers a
  step-down (subject to the voltage and comfort guards of Section 2.4).
- **Tree 2** (`tree2_mc_rules.txt`) — features in file order: `vpv_norm`,
  `ipv_norm`, `i_neg_norm`, `ratio`, `duty`, `margin_norm` *(headroom form)*,
  `p_slack_norm`. A leaf `value` > 0.5 permits a step-up (subject to the inhibit
  timer, safe-voltage gate, settled condition, and soft ceiling of Section 2.4).

The guard thresholds and tuning parameters are listed in **Table 3** of the paper.
Reported accuracy: Tree 1 — MAE 0.035·`Isc`, R² 0.980; Tree 2 — 99.3 % accuracy,
0.94 % false-YES, 0.51 % false-NO.

---

## Validation panels (`Test Panels/`)

Ten panels, anonymised as `Panel 1.mat` … `Panel 10.mat`, matching the panel numbering in the paper. Panels 1–9 are correctly sized; **Panel 10 is a deliberately undersized 130 W module** included as a graceful-failure boundary case.

Each `.mat` file contains:

| Variable | Shape | Meaning |
|----------|-------|---------|
| `Isurf_panel_T25` | 501 × 11 × 3 | Panel current surface at 25 °C: 501 voltage points × 11 normalised irradiance levels (0.0–1.0) × 3 (the lookup axes used by the simulation). |
| `panel_Voc` | 1 × 1 | Open-circuit voltage (V). |
| `panel_Isc` | 1 × 1 | Short-circuit current (A). |

The controller receives only `panel_Voc` and `panel_Isc` per panel; the full surface is used by the plant model to generate the operating-point response. No maximum-power-point values are provided to the controller (the minimal-information deployment case described in Section 3.1).

---

## Validation run data (`Results/`)

The 360 runs are provided as one ZIP archive per panel: `Results/Panel 1.zip`
through `Results/Panel 10.zip`. Each archive contains that panel's 36 runs
(4 contracts × 9 scenarios). Individual CSV files are named:

```
Panel <N>_<contract>_<scenario>.csv
```

where `<N>` is 1–10, `<contract>` ∈ {`20V`, `28V`, `36V`, `48V`}, and `<scenario>` is one of the nine irradiance profiles (see below). Each file is a 60-second run sampled at the controller timestep (15 000 rows).

**Columns:**

| Column | Units | Meaning |
|--------|-------|---------|
| `t` | s | Time. |
| `vout` | V | Converter output voltage (the regulated USB-PD contract voltage). |
| `vpv` | V | Photovoltaic panel terminal voltage. |
| `ineg` | A | Currently contracted (negotiated) current limit. |
| `ipv` | A | Photovoltaic panel current. |
| `irr` | – | Normalised irradiance (0–1). |
| `p_avail` | W | Power available from the panel at the current operating point. |
| `p_deliv` | W | Power delivered to the load (`vout` × `ineg`). |
| `ceil` | A | Internal soft current ceiling used by the recovery logic. |

**Irradiance scenarios:**

| Scenario | Description |
|----------|-------------|
| `steady_full` | Constant 1.0 p.u. |
| `steady_mid` | Constant ≈ 0.5 p.u. |
| `boost_band` | Sustained operation in the 0.3–0.6 p.u. band (near the boost-ratio limit). |
| `rampchar_fast` / `_med` / `_slow` | Controlled ramps between 1.0 and 0.4 p.u. at three rates. |
| `day_partly_cloudy` / `_deteriorating` / `_clearing` | Multi-level day-like profiles. |

---

## Per-run metrics (`per_run_metrics.json`)

Nested as `metrics["Panel N"]["<contract>"]["<scenario>"]`, each run reporting:

| Field | Meaning |
|-------|---------|
| `vout_ss` | Steady-state output voltage (V). |
| `vpv_ss` | Steady-state panel voltage (V). |
| `ineg_ss` | Steady-state contracted current (A). |
| `ineg_std` | Std. dev. of contracted current over the run (oscillation severity). |
| `vpv_min` | Minimum panel voltage over the run, post-start-up window (V). |
| `p_avail_ss`, `p_deliv_ss`, `p_margin_ss` | Steady-state available / delivered / margin power (W). |
| `p_util_pct` | Power utilisation (delivered / available, %). |
| `n_steps` | Number of renegotiation events. |
| `recovery_s` | Recovery time after an irradiance drop (s; 0 where not applicable). |
| `oscillating` | Boolean oscillation flag. |
| `collapsed` | Boolean contract-loss flag. |

These are the values aggregated in Tables 7 and 8 of the paper.

---

## Reproducing / verifying the reported metrics

The trees and the validation data together let you verify the paper's claims without the simulator:

- Confirm that applying the published `Trees/` rules to the panel curves, under the guard logic of Section 2.4, reproduces the renegotiation behaviour seen in the `Results/` time-series.
- Recompute any cell of Tables 7 and 8 from `per_run_metrics.json` (e.g. per-contract mean utilisation, collapse counts).

The supervisory controller implementation, the co-simulation harness, the 366-panel datasheet **training** corpus, and the PLECS converter model are available from the corresponding author on reasonable request.

---

## Citation

If you use this data or the trained models, please cite the paper:

```bibtex
@article{marinov_pv_usbpd_2026,
  author  = {Marinov, Angel and Hinov, Nikolay},
  title   = {Direct Photovoltaic-to-USB-PD Regulation by Curve-Free Supervisory Current-Limit Renegotiation},
  journal = {TBD},
  year    = {2026},
  note    = {Supplementary data: https://github.com/USER/REPO}
}
```

## License

Data and models in this repository are released under the Creative Commons Attribution 4.0 International License (CC BY 4.0). See [`LICENSE`](LICENSE).
