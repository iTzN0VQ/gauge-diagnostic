# Gauge Diagnostic — React Hand-off Pack

This is the full React + TypeScript + Tailwind port of the gauge diagnostic. It expands on the standalone `index.html` in the parent folder with:

- **17 refrigerants** with PT charts and linear interpolation
- **NCI performance analysis** — compression ratio, condenser approach, condensing-over-ambient, evaporator air ΔT
- **Animated SVG pressure rings** (low / normal / critical color zones per refrigerant)
- **Quick-diagnosis verdict card** with severity badging
- **PDF report generation** (jsPDF + jspdf-autotable, branded)
- **Save-to-job** hook (optional)
- **AI streaming analysis** hook (optional)
- **framer-motion** animations throughout

If you just want the in-browser tool, use the parent `index.html` — no React, no build, no dependencies. The files below are for integrating into an existing React project.

## What this feature does

A single tab/component that lets the tech pick a refrigerant (17 supported) and enter suction PSI, liquid (discharge) PSI, suction line temp °F, and liquid line temp °F. Optionally accepts outdoor ambient, return-air dry bulb, and supply-air dry bulb for advanced NCI performance analysis.

Computes:
- Evap & condenser saturation temps (via 17 built-in P/T charts with linear interp)
- Superheat = suction line temp − evap sat
- Subcooling = condenser sat − liquid line temp
- Compression ratio = (P_disch + 14.7) / (P_suct + 14.7)
- Condenser approach & condensing-over-ambient
- Evaporator ΔT (return DB − supply DB)

Renders animated SVG pressure-rings, color-graded result cards, a one-glance "quick diagnosis" verdict, and an expandable list of rule-based diagnostic hints (Low Charge / Overcharge / Restriction / High Head / Liquid Floodback / etc.) each with probable causes + things to check. Generates a branded PDF report via jspdf + jspdf-autotable.

No external API calls are required for any of the math. The AI analysis button is optional and stubbed.

## Dependencies

```bash
npm install react react-dom framer-motion lucide-react jspdf jspdf-autotable
npm install -D typescript tailwindcss postcss autoprefixer
```

If the target uses shadcn/ui, the gauge code uses no shadcn components — just raw Tailwind classes referencing CSS variables (see Tailwind section below). It will work with any shadcn theme.

## File layout to create

```
src/
  components/
    GaugeTab.tsx              ← main UI
    previews/
      GaugePreview.tsx        ← animated marketing/preview tile (optional)
  lib/
    hvac-data.ts              ← PT charts + diagnostic rules
    nova-diagnostics.ts       ← compression ratio + condenser/evap analyzers
    gauge-report.ts           ← PDF generator
  hooks/
    use-ai-stream.ts          ← stub or your AI integration
  contexts/
    NovaContext.tsx           ← stub or your gauge-context store
    JobContext.tsx            ← stub or your save-to-job
```

## Math reference

| Quantity | Formula |
|---|---|
| Evap saturation temp | `getSatTemp(refrigerant, suction_psi)` via linear interp in PT chart |
| Cond saturation temp | `getSatTemp(refrigerant, liquid_psi)` |
| Superheat (°F) | `suction_line_temp − evap_sat` |
| Subcooling (°F) | `cond_sat − liquid_line_temp` |
| Compression ratio | `(disch_psi + 14.7) / (suct_psi + 14.7)` — normal 2.0–3.5, danger >4 |
| Condenser approach | `liquid_line_temp − outdoor_ambient` — normal 5–25 °F |
| Condensing over ambient (COA) | `sat_cond − outdoor_ambient` — normal 15–35 °F |
| Air ΔT | `return_DB − supply_DB` — normal 14–22 °F |
| Evaporator TD | `return_WB − sat_evap` — normal 2–6 °F |

## Diagnostic rules (`diagnoseFromGauges`)

| Condition | Verdict |
|---|---|
| superheat > 15 AND subcooling < 8 | Low refrigerant charge (critical) |
| superheat < 5 AND subcooling > 18 | Overcharge (warning) |
| superheat > 15 AND subcooling > 15 | Restriction in circuit (critical) |
| suction < 50 PSI AND superheat > 20 | Severe undercharge / bad compressor (critical) |
| discharge > 400 PSI | High head (critical) |
| discharge < 150 PSI AND suction > 0 | Weak compressor / low charge (warning) |
| 5 ≤ superheat ≤ 15 AND 8 ≤ subcooling ≤ 15 | Normal (info) |
| 0 ≤ superheat < 3 | Liquid floodback risk (critical) |

## Supported refrigerants

R-410A, R-22, R-32, R-454B, R-407C, R-134a, R-404A, R-290, R-438A, R-507A, R-448A, R-449A, R-513A, R-1234yf, R-1234ze(E), R-123.

## Source for `hvac-data.ts` (PT_CHARTS + getSatTemp + diagnoseFromGauges)

Already in this repo at [`pt-charts.json`](../pt-charts.json) and inlined inside [`index.html`](../index.html). For the TypeScript module wrapper, copy `pt-charts.json` and wrap as:

```ts
import PT from './pt-charts.json';
export interface PTEntry { psi: number; tempF: number; }
export const PT_CHARTS: Record<string, PTEntry[]> = PT;
export function getSatTemp(refrigerant: string, psi: number): number {
  const chart = PT_CHARTS[refrigerant];
  if (!chart?.length) return 0;
  if (psi <= chart[0].psi) return chart[0].tempF;
  if (psi >= chart.at(-1)!.psi) return chart.at(-1)!.tempF;
  for (let i = 0; i < chart.length - 1; i++) {
    if (psi >= chart[i].psi && psi <= chart[i + 1].psi) {
      const r = (psi - chart[i].psi) / (chart[i + 1].psi - chart[i].psi);
      return Math.round((chart[i].tempF + r * (chart[i + 1].tempF - chart[i].tempF)) * 10) / 10;
    }
  }
  return 0;
}
```

## NCI helpers (`nova-diagnostics.ts`)

```ts
export type PerformanceStatus = 'normal' | 'warning' | 'danger';
const ATM_PRESSURE_PSIA = 14.7;
const round1 = (n: number) => Math.round(n * 10) / 10;

export function calculateCompressionRatio(dischargePsi: number, suctionPsi: number) {
  const absD = dischargePsi + ATM_PRESSURE_PSIA;
  const absS = suctionPsi + ATM_PRESSURE_PSIA;
  if (absS <= 0) return { ratio: 0, status: 'danger' as PerformanceStatus, label: 'Invalid', detail: 'Suction pressure below absolute zero' };
  const ratio = round1(absD / absS);
  let status: PerformanceStatus = 'normal', label = 'Normal', detail = `Compression ratio ${ratio}:1 within normal range (2.0–3.5).`;
  if (ratio > 4.0) { status = 'danger'; label = 'High'; detail = `Compression ratio ${ratio}:1 exceeds 4.0 — compressor stress.`; }
  else if (ratio > 3.5) { status = 'warning'; label = 'Elevated'; detail = `Compression ratio ${ratio}:1 elevated. Monitor closely.`; }
  else if (ratio < 1.5) { status = 'warning'; label = 'Low'; detail = `Compression ratio ${ratio}:1 low. Possible reversing valve leak.`; }
  return { ratio, status, label, detail };
}

export function analyzeCondenserPerformance(liquidLineTemp: number, outdoorAmbientTemp: number, satCondenserTemp: number) {
  const approach = round1(liquidLineTemp - outdoorAmbientTemp);
  const coa = round1(satCondenserTemp - outdoorAmbientTemp);
  let approachStatus: PerformanceStatus = 'normal';
  if (approach > 25) approachStatus = 'warning';
  if (approach > 35) approachStatus = 'danger';
  if (approach < 5) approachStatus = 'warning';
  let coaStatus: PerformanceStatus = 'normal';
  if (coa > 35) coaStatus = 'warning';
  if (coa > 45) coaStatus = 'danger';
  if (coa < 15) coaStatus = 'warning';
  return { condenserApproach: approach, approachStatus, condensingOverAmbient: coa, coaStatus, detail: `Approach ${approach}°F, COA ${coa}°F` };
}

export function analyzeEvaporatorPerformance(returnDBTemp: number, supplyDBTemp: number, returnWBTemp?: number, satEvaporatorTemp?: number) {
  const deltaT = round1(returnDBTemp - supplyDBTemp);
  let deltaStatus: PerformanceStatus = 'normal';
  let detail = `Air ΔT ${deltaT}°F within normal range (14–22°F).`;
  if (deltaT > 22) { deltaStatus = 'warning'; detail = `Air ΔT ${deltaT}°F high (>22°F) — possible low airflow.`; }
  if (deltaT > 28) { deltaStatus = 'danger'; detail = `Air ΔT ${deltaT}°F dangerously high — check duct or frozen coil.`; }
  if (deltaT < 14) { deltaStatus = 'warning'; detail = `Air ΔT ${deltaT}°F low (<14°F) — high airflow, low charge, or TXV issue.`; }
  let evapTD: number | null = null, evapTDStatus: PerformanceStatus | null = null;
  if (returnWBTemp != null && satEvaporatorTemp != null) {
    evapTD = round1(returnWBTemp - satEvaporatorTemp);
    evapTDStatus = 'normal';
    if (evapTD > 6) evapTDStatus = 'warning';
    if (evapTD > 10) evapTDStatus = 'danger';
    if (evapTD < 2) evapTDStatus = 'warning';
  }
  return { airDeltaT: deltaT, deltaStatus, evapTD, evapTDStatus, detail };
}
```

## Tailwind theme variables

The component uses these semantic Tailwind classes that resolve to CSS variables. Add this block to `src/index.css` (or your global stylesheet):

```css
@tailwind base;
@tailwind components;
@tailwind utilities;

@layer base {
  :root {
    color-scheme: dark;
    --background: 220 14% 7%;
    --foreground: 30 8% 95%;
    --card: 220 12% 10%;
    --card-foreground: 30 8% 95%;
    --primary: 22 92% 52%;          /* orange accent — change to your brand */
    --primary-foreground: 0 0% 100%;
    --secondary: 220 10% 14%;
    --secondary-foreground: 30 8% 80%;
    --muted: 220 10% 12%;
    --muted-foreground: 220 6% 52%;
    --border: 220 8% 18%;
    --radius: 0.5rem;
  }
}
```

And in `tailwind.config.ts` extend `theme.extend.colors` with `hsl(var(--...) / <alpha-value>)` mappings.

## License

MIT — same as the parent repo. The full `GaugeTab.tsx`, `GaugePreview.tsx`, `hvac-data.ts`, `nova-diagnostics.ts`, and `gauge-report.ts` source files are reproduced inline in this hand-off pack (~1,500 lines). Drop them into your project under `src/components/` and `src/lib/`, install the deps, paste the Tailwind theme block, and the Pressure Analyzer is live.

See the parent [`README.md`](../README.md) for the standalone (no-React) version that ships as a single HTML file.
