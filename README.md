# RTCM QC Viewer

A self-contained, client-side HTML tool for visualizing quality metrics from GNSS RTCM3 binary logs. No server, no build step — open the file in a browser and drag a `.rtcm` log onto the page.

## Features

| Tab | Content |
|-----|---------|
| **Overview** | File summary, epoch count, data rate, gap list, message type histogram |
| **Signals** | CNR histogram, CNR vs time scatter, satellite count vs time (SNR ≥ 32 dBHz filter) |
| **Satellites** | Per-satellite table: CNR, observation count, cycle slips, MP1/MP2 RMS, MW std dev |
| **Timeline** | Epoch interval distribution, data-rate heat map, gap annotations |
| **Multipath/MW** | Bar charts of MP1/MP2 RMS and Melbourne–Wübbena std dev by satellite |
| **Messages** | Raw message-type breakdown across all RTCM3 message types in the file, including proprietary types 4054 (GEODNET) and 4045 (XONA) |
| **Skyplot & Visibility** | Polar skyplot with color-coded trajectories, expected vs tracked satellites line chart, elevation mask & date settings, CSV data export |
| **Epoch Latency** | Statistical summary, charts, and detailed table for GEODNET packet arrival delays and sync durations |

## Usage

1. Open `index.html` in any modern browser (Chrome, Firefox, Edge).
2. Drag and drop an RTCM3 binary log file onto the drop zone.
3. The file is parsed entirely in the browser — no data leaves your machine.

Supported message types: MSM4, MSM5, MSM6, MSM7 for GPS, GLONASS, Galileo, QZSS, SBAS, BeiDou, IRNSS.

## Technical Notes

### Signal naming
Signal names follow the two-character RTCM3 convention taken directly from the `msm_sig_*` tables in RTKLIB's `rtcm3.c` (e.g. `1C` = L1 C/A, `2W` = L2 Z-tracking, `5I` = L5 I-channel, `2I` = BDS B1I). The integer before the letter is the band/frequency number; the letter identifies the tracking mode.

### Frequency lookup
Frequencies are keyed by `[system][sigName]` because the same two-character code can refer to different frequencies across constellations (e.g. GPS `2I` vs BDS `2I`). GLONASS frequencies are computed per-satellite using FCN from message type 1020 or a built-in default table.

### Multipath computation
MP1/MP2 are computed using the standard dual-frequency linear combination:

```
beta(f) = (f_GPS_L1 / f)²
ion     = -(L1 - L2) / (beta1 - beta2)          [metres]
MP1     = P1 - L1 - 2·beta1·ion
MP2     = P2 - L2 - 2·beta2·ion
```

A 50-epoch sliding-window mean is subtracted per arc to remove the integer ambiguity offset. Arc boundaries are detected from lock-time decreases. The final RMS subtracts the residual mean of the corrected series. This matches the algorithm in `geo_qc.cpp`.

### Melbourne–Wübbena
```
LW  = (L1·f1 - L2·f2) / (f1 - f2)              [metres]
PN  = (P1·f1 + P2·f2) / (f1 + f2)              [metres]
MW  = (LW - PN) / lambda_WL                      [cycles]
```

### Skyplot & Orbit Calculations
- **Satellite Position Propagation**: Orbits are computed from broadcast ephemeris parameters using standard Keplerian formulas (for GPS, Galileo, BeiDou, QZSS, and IRNSS) and a Runge-Kutta 4th-order numerical integration solver (for GLONASS).
- **Receiver Coordinates**: Derived from the average of decoded RTCM 1005/1006 Antenna Reference Point (ARP) messages (38-bit signed integer values decoded using floating-point math).
- **Azimuth & Elevation**: Coordinates are translated from ECEF to local ENU (East-North-Up) using the reference receiver geodetic position:
  ```
  [azimuth, elevation] = enu2azel(ecef2enu(rxXYZ, satXYZ))
  ```
- **Dynamic Re-parsing**: If the approximate date is manually modified in the UI, the page automatically re-parses the raw binary file buffers to recalculate the GPS reference week number and update the decoded orbit coordinates from scratch.
- **GEODNET / BASE Log Parsing**: Auto-detects the `$GEOD,received_utc_time_is_ms,buffer_length,raw_buffer\r\n` and `$BASE,received_utc_time_is_ms,buffer_length,raw_buffer\r\n` patterns. The binary parser scans raw byte offsets without string splitting to preserve binary data payloads containing commas or newlines.
- **Arrival Latency & Sync Statistics**: For GEODNET logs, calculates epoch reception latency (first observation message arrival delay), sync latency (last synchronized message arrival delay), and total epoch reception duration relative to the epoch's true measurement time.
- **RINEX 3.04 & 2.11 Exporters**:
  - **Standard-Compliant Headers**: Formats metadata headers (`OBSERVER / AGENCY`, `REC # / TYPE / VERS`, `ANT # / TYPE`, `TIME OF FIRST OBS`, `TIME OF LAST OBS`) strictly following standard column specifications and aligning labels to start exactly at column 61 (80-character limit). Strips null-padding bytes from decoded equipment strings.
  - **RINEX 3.04**: Exports GPS, GLONASS, Galileo, and BeiDou observations.
  - **RINEX 2.11**: Filters out unsupported constellations (BeiDou and QZSS), maps GPS/GLONASS P-code signals to `P1`/`P2`, maps Galileo signals to bands 1/5/6/7/8, and resolves signal collision/overwrite issues (e.g. `1C` and `1W` on L1) using dynamic prioritized band mapping.
  - **Signal Validation**: Only signal names matching the RTCM3 two-character pattern (`\d[A-Z]`, e.g. `1C`, `2W`, `5I`) are included in the RINEX observation type headers and records; malformed codes are silently skipped.

## Dependencies

Chart.js 4 (loaded from CDN at runtime). No other dependencies.

## License

MIT
