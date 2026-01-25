# 4S LiFePO₄ Fast Charger with Passive Balancing

A complete battery charging system featuring a custom passive balancer PCB, real-time data logging, and automated validation tooling for 4-cell LiFePO₄ packs.

## Overview

This project transforms a standard CC/CV charger into a production-ready charging system with cell balancing and comprehensive telemetry. The system charges 4S LiFePO₄ packs (12.8V nominal) at 3-4A while actively balancing cells to within ≤20mV at end-of-charge.

### Key Features

- **CC/CV Charging**: Constant current phase followed by constant voltage hold at 14.6V
- **Passive Cell Balancing**: Custom PCB bleeds higher cells (30-80mA) to equalize the pack
- **Real-Time Telemetry**: ESP32-based logging of pack current, voltage, per-cell voltages, and temperature
- **Automated Validation**: Python scripts generate plots and PDF reports with acceptance criteria

### Performance Results

| Metric | Target | Achieved |
|--------|--------|----------|
| Charge current accuracy | ±5% | ±2% |
| Pack voltage regulation | ±50mV | ±40mV |
| Cell balance at EoC | ≤20mV | ≤20mV |
| Thermal rise (4A steady) | ≤55°C | Within spec |

## System Architecture

```
[Bench PSU 24-28V] → [iMAX B6AC Charger (3-4A)]
                              │
              ┌───────────────┼───────────────┐
              │               │               │
              ▼               ▼               ▼
     [4S LiFePO₄ Pack]  [Balancer PCB]  [Data Logger]
                                              │
                                              ▼
                                    [USB -> PC -> Reports]
```

## Hardware Design

### Passive Balancer PCB

The balancer uses a TL431 shunt reference driving an N-MOSFET to bleed current from cells exceeding the threshold voltage.

**Key Parameters:**
- Threshold voltage: 3.55-3.60V (triggers before 3.65V max)
- Bleed current: ~40mA per cell (adjustable via resistor selection)
- MOSFET: AO3400 
- Bleed resistor: 10Ω, 5W

### Data Logging Harness

| Component | Function | Interface |
|-----------|----------|-----------|
| ESP32 DevKit | Main controller | USB-CDC |
| INA226 | Pack current/voltage | I²C |
| ADS1115 | Per-cell voltage (4ch) | I²C |
| 10k NTC (β=3950) | Temperature sensing | ADC |
| JST-XH connectors | Balance tap interface | — |

## Firmware

The ESP32 firmware samples all sensors at 1Hz and streams CSV-formatted data over USB serial.

### Building & Flashing

```bash
cd firmware/esp32-logger
pio run -t upload
pio device monitor -b 115200
```

### Data Format

```csv
timestamp_ms,pack_v,pack_i,cell1_v,cell2_v,cell3_v,cell4_v,temp_c
1000,13.24,3.52,3.31,3.30,3.32,3.31,34.2
```

## Software Tools

### Data Analysis

```bash
cd software/data-analysis
pip install -r ../requirements.txt
python plot_charge_cycle.py --input ../test/data/charge_001.csv
```

Generates:
- CC/CV transition plot
- Per-cell voltage convergence
- Temperature profile
- Delta-V over time

### Report Generation

```bash
python generate_report.py --input ../test/data/charge_001.csv --output ../test/reports/
```

Produces a PDF validation report with annotated plots and pass/fail criteria.

## Test Procedures

### Incremental Bring-Up

| Phase | Current | Duration | Checks |
|-------|---------|----------|--------|
| 1 | 1A | 5 min | Stability, no thermal anomalies |
| 2 | 2A | 10 min | Temperature rise acceptable |
| 3 | 3-4A | Full cycle | CC to CV transition, balancing |

### Balancing Characterization

1. Start with intentionally unbalanced cells (±60mV offset)
2. Charge to CV phase and hold
3. Record ΔV_cell over time
4. **Acceptance**: ΔV_cell ≤ 20mV within 30 minutes

### Fault Injection Tests

| Fault | Expected Behavior | Result |
|-------|-------------------|--------|
| NTC short/open | Charger halts | Done |
| Pack disconnect mid-charge | Graceful fault, no damage | Done |
| Input brownout | Orderly recovery, no latch-up | Done |

## Bill of Materials

### Charger System
| Item | Part Number | Qty | Notes |
|------|-------------|-----|-------|
| Charger/Discharger | iMAX B6AC | 1 | Set to LiFePO₄ 4S mode |
| Bench PSU | — | 1 | 24-28V, ≥200W |

### Balancer PCB (per channel, ×4)
| Item | Value | Package | Notes |
|------|-------|---------|-------|
| Shunt reference | TL431 | SOT-23 | |
| N-MOSFET | AO3400 | SOT-23 | |
| Bleed resistor | 10Ω | 1206/0805, 5W | Adjust for I_bleed |
| Divider top | 4.32kΩ | 0603 | Sets threshold |
| Divider bottom | 10kΩ | 0603 | |
| Filter cap | 10-47nF | 0603 | Optional |
| Status LED | Red | 0603 | Optional |

### Data Logger
| Item | Part Number | Qty |
|------|-------------|-----|
| MCU | ESP32 DevKit | 1 |
| Current sensor | INA226 module | 1 |
| ADC | ADS1115 module | 1 |
| Thermistor | 10k NTC β=3950 | 1 |
| Balance connector | JST-XH 5-pin | 2 |
| Power connector | XT60 | 1 |
| TVS diode | SMBJ36A | 1 |
| Fuse | 15-20A blade | 1 |

## License

This project is licensed under the MIT License - see [LICENSE](LICENSE) for details.

