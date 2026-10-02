# Mouse, rat, and custom filament ladders

The algorithm is not intrinsically mouse-only. The original implementation uses
eight rows from the bundled workbook and a fixed delta of 0.441428571. Adding
stronger filaments requires both a matching lookup table and a matching log
interval; extending the spreadsheet alone is insufficient.

## Rat reference and provenance

The [NIH/NINDS rat hind-paw protocol](https://pspp.ninds.nih.gov/TestDescription/TestPWT)
lists the eight handle codes below. The `rat` preset uses these codes as `Log`.
Its `Force (g)` values are **derived** as `10**Log / 10000`, not measurements of
any laboratory's kit and not rounded manufacturer force markings. Consequently,
`Log_new` and `Log` agree for this preset. The original
[Chaplan et al. (1994) study](https://pubmed.ncbi.nlm.nih.gov/7990513/)
reports a 0.41–15.1 g range in rats. A species label alone does not identify the
ladder used in a particular experiment.

| `last_filament` ID | Handle code (`Log`) | Derived force (g, rounded here) |
| --- | --- | --- |
| 1 | 3.61 | 0.4074 |
| 2 | 3.84 | 0.6918 |
| 3 | 4.08 | 1.2023 |
| 4 | 4.31 | 2.0417 |
| 5 | 4.56 | 3.6308 |
| 6 | 4.74 | 5.4954 |
| 7 | 4.93 | 8.5114 |
| 8 | 5.18 | 15.1356 |

The preset IDs are local to this eight-filament ladder. They are **not** the
positions in a 20-piece kit, the handle codes, or gram values. Remap recorded IDs
before using the preset, or use a custom CSV preserving your recorded IDs.

The mean interval is `(5.18 - 3.61) / 7 = 0.224285714...`. The calculation remains
`10**(Xf + k * delta) / 10000`; the existing workbook supplies the same k table.
Delta is calculated using the selected log column across the complete ladder,
not just the last filament or the filaments appearing in one animal's data.

## Why not use every filament in a 20-piece kit?

[Stoelting sells a 20-probe kit](https://stoeltingco.com/Neuroscience/Touch-Test-Sensory-Probes~9834)
and describes using subsets. A product inventory is not an experimental ladder.
The proposed 20-row screenshot includes forces up to 300 g, uneven log steps,
rounded low-force labels (some shown as zero), and a blank force column. It is
not sufficient calibration data. Neither the NIH protocol nor the Chaplan range
above establishes 300 g as a rat testing default.

Handle codes and rounded force markings are distinct inputs. For example, the
screenshot pairs code 5.46 with a 26 g marking, while `10**5.46 / 10000` is about
28.84 g. Do not infer a measured force from a code or use rounded zeros. Use the
actual kit calibration and the ordered subset used by the experiment.

## Using the app

In Step 1 select **Rat — NIH handle codes** or **Custom calibrated ladder**.
The status shows the number of filaments and delta. Keep the original
`VF_Calculator_Up-down.xlsx` selected: it still supplies k values. Profile, custom
CSV path, and log choice are saved in session settings. After loading a session,
reload measurement/metadata files and recompute as usual. Adjust the plot's y-axis
maximum in Step 3 when the existing 10 g maximum would hide rat results.

CLI examples:

```bash
python run.py --compute --data rat_data.csv --filament-set rat --output results/
python run.py --compute --data rat_data.csv --filament-set custom \
  --custom-filaments my_calibrated_ladder.csv --output results/
```

Exported measurement rows include `vf_filament_set`, `vf_log_column`, and
`vf_delta` alongside `threshold_50`. Retain the selected CSV with the experiment
for the full calibration and ID mapping.

## Custom CSV format

Use one row per filament in ascending force order. Include **every step in the
experimental ladder**, including steps not reached in this dataset. There is no
eight-row limit. IDs must be unique positive integers; they need not be
consecutive. Required columns are `Filament_number` and `Force (g)`. `Log` is
optional; if omitted, both log choices use `log10(force_g * 10000)`.

A synthetic format example (not a recommended experimental ladder):

```csv
Filament_number,Force (g)
10,1
20,2
30,4
40,8
50,16
```

If you supply `Log`, it represents your reference/handle codes. `Log_new` always
uses your force values at full precision. Both log choices calculate their own
mean spacing. Missing, nonfinite, zero/negative forces, duplicate IDs, fractional
IDs, and non-increasing forces/logs are rejected. The k table is never replaced
by the CSV.

## Method limits and compatibility

This change retains the **mean-spacing Dixon approximation**. Commercial ladders
are not perfectly log-spaced; computing their mean interval does not make the
estimator exact. [Christensen et al. (2020)](https://pubmed.ncbi.nlm.nih.gov/31889375/)
identify filament spacing as a source of error and present a different algorithm
using the exact intervals. That algorithm is not implemented here. Strongly
uneven custom ladders require methodological review before interpreting results.

Unsupported response patterns and unknown final IDs continue to produce NaN.
This PR does not add a new stopping rule, reconstruct full stimulus histories,
or implement protocol-specific floor/ceiling censoring. It does not clamp numeric
estimates to the filament range. Apply and report the laboratory's predefined
boundary handling separately.

The default `legacy` profile retains the shipped workbook, k values, three-decimal
`Log_new` rounding, and fixed delta, so existing mouse results remain unchanged.
The workbook also contains a force/log discrepancy for filament 5 (0.178 g versus
stored Log 3.61), beyond the filament-4 discrepancy previously documented. This
PR does not guess replacement calibration values or change legacy results.

Sources checked 2026-10-02. The rat preset is a documented reference example;
confirm the actual ladder and calibration before analyzing a laboratory dataset.
