# InfluxDB Demo Data — Rack Power (2026)

Sample time-series data for datacenter rack power monitoring, intended for
InfluxDB import and Grafana dashboard testing.

## File

`demo-power-data-racks_2026.csv`

## Coverage

| Property | Value |
|---|---|
| Period | 2026-01-01 00:00 UTC → 2026-02-01 00:00 UTC |
| Interval | 60 seconds |
| Total rows | 535,680 |
| File size | ~40 MB |

## Measurement

**`rack_power`**

### Tags

| Tag | Values |
|---|---|
| `locationId` | `Berlin`, `Aachen`, `Saint-Etienne` |
| `rackId` | `rack-1`, `rack-2`, `rack-3` |

One rack per location:

| rackId | locationId | Baseline power |
|---|---|---|
| rack-1 | Berlin | ~3.2 kW |
| rack-2 | Aachen | ~4.8 kW |
| rack-3 | Saint-Etienne | ~2.1 kW |

### Fields

| Field | Unit | Description |
|---|---|---|
| `power_w` | Watts | Active power draw |
| `current_a` | Amperes | Current — derived from `power / (voltage × pf)` |
| `voltage_v` | Volts | Supply voltage (229–241 V range) |
| `power_factor` | — | Power factor (0.87–0.98) |

## Simulated patterns

| Pattern | Detail |
|---|---|
| **Daily cycle** | Sine wave peaking ~13:00, trough ~03:00 |
| **Weekends** | 55% of weekday baseline (reduced server load) |
| **Monthly drift** | +10% load growth across January |
| **Spikes** | ~0.8% chance per minute; 5–20 min duration; up to +45% above baseline |
| **Noise** | Continuous random walk around the target level |

## Import

In the InfluxDB UI: **Load Data → Buckets → your bucket → Upload CSV**, then
select `demo-power-data-racks_2026.csv`.

After upload, set the Data Explorer time range to:
- Start: `2026-01-01 00:00:00`
- Stop: `2026-02-01 00:00:00`

## Regenerating

The data was generated with a PowerShell script. Key parameters to adjust:

```powershell
$start        = [datetime]"2026-01-01T00:00:00Z"
$end          = [datetime]"2026-02-01T00:00:00Z"
$intervalSec  = 60       # seconds between samples
$spikeChance  = 0.008    # probability per sample of triggering a spike
$weekFactor   = 0.55     # weekend load as a fraction of weekday load
$monthDrift   = 0.10     # fractional load increase over the full period
```
