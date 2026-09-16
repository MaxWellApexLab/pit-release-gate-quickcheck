# pit-release-gate

[![CI](https://github.com/MaxWellApexLab/pit-release-gate/actions/workflows/ci.yml/badge.svg)](https://github.com/MaxWellApexLab/pit-release-gate/actions/workflows/ci.yml)
[![codecov](https://codecov.io/gh/MaxWellApexLab/pit-release-gate/branch/master/graph/badge.svg)](https://codecov.io/gh/MaxWellApexLab/pit-release-gate)
[![PyPI](https://img.shields.io/pypi/v/pit-release-gate)](https://pypi.org/project/pit-release-gate/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![PIT Hygiene](https://img.shields.io/badge/PIT%20Hygiene-pledged-2ea44f)](https://github.com/MaxWellApexLab/pit-hygiene)

Completeness-aware release control for staggered-arrival cross-sectional data.

**[Audit registry](https://github.com/MaxWellApexLab/pit-audit-registry) ·
[Pledge](https://github.com/MaxWellApexLab/pit-hygiene) ·
[Papers](#papers) ·
[Worked example: pandas DataFrame](docs/from-dataframe.md) ·
[Result schema](docs/results-schema.md)**

Rebuilt from as-filed SEC EDGAR filings, on observed filing dates, **7 of 14
standard fundamental signals are contaminated** by incomplete-cross-section
leakage — [measured, reproducibly, in the companion registry](https://github.com/MaxWellApexLab/pit-audit-registry/blob/main/methodology/2026-08_sec-edgar/report.md).
This tool measures each signal's susceptibility *before* release and withholds
only the signals that need it. The screen costs one `fit_trailing` call per
signal.

![Known-ground-truth demo: naive release is biased exactly when the leak is strong; the gate routes that signal to the deadline and its bias is exactly zero, while benign signals still release at ~36% completeness](https://raw.githubusercontent.com/MaxWellApexLab/pit-release-gate/master/docs/assets/demo_bias.png)

*The shipped fixed-seed demo, where the right answer is planted: naive early
release carries a systematic bias of −0.386 on the strong-leak signal; the gate
routes it to the deadline (bias exactly 0.0) while releasing the two benign
signals at 36–39% completeness. `tests/test_reproduces_paper.py` pins these
numbers; [`tools/make_readme_chart.py`](tools/make_readme_chart.py) redraws
this figure from the live demo.*

**You need this if:**

- you build **same-period cross-sectional signals** — industry-adjusted ratios,
  cross-sectional ranks, peer medians — on entities that report on their own
  schedule;
- you **rebuild panels from as-filed sources** (EDGAR `companyfacts`, raw
  filings) instead of using a vendor's curated release;
- you own a **feature-store pipeline** where an as-of join reads whatever has
  arrived by *t*;
- you want a **per-signal susceptibility number** published next to every
  released signal, the way a standard error is.

**Not for you if** you are hunting look-ahead bugs in a backtest engine, or your
data has no arrival times — that is a different failure mode
([scope statement](https://github.com/MaxWellApexLab/pit-audit-registry#what-gets-screened)).

**No telemetry — structurally, not merely by default:** no module in this
package imports a transport, and [a test enforces that over every module](tests/test_export.py).

## Install

```bash
pip install pit-release-gate
```

Requires Python 3.10+ (`numpy`, `pandas`, `scipy`).

## Quickstart (30 seconds)

```python
import numpy as np
from pit_release_gate import SusceptibilityGate, ReleaseController, make_group

rng = np.random.default_rng(0)

# 1. Fit the susceptibility gate on prior COMPLETED periods (honest estimation:
#    never on the period being gated — its cross-section is still incomplete).
train = [make_group(c_a=0.3, c_x=0.7, rng=rng) for _ in range(10)]
gate = SusceptibilityGate(threshold=0.10)
rho = gate.fit_trailing(train)

# 2. Gate a fresh, live period with the frozen estimate.
controller = ReleaseController(gate=gate)
live = make_group(c_a=0.3, c_x=0.7, rng=rng)
decision = controller.run_until_release(live, policy="gated")

print(f"rho_hat={rho:+.3f}  ->  {decision.action} "
      f"at completeness {decision.completeness:.0%} ({decision.policy})")
```

To gate your own data, build an `AsOfDataStore` from your design matrix, signal
values, and per-entity filing-arrival times, then call
`ReleaseController.decide(store, t)` at each evaluation time — it returns
`WITHHOLD`, `REWEIGHT_RELEASE`, or `RELEASE` plus the released values.

## Screen your own panel

One long table — one row per (entity, period) — and one call. The table can be
a pandas DataFrame, a polars DataFrame, a pyarrow Table, or a dict of arrays:
columns are read by duck typing, so the screen itself depends on no dataframe
library.

```python
from pit_release_gate import screen_dataframe

record = screen_dataframe(
    panel,                      # columns: period, arrival, value, size
    value=["accruals", "roa"],  # one signal or several
    trailing_k=5,               # fit on 5 prior COMPLETED periods, then freeze
)
for s in record["signals"]:
    print(s["name"], s["verdict"], s["periods_flagged"], "/", s["periods_screened"])
```

Or without writing any Python:

```bash
pit-release-gate --csv panel.csv --value accruals --value roa --export results.json
```

```text
screened panel.csv: 2 signal(s), trailing_k=5, threshold=0.1
  signal                    periods  flagged   mean rho  max |rho|  phi_req  verdict
  accruals                        7        7    -0.8721     0.8841    1.000  susceptible
  roa                             7        0    +0.0268     0.0564    0.377  benign
  totals: 1 benign, 1 susceptible, 14 signal-cycles screened
```

This is the same frozen protocol the [published audits](https://github.com/MaxWellApexLab/pit-audit-registry)
use: the estimate applied to a period is never fitted on that period, and the
first `trailing_k` periods are used for fitting only. The output is a
`pit-screen-results` v1.0 record — the file a *screened with* badge should
point at.

**One caveat, stated up front.** A verdict fires if any single screened period
crosses the threshold, so it inherits that period's sampling noise
(roughly `1 / sqrt(trailing_k x entities_per_period)`). On a small panel,
a reading just over the threshold may be noise. Establish your noise floor
first — screen a signal you expect to be unexposed, or shuffle arrival order
within periods and re-screen — before treating a marginal verdict as a finding.

## What this catches that your current tools don't

| guarantee | as-of join / bitemporal store | purged & embargoed CV | **pit-release-gate** |
|---|---|---|---|
| No value was read before it was available | ✅ | — | assumed as input (bring your arrival times) |
| Train and test don't overlap through time | — | ✅ | — |
| The **set of entities** present at *t* was not selected on the disturbance | ❌ | ❌ | **✅ measured per signal (ρ̂), gated per signal** |

A point-in-time-correct join over an incomplete cross-section is a correct join
over a biased sample. [Statement of need](#statement-of-need) has the full
argument; the
[as-of join methodology page](https://github.com/MaxWellApexLab/pit-audit-registry/blob/main/methodology/2026-08_feast-pit-join/report.md)
has the measured demonstration.

## API overview

Five public components, all importable from the top-level `pit_release_gate` package:

| component | what it does |
|---|---|
| [`AsOfDataStore`](src/pit_release_gate/store.py) | Holds one period's as-filed records for a cross-sectional group: design matrix, signal values, and a filing-arrival time per entity. |
| [`CompletenessMonitor`](src/pit_release_gate/monitor.py) | Reports the arrived fraction at an evaluation time, plus a composition-shift gauge for the arrived subset. |
| [`SusceptibilityGate`](src/pit_release_gate/gate.py) | Estimates ρ̂, the partial correlation between filing latency and the complete-cross-section residual given observables. `fit_trailing` enforces the honest-estimation contract: prior *completed* periods only. |
| [`PropensityReweighter`](src/pit_release_gate/reweight.py) | Inverse-filing-propensity weights. Included to make a negative result executable: reweighting on observables corrects composition, but cannot remove selection on the disturbance. |
| [`ReleaseController`](src/pit_release_gate/controller.py) | Maps \|ρ̂\| to a required completeness `φ_req = min(1, φ_min + κ·\|ρ̂\|)` and returns `WITHHOLD` / `REWEIGHT_RELEASE` / `RELEASE` at each evaluation time. |

`make_group`, `run_demo` and `demo` ([`simulate.py`](src/pit_release_gate/simulate.py))
generate and run the known-ground-truth worked example described below.

## The known-ground-truth demo

The package ships a self-contained worked example with a *planted* leakage strength,
so the right answer is known exactly and no licensed data is needed:

```bash
pit-release-gate            # or: python -m pit_release_gate
```

Real output, abridged to the signal the gate exists for:

```text
Strong-leak  (c_a=1.0, c_x=0.7)   rho_trailing=-0.868 (fitted ex ante)  (SUSCEPTIBLE -> wait)
  policy        comp% [95%CI]    flip% [95%CI]   biasB(signed) [CI]   route
  naive           35 ± 0.0      63.6 ± 2.8     -0.386 ±0.037   naive
  deadline       100 ± 0.0       0.0 ± 0.0     +0.000 ±0.000   deadline
  gated          100 ± 0.0       0.0 ± 0.0     +0.000 ±0.000   gated(phi_req=1.00)  <-- gated
```

It compares five release policies (`naive`, `threshold`, `reweight`, `deadline`,
`gated`) on four signal types. Headline behavior:

| signal | susceptibility ρ̂ | gated releases at | gated bias |
|---|---|---|---|
| Clean | ≈ +0.005 (benign) | **36%** completeness | ≈ 0 |
| Composition (selection on observables only) | ≈ −0.036 (benign) | **39%** completeness | ≈ 0 |
| Mild leak | ≈ −0.53 | 88% completeness | −0.099 (naive: −0.319) |
| Strong leak | ≈ −0.87 | 100% (deadline) | **exactly 0.0** (naive: −0.386) |

A sensitivity sweep of the policy slope κ shows the timeliness–bias dial:
κ = 0.5 → release at 59% completeness (bias −0.229); κ = 1.0 → 83% (−0.118);
κ = 2.0 → 100% (bias exactly 0). The demo is deterministic (fixed seed), and
`tests/test_reproduces_paper.py` asserts these numbers.

## Statement of need

When the entities of a cross-section report on staggered dates — companies filing
financial statements are the canonical case — any same-period cross-sectional signal
computed before the last filer arrives is estimated from an incomplete, and possibly
*selectively* incomplete, cross-section. If filing timing depends on the very
disturbance the signal measures, releasing early produces a systematic bias
(incomplete-cross-section leakage), while a blanket wait-for-the-deadline rule removes
the bias at a timeliness cost paid by every signal, biased or not.

Point-in-time discipline in ML pipelines currently rests on tooling that answers one
question: *was this value readable at time t?* Feature-store as-of joins, bitemporal
and vintage-aware storage, and purged or embargoed cross-validation all enforce
read-time correctness, and they do it well.

None of them answers a second question: given that every value read was legitimately
readable, was the *set* of entities that had reported by t a selected sample? An
as-of join over an incomplete cross-section is a correct join over a biased sample.
The two failures need different remedies — the first is fixed by timestamp hygiene,
the second only by waiting or by an explicit correction. Researchers building
cross-sectional signals on staggered-arrival panels have had no routine, per-signal
screen for the second. `pit-release-gate` is that screen, plus the release controller
that acts on it: one `fit_trailing` call per signal, so reporting a susceptibility
estimate alongside a released signal costs about as much as reporting a standard
error.

## Export your screen result

```bash
pit-release-gate --export results.json
```

writes a `pit-screen-results` v1.0 record: per screened signal, how many periods
were screened, how many the measure flagged, mean and max ρ̂, the required
completeness that was assigned, and the verdict (`benign` / `susceptible`), plus
the five settings that produced those verdicts. It is the file a *screened with*
badge should point at. The format is a standalone versioned interchange spec —
[`docs/results-schema.md`](docs/results-schema.md) — that any other tool is free
to emit or consume.

**`--export` is fully offline**: it writes a local file and makes no network
call. Records carry summary statistics only — never input rows, file paths,
usernames, hostnames, or any environment detail beyond the tool version — and no
clock is read while building one, so the same screen always exports byte-identical
bytes.

Where the record goes afterwards is entirely your business — commit it next to
your badge, publish it, or keep it. This tool does not send it anywhere. There is
no background thread, no `atexit` hook, no anonymous usage counter, and nothing
to opt out of.

Programmatic use, for screens on your own data:

```python
from pit_release_gate import build_results, screen_config, summarize_signal, validate_results

sig = summarize_signal("accruals", rhos=[...], phi_reqs=[...], rho_threshold=0.10)
record = build_results([sig], screen_config(0.10, 0.35, 1.0, trailing_k=8, min_entities=6))
assert validate_results(record) == []
```

## External references

- Listed in [awesome-quant](https://github.com/wilsonfreitas/awesome-quant)
  (Factor Analysis section).
- The worked pandas example in the docs was contributed by an external
  contributor ([#9](https://github.com/MaxWellApexLab/pit-release-gate/pull/9)).
- The output record format is a separately versioned public spec
  ([`docs/results-schema.md`](docs/results-schema.md)) so other tools can emit
  or consume screen results without importing this package.
- Seen in the wild: the gate and controller were read line by line in
  QMX's systematic survey of backtest-safety tooling. Their quality
  verdict: "clean, small, and does real statistics ... worth reading as a
  design reference". The selection-on-arrival distinction now sits in
  their design ledger as [row 78](https://github.com/MubarakHimself/QMX/blob/af6628888fb8901bf9e877cbd07e438dbb0f39b4/workroom/reference/03-wave2-supplement.md).
  No formal citation, and none needed; traveling is the point.

## Papers

The method and its evaluation are developed in three public preprints
(paper 3 is under review at an IEEE conference; papers are self-archived
with DOIs pending journal publication):

1. **Correct-by-Construction Factor Computation: A Verifiably Point-in-Time Engine
   for Tradeable Signals** — [doi:10.6084/m9.figshare.32952482](https://doi.org/10.6084/m9.figshare.32952482)
2. **Measuring Incomplete-Cross-Section Leakage: A Matched Placebo, a Susceptibility
   Screen, and Evidence from Taiwan and US As-Filed Data** — [doi:10.6084/m9.figshare.33061955](https://doi.org/10.6084/m9.figshare.33061955)
3. **Susceptibility-Graded Release Control: Preventing Incomplete-Cross-Section
   Leakage in Financial Machine-Learning Pipelines without a Blanket Timeliness
   Penalty** — [doi:10.6084/m9.figshare.33158615](https://doi.org/10.6084/m9.figshare.33158615)

This package is the reference implementation of paper 3's release controller; its
demo reproduces the paper's controlled experiment.

## Badge

If you have run the susceptibility screen on your own data — whatever the result — you
are welcome to say so:

```markdown
[![screened with pit-release-gate](https://img.shields.io/badge/screened%20with-pit--release--gate-blue)](https://github.com/MaxWellApexLab/pit-release-gate)
```

The badge reads **screened with**, not *passed* — it states that the screen was run, the
same way a formatter badge states that the formatter was run. A benign result and a
susceptible result are equally worth badging; the second one arguably more, because it
means the screen found something and your pipeline now waits for it.

**Make it point at something.** A badge is worth reading only if there is evidence behind
it. Commit your screen output — `pit-release-gate --export results.json` produces exactly
that file: which signals came out benign, which came out susceptible, and the required
completeness each was assigned — and link the badge at it rather
than at this repo. A worked example is the OSAP screen in the
[PIT audit registry](https://github.com/MaxWellApexLab/pit-audit-registry/blob/main/audits/2026-08_osap/report.md).

**Related:** the [PIT Hygiene pledge](https://github.com/MaxWellApexLab/pit-hygiene) is a
broader, tool-neutral statement about how a staggered-arrival pipeline is built; this badge
is the narrower statement that this particular screen was run.

## Cite this

See [`CITATION.cff`](CITATION.cff). If you use this software, please cite paper 3:

```bibtex
@article{wu2026releasecontrol,
  title  = {Susceptibility-Graded Release Control: Preventing Incomplete-Cross-Section
            Leakage in Financial Machine-Learning Pipelines without a Blanket
            Timeliness Penalty},
  author = {Wu, Kuan-Ta and Wu, Kuan-I},
  year   = {2026},
  doi    = {10.6084/m9.figshare.33158615}
}
```

## Community guidelines

- **Report a bug or request a feature:** open an issue at
  [github.com/MaxWellApexLab/pit-release-gate/issues](https://github.com/MaxWellApexLab/pit-release-gate/issues).
- **Contribute:** see [CONTRIBUTING.md](CONTRIBUTING.md) for the development setup,
  test requirements, and pull-request process.
- **Get help:** open an issue with a minimal reproducing example, or email
  maxwellapexlab@proton.me.

## License

MIT — see [LICENSE](LICENSE).

An additional patent grant covers this implementation — see [PATENTS](PATENTS).
Max Well Apex LLC holds pending U.S. patent applications on techniques this
software implements; the PATENTS file grants a royalty-free license to the
claims necessarily practiced by this implementation, with the customary
termination-on-litigation clause. In short: using, modifying, and shipping
this code is safe; the grant follows the Go project's PATENTS model.

---

Max Well Apex LLC — maxwellapexlab@proton.me
