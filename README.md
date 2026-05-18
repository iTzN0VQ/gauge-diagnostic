# gauge-diagnostic

A single-page HVAC refrigerant gauge diagnostic. Tech enters suction/discharge pressures and suction/liquid line temperatures, gets:

- **Evap saturation temperature** (interpolated from a PT chart)
- **Condenser saturation temperature**
- **Superheat** = suction line temp − evap saturation
- **Subcooling** = condenser saturation − liquid line temp
- **Compression ratio**
- **A diagnostic verdict** with likely causes and field-tested fixes (low charge, overcharge, restriction, compressor weakness, floodback risk, etc.)

> Built by [Rizvan Fazli](https://github.com/iTzN0VQ) — Mitsubishi-certified lead HVAC tech. Part of [NovaCTRL.ai](https://novactrl.ai).

---

## Live tool

Use it without installing anything: **[novactrl.ai/tools/gauge.html](https://novactrl.ai/tools/gauge.html)**.

Or clone this repo and open `index.html` in any browser. No build step, no server, no API keys — everything runs client-side.

---

## What's in this repo

| File              | What it is                                                                                  |
|-------------------|---------------------------------------------------------------------------------------------|
| `index.html`      | The full standalone tool. Open in any browser. 16 refrigerants, ~525 PT data points, full diagnostic engine — all inlined. |
| `pt-charts.json`  | Just the PT data, as JSON. Embed this in your own tool. 16 refrigerants × 30-40 data points. |
| `LICENSE`         | MIT.                                                                                        |

---

## Supported refrigerants

R-410A, R-22, R-32, R-454B, R-407C, R-134a, R-404A, R-290 (propane), R-438A, R-507A, R-448A, R-449A, R-513A, R-1234yf, R-1234ze(E), R-123.

---

## Diagnostic rule set

The engine flags these conditions based on superheat/subcool/suction/discharge combinations:

| Condition                                     | Trigger                                              | Severity   |
|-----------------------------------------------|------------------------------------------------------|------------|
| Low refrigerant charge                        | Superheat > 15°F **and** subcool < 8°F               | Critical   |
| Overcharged system                            | Superheat < 5°F **and** subcool > 18°F               | Warning    |
| Restriction in refrigerant circuit            | Superheat > 15°F **and** subcool > 15°F              | Critical   |
| Severe undercharge / compressor issue         | Suction < 50 psi **and** superheat > 20°F            | Critical   |
| Very high discharge pressure (condenser fault)| Discharge > 400 psi                                  | Critical   |
| Low discharge pressure (weak compressor)      | Discharge < 150 psi and suction > 0                  | Warning    |
| Properly charged                              | Superheat 5-15°F **and** subcool 8-15°F              | Info       |
| Liquid floodback risk                         | Superheat 0-3°F                                      | Critical   |

Each verdict includes specific likely causes and step-by-step field-tested fixes.

---

## Embedding the engine in your own tool

The diagnostic logic is in plain JavaScript inside `index.html` — copy the `PT_CHARTS`, `getSatTemp()`, and `diagnoseFromGauges()` functions and drop them into any project. No dependencies.

```js
// Minimal example
const evapSat = getSatTemp('R-410A', 125);     // → suction saturation temp °F
const condSat = getSatTemp('R-410A', 385);     // → discharge saturation temp °F
const superheat = 52 - evapSat;
const subcool   = condSat - 102;
const hints     = diagnoseFromGauges(superheat, subcool, 125, 385);
// hints[0] = { condition, causes[], fixes[], severity }
```

---

## Caveats

- The PT charts are abridged for compact size. Real-world charging always cross-references the **manufacturer's** charging chart for the specific equipment.
- The diagnostic ruleset uses cooling-mode thresholds. For heat pumps in heating mode, interpret accordingly — measure at the appropriate access ports.
- This tool **does not replace** a senior tech's judgment. It's a sanity check and a teaching tool. The fixes section is a starting point, not a checklist.

---

## License

MIT — see [LICENSE](./LICENSE). Use it, fork it, embed it, improve it.

---

## Author

Rizvan Fazli · rizvanfazli8@gmail.com · [NovaCTRL.ai](https://novactrl.ai)
