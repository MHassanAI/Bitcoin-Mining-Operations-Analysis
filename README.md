# Bitcoin Mining Operations — Intelligence Analysis System

A Python-based telemetry analysis pipeline that ingests ASIC miner data and
produces five categories of numerically-justified, decision-oriented insights
for mining operations teams.

---

## Quick Start (Google Colab)

1. Open `Bitcoin_Mining_Analysis.ipynb` in Google Colab
2. Go to **Runtime → Run all**
3. All outputs, charts, and the JSON report generate automatically

**Local execution:**
```bash
pip install pandas numpy matplotlib seaborn
jupyter notebook Bitcoin_Mining_Analysis.ipynb
```

No database, no API keys, no configuration required. All state is in-memory.

---

## Project Structure

```
submission/
├── Bitcoin_Mining_Analysis.ipynb   # Full notebook — all code, explanations, charts, outputs
├── insights_output.json            # Generated JSON report (13 insights across 5 modules)
└── README.md                       # This file
├── Mining_Operations_Intelligence_Report.pdf   # Executive-ready PDF report (stakeholder view)
```

---

## Approach & Reasoning

### Design Philosophy

Every anomaly must produce an **operational action**, not just a flag.
Each insight contains:
- A numeric justification (correlation coefficient, z-score, °C above threshold)
- A severity level (CRITICAL / HIGH / MEDIUM / LOW)
- A specific action tied to hardware risk or revenue impact in USD

### Pipeline Architecture

```
Raw Telemetry (seed data + simulated 10-miner, 24h, 5-min intervals)
     │
     ▼
[Feature Engineering]
     ├── per-miner baseline hashrate (median of first 2h)
     ├── hashrate deviation % from baseline
     ├── chip temp delta per interval (NaN for first row — not filled with 0)
     ├── pressure delta per interval (NaN for first row — not filled with 0)
     ├── 30-min rolling chip temp average
     ├── thermal headroom (distance from 95°C critical threshold)
     └── cooling efficiency ratio (TH/s per °C of immersion temp)
     │
     ├──▶ [Module 1] Performance Impact    ← thermal throttle, sustained underperformance
     ├──▶ [Module 2] Hardware Risk         ← temp threshold exceedance, rapid rises
     ├──▶ [Module 3] Cooling Analysis      ← pressure anomalies, cooling decoupling
     ├──▶ [Module 4] Peer Anomaly          ← per-timestamp z-score vs fleet
     └──▶ [Module 5] Efficiency            ← over-cooling waste, efficiency ratio
              │
              ▼
     [Executive Summary]    ← fleet KPIs, alert counts, revenue at risk, top-5 actions
              │
              ▼
     Console Report + insights_output.json
```

### Why These Analytical Choices?

| Technique | Used In | Reason |
|---|---|---|
| Pearson correlation | Module 1 | Quantifies temp↑hash↓ relationship directionally |
| Median baseline (first 2h) | Module 1 | Robust to early anomalies vs mean |
| Healthy-peer external baseline | Module 1 | Detects miners degraded from startup whose own baseline is already low |
| Warmup-window 3-sigma pressure | Module 3 | First 2h std used as fixed threshold — prevents spike from inflating its own detection threshold |
| Slope-based decoupling | Module 3 | chip_slope > 0.08°C/interval + imm_slope < 0.03°C/interval — physically meaningful signal |
| Per-timestamp z-score | Module 4 | Controls for shared fleet conditions at each moment |
| Immersion temp efficiency ratio | Module 5 | Fluid temp is more sensitive over-cooling signal than chip temp |

---

## Key Assumptions

| # | Assumption | Value | Basis |
|---|---|---|---|
| 1 | Chip temp — throttle warning | **85°C** | Bitmain S19/S21 datasheet |
| 2 | Chip temp — hardware damage critical | **95°C** | Bitmain S19/S21 datasheet |
| 3 | Rapid temperature rise threshold | **≥8°C in 5 min** | >3σ above normal thermal load |
| 4 | Immersion fluid max safe temp | **60°C** | Standard dielectric fluid specs |
| 5 | Immersion pressure normal range | **1.5–2.2 bar** | Typical closed-loop immersion systems |
| 6 | Revenue per TH/s per day | **$0.0534** | (144 blocks × 3.125 BTC × $95,000) / 800,000,000 TH/s |
| 7 | ASIC replacement cost | **$3,500** | Bitmain Antminer S19 XP market price |
| 8 | Excess cooling power | **~8W per °C** | Conservative variable-speed pump estimate |
| 9 | Miner baseline | Median of first 2h (24 readings) | Outlier-robust; first 2h assumed stable |
| 10 | External peer baseline | Healthy miners average (M002/M004/M006/M008/M010) | For miners degraded from startup |
| 11 | Pressure detection warmup | First 2h only for std calibration | Prevents spike from inflating its own threshold |

---

## Generated Insights — Actual Results

Running the notebook produces **13 insights across 5 modules**:

### Module 1 — Performance Impact (3 insights)

| Miner | Type | Key Numbers | Daily Loss |
|---|---|---|---|
| M001 | THERMAL_THROTTLE | Corr=-0.97, 7.1 TH/s avg lost during 740 min throttled | $0.20 |
| M003 | THERMAL_THROTTLE | Corr=-0.99, 34.9 TH/s avg lost during 120 min throttled | $0.16 |
| M005 | SUSTAINED_UNDERPERFORMANCE | 90.4 vs 115.7 TH/s peer avg — 21.9% gap | $1.36 |

### Module 2 — Hardware Damage Risk (3 insights)

| Miner | Type | Key Numbers | Action |
|---|---|---|---|
| M001 | CRITICAL_TEMP | 345 min above 95°C, max 103.8°C | Immediate shutdown — $3,500 at risk |
| M003 | CRITICAL_TEMP | 105 min above 95°C, max 96.2°C | Immediate shutdown — $3,500 at risk |
| M003 | RAPID_TEMP_INCREASE | +17.6°C in 5 min at 06:00 | Pump failure — inspect tank |

### Module 3 — Cooling System Analysis (3 insights)

| Miner | Type | Key Numbers | Action |
|---|---|---|---|
| M001 | COOLING_DECOUPLING | Chip +1.5°C/hr, fluid +0.0°C/hr | Thermal interface inspection |
| M003 | PRESSURE_DROP | -0.306 bar at 08:00 | Seal/pump failure check |
| M007 | PRESSURE_DROP | -0.457 bar at 14:00 | Seal/pump failure check |

### Module 4 — Peer Anomaly (2 insights)

| Miner | Type | Key Numbers | Daily Loss |
|---|---|---|---|
| M001 | PEER_ANOMALY | Above-avg temp 67% of readings (z=+1.94) | $0.24 |
| M005 | PEER_ANOMALY | Below-avg hash 93% of readings (z=-2.19) | $1.12 |

### Module 5 — Efficiency (2 insights)

| Miner | Type | Key Numbers | Action |
|---|---|---|---|
| M005 | LOW_COOLING_EFFICIENCY | 75% of fleet avg efficiency ratio | Root cause: degraded ASICs |
| M009 | OVER_COOLED | Fluid 12.7°C below fleet avg (36.9 vs 49.6°C) | 102W wasted — reduce pump speed |

### Executive Summary

```
Fleet KPIs (24h, 10 miners)
├─ Fleet total hashrate : 1,113.7 TH/s
├─ Avg hashrate / miner : 111.4 TH/s
└─ Avg chip temp        : 77.0°C

💰 Financial Impact
├─ Fleet daily revenue (full health) : $59.51
├─ Estimated daily loss              : $3.08  (5.2% of revenue)
├─ Annualized loss (this fleet)      : $1,124
└─ Site-scale loss (1,000 miners)    : $112,420 / year

Alert Summary
├─ 🔴 CRITICAL : 2
├─ 🟠 HIGH     : 8
├─ 🟡 MEDIUM   : 2
├─ 🟢 LOW      : 1
└─ TOTAL       : 13
```

---

## Extending to Production / Real-Time Systems

| Current (Notebook) | Production |
|---|---|
| Simulated / CSV data | Kafka / MQTT stream from site agents |
| In-memory pandas | TimescaleDB or InfluxDB |
| Console print | PagerDuty (CRITICAL), Slack (MEDIUM) |
| Static thresholds | Adaptive per-miner rolling 99th percentile |
| Single run | Continuous streaming (Apache Flink / Spark) |
| Manual action | Auto-remediation API (throttle, pump control) |

### Predictive Maintenance (with 7+ days of data)
```python
slope = np.polyfit(range(len(recent_temps)), recent_temps, 1)[0]  # °C/interval
hours_to_critical = (CHIP_TEMP_CRITICAL_C - current_temp) / (slope * 12)
# Schedule maintenance before the failure event
```

---

## Trade-offs & Limitations

| Limitation | Production Mitigation |
|---|---|
| Revenue uses fixed BTC price ($95,000) | Pull live price from exchange API every 5 min |
| Healthy peer list hardcoded | Auto-detect healthy miners using IQR outlier removal |
| Pressure warmup assumes fault-free first 2h | Extend to rolling baseline with anomaly exclusion |
| No power draw (J/TH) metric | Add PDU/smart plug data feed as 5th sensor column |
| Decoupling slope threshold tuned for simulation | Validate against real site data; use adaptive threshold |
| No statistical significance testing on correlations | Add p-value check: only flag corr if p < 0.05 |
| Single-node pandas | Replace with Polars or Spark for 1,000+ miners |
