# PPP submission — home base, 2026-09-01 raw log

> Coordinates in this public copy are rounded to ~1 km. The full-precision
> reports, the RINEX observation file and the NAV files are kept out of the
> repo (see "Files") — they are on the base's own disk and in local backups.

## Result (2026-09-02) — IGc20 / ITRF2020 at epoch 2026-09-01

Engine NRCan CSRS-PPP v5.15.5, Static, ultra-rapid products (`EMR1DCBULT`),
GPS + GLONASS (Galileo unavailable, see below).

| Coord | Value (rounded) | σ (95 %) |
|---|---|---|
| Latitude | **58.60 N** | ±3.7 mm |
| Longitude | **16.13 E** | ±2.6 mm |
| Ellipsoidal height | **~60 m** | ±11.6 mm |
| ECEF X / Y / Z | — (σ ±6.1 / ±3.1 / ±10.4 mm) | |

`settings.conf` value: `position='<lat> <lon> <ellipsoidal_h>'` — the PPP
values with all decimals.

**Applied + verified live 2026-09-02 18:20 CEST** via the web UI (Settings → Main service
→ Options → *Base coordinates* — the field is inside that collapsed panel, and its HTML
`pattern` requires 2–6 decimals on the elevation). Saving restarts the stream services
automatically. Verified afterwards: all three stream services run with the new `-p`
position, the local caster sourcetable advertises it, and RTK2Go reconnected
`[CC---] 47 kbps`. The pre-survey backup `settings.conf.bak-pre-survey2-20260901`
still holds the interim position.

### Delta vs the 24 h survey-in it replaces

| Axis | Δ (PPP − survey-in) |
|---|---|
| Lat | −0.202 m (S) |
| Lon | +0.007 m (E) |
| h | −0.234 m |

0.31 m 3D — the survey-in's true absolute bias (multipath + broadcast ephemeris +
atmosphere don't average out). Formal PPP uncertainty is mm-class, so this delta is
real systematic error, not noise.

### Quality gates — all passed

`IAR GPS 94.84 %` (gate >90 %) · `AVF 1.000` · carrier residuals L1 5 mm / L2 3 mm ·
2 204 of 2 206 epochs used.

### Caveats

1. **`ANT NOT FOUND`** — `ADVNULLANTENNA` isn't in the IGS antex, so the height is the
   antenna's *electrical phase centre*, not the physical ARP (a few cm high). Horizontal
   is unaffected, and relative RTK accuracy to any rover is unaffected.
2. **Galileo not processed** — the data was submitted the day after collection, so only
   ultra-rapid products existed and that line carries no Galileo. GLONASS was used but
   with estimated PCOs and no ambiguity fixing (`IAR GLO OFF`), so this is effectively a
   GPS-only solution. Resubmitted with IGS Finals on 2026-09-24 — the position was
   confirmed to within 5 mm but the sigmas did **not** tighten, because Galileo turned out
   to be unusable with this receiver. See "Finals rerun" below.
3. **First run was NAD83 and is void** — see below.

### Runs

| Run | Date | Frame | Products | Verdict |
|---|---|---|---|---|
| 1st | 2026-09-02 | NAD83(CSRS) | ultra-rapid | **VOID — not applied.** NAD83 tab was left selected at submit. NAD83 is pinned to the North American plate, so in Sweden it sits ~2.3 m off ITRF; the report's 1.50 m / 1.67 m / 0.49 m "correction" is that datum shift, not a position fix. |
| 2nd | 2026-09-02 | IGc20 (ITRF2020) | ultra-rapid | **Valid — the applied result above.** |
| 3rd | 2026-09-24 | IGc20 (ITRF2020) | **Final** (`EMR0MGBFIN`) | **Valid — confirms run 2. Not applied**, see below. |

**Lesson: always check `SYST` in the `.sum` POS block (and `FRAME` in `.pos`) before
applying anything.** A wrong-datum solution passes every quality gate — the residuals,
ambiguity resolution and variance factor were identical in both runs.

### Finals rerun (2026-09-24) — confirmation, not correction

The same 30 s file resubmitted once IGS Finals were published, to pick up Galileo and
better orbits/clocks. Verified `SP3`/`CLK` read `EMR0MGBFIN` (not `...ULT`) and `SYST`
reads `IGc20`.

| | Δ vs the applied position | σ (95 %) |
|---|---|---|
| Lat | −1.6 mm | 3.7 mm (unchanged) |
| Lon | +2.7 mm | 2.6 mm (unchanged) |
| Height | +4.8 mm | 11.5 mm (was 11.6) |

`IAR GPS` 94.84 % → 95.36 %, `AVF` 1.000, residuals unchanged. Every difference sits
inside the formal uncertainty, so **the run-2 position was left in service**. A 5 mm shift
is well below the real error floor here, which is dominated by the uncalibrated antenna's
unknown phase-centre offset (several cm in height).

**Galileo gives nothing with this receiver, and cannot.** The Finals run did process
Galileo — that warning disappeared — but the `.sum` shows `OBS E C1X L1X`, i.e. E1 only,
with `IAR GAL OFF` and phase residuals of exactly `0.000` at every elevation: Galileo
phase was not used in the solution, only its pseudoranges. The RINEX carries E1 **and**
E5b (`C7X/L7X`). The likely cause is that the standard iono-free Galileo combination is
E1+**E5a**, while the ZED-F9P tracks E1+**E5b** — in which case no submission setting will
produce dual-frequency Galileo at NRCan with this receiver. Don't spend another run
chasing it.

## Files (local only, not in this repo)

| File | Size | Contents |
|---|---|---|
| `HOMEBASE_30s.26O.gz` | 3.4 MB | RINEX 3.04 OBS, 30 s, 2 206 epochs — **the file submitted** |
| `HOMEBASE_30s.26N` | 878 kB | matching broadcast NAV |
| `HOMEBASE_1Hz.26O.gz` | 95.6 MB (383 MB raw) | RINEX 3.04 OBS, 1 Hz, 66 172 epochs — full-rate original |
| `HOMEBASE_1Hz.26N` | 878 kB | matching broadcast NAV |
| `full_output/` | | NRCan report set, NAD83 run (void) |
| `full_output_itrf/` | | NRCan report set, ITRF run (valid, applied): `.sum`, `.pos`, `.csv`, `.pdf`, `.clk`, `.tro` |
| `full_output_finals/` | | NRCan report set, IGS Finals run (valid, confirms the applied position) |

**Submit the 30 s file.** Both NRCan and AUSPOS decimate to 30 s internally for
static solutions, so the 1 Hz version buys no accuracy — only upload time and the
risk of hitting a file-size limit. NAV files are optional either way (PPP uses
precise ephemeris).

## Source data

`~/rtkbase/rtkbase/data/2026-09-01_00.zip` (150 MB) — RTKBase's nightly archive,
one continuous `2026-09-01_00-00-00_GNSS-1.ubx` (385 MB).

**Coverage is 18h22m, not a full 24 h day**: `str2str_file` was stopped at
2026-09-01 18:22 UTC (20:22 CEST — a clean `systemctl stop`, no error). The log
therefore runs 2026-09-01 00:00:00 → 18:22:51 UTC. Well above NRCan's needs for
a static solution, but a full 24 h would tighten the height sigma slightly.

Quality: **66 172 epochs** over 66 171 s → effectively 100 % 1 Hz coverage, no
gaps. 42–43 satellites per epoch, GPS + GLONASS + Galileo + BeiDou + QZSS,
dual-frequency. 4 163 UBX parse errors (~0.006 % of the stream — normal).

## Header baked into the RINEX

- **Marker:** `HOMEBASE`
- **Receiver:** `0/UBLOX_ZED-F9P/1.0`
- **Antenna:** `0/ADVNULLANTENNA` — placeholder. The ArduSimple Survey GNSS
  multiband antenna is **not** NGS-calibrated, so PPP returns the *electrical
  phase centre*, not the physical ARP. Expect a few cm systematic height offset;
  irrelevant for relative RTK accuracy.
- **Approx position (ECEF m):** the 2026-09-01 survey-in LLH converted to ECEF
  (WGS84).
- **Antenna delta H/E/N:** `0/0/0`
- **Interval:** 1.000 s (1 Hz file) / 30 s (submitted file)

## How it was built

On the Pi (`convbin` ships with RTKBase's RTKLIB), ~2 min:

```bash
mkdir -p ~/ppp_work/2026-09-01 && cd ~/ppp_work/2026-09-01
unzip -o ~/rtkbase/rtkbase/data/2026-09-01_00.zip
/usr/local/bin/convbin \
    -r ubx -v 3.04 \
    -od -os -ot -ol \
    -ti 1 -tt 0.5 \
    -hm HOMEBASE \
    -ha "0/ADVNULLANTENNA" \
    -hp "<survey-in ECEF X/Y/Z, m>" \
    -hr "0/UBLOX_ZED-F9P/1.0" \
    -ho "Axel Brinkeby/Private" \
    -o HOMEBASE_1Hz.26O \
    -n HOMEBASE_1Hz.26N \
    2026-09-01_00-00-00_GNSS-1.ubx
gzip -k HOMEBASE_1Hz.26O
```

The 30 s version is the same command with `-ti 30` and the `HOMEBASE_30s` output
names — 2 206 epochs, complete coverage, 12.8 MB → 3.4 MB gzipped.

Cleanup done 2026-09-02: `~/ppp_work/` removed (~870 MB) and `str2str_file`
disabled (`systemctl disable --now`) to stop SD wear. The source archives stay in
`~/rtkbase/rtkbase/data/`, so this RINEX can be rebuilt at any time; re-enable
`str2str_file` if a future PPP session needs fresh raw data.

## Submission procedure (as done)

**Primary — NRCan CSRS-PPP** (uses GPS + Galileo from the file when products
allow; AUSPOS is GPS-only):

1. <https://webapp.csrs-scrs.nrcan-rncan.gc.ca/geod/tools-outils/ppp.php> → log in
   (free account, one-time registration)
2. Mode **Static**, reference frame **ITRF** (the form defaults to the NAD83 tab —
   see "Runs" above for what happens if you miss this)
3. Upload `HOMEBASE_30s.26O.gz`
4. Antenna model `ADVNULLANTENNA`, antenna height 0 m (baked into the header)
5. Result by email within hours (`.sum`, `.pos`, `.csv`, `.pdf`)

**Cross-check — AUSPOS** (<https://gnss.ga.gov.au/auspos>, no account, email only).
Different method — a relative network solution against the ~15 nearest IGS/APREF
stations, not PPP — so agreement within 1–2 cm is a genuine independent validation.
Same `.gz`; if `ADVNULLANTENNA` isn't in their antenna list pick `NONE`/unknown,
height 0.

Expect `ANT NOT FOUND` (or equivalent) from both — `ADVNULLANTENNA` isn't in the
IGS antex. The returned position is then the electrical phase centre, a few cm
above the true ARP in height. Harmless for relative RTK accuracy.

**Quality gates** (from `.sum`): ambiguity resolution > 90 % fixed, carrier-phase
residuals ≤ ~1 cm, a-posteriori variance factor ≈ 1.0.
