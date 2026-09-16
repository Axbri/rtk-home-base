# Base setup — building the home station (ZED-F9P + Pi 3B+)

End-to-end guide for building and commissioning a private RTK base station
from scratch: what it is, the hardware, how everything connects, software
install, and the position procedure (survey-in → NRCan CSRS-PPP). Companion
to [`operations.md`](operations.md) (day-to-day ops) and
[`settings-deltas.md`](settings-deltas.md) (config changes).

## What it is (one paragraph)

A fixed GNSS receiver at a **precisely known position** sees the same
satellites as a moving rover (drone, robot). Because the base knows exactly
where it is, it can measure the current error in the satellite signals
(atmosphere, orbits, clocks) and broadcast those errors as RTCM 3
corrections over the internet (NTRIP). A rover applying the corrections
resolves the carrier-phase ambiguities and reaches **cm-level position**
(RTK Fix) instead of the usual ~5 m. The catch: the corrections are only as
good as the base's knowledge of its own position — hence the survey/PPP
procedure in the second half of this document.

This base serves two consumers:

1. **Local NTRIP caster** on the Pi (`:2101/LOCAL`) — low-latency,
   internet-independent path for your own rover on the LAN/tailnet
2. **RTK2Go** — public community caster (own mountpoint)

## Hardware (BOM)

| Part | Model |
|---|---|
| GNSS receiver | ArduSimple **simpleRTK2B Budget** (u-blox **ZED-F9P**, L1/L2, GPS+GAL+GLO+BDS) |
| Antenna | ArduSimple **Survey GNSS multiband** (TNC-f, IP66, not NGS-calibrated) — 2.5 m TNC→SMA-m pigtail included |
| SBC | **Raspberry Pi 3 Model B+** (microSD, USB 2.0) |
| Storage | microSD class A1/A2, ≥16 GB (OS + raw logs) |
| Power | 5 V / 3 A USB supply (feeds the Pi; the Pi's USB feeds the F9P; the F9P feeds the antenna bias) |
| *Roof mount only* | SMA-f/SMA-f **lightning arrestor** (DC-pass, GNSS band) + SMA extension cable |

### Component notes

* **ZED-F9P (dual-band L1/L2)** is plenty for hobby RTK, and it is the
  receiver upstream RTKBase is built around → zero extra configuration.
* **The Survey multiband antenna is not NGS-calibrated.** This only affects
  *absolute height* at the cm level (see the PPP caveat at the end).
  Relative RTK to your own rover is unaffected. Use
  `antenna_info='ADVNULLANTENNA'`.
* **RG-58 / the included pigtail** is fine for short runs. For permanent
  runs >7–8 m, go to LMR-400.

## How everything connects

### RF chain (antenna → receiver)

The antenna has a TNC-f connector; the included 2.5 m pigtail is TNC→SMA-m
and goes straight into the simpleRTK2B's SMA-f. Two mounting options:

**Option A — balcony/window (simplest):**

```
Antenna (TNC-f, IP66)  — clear sky view, on a 5/8" thread or magnetic mount
  │  included 2.5 m TNC→SMA-m pigtail
  ▼
simpleRTK2B Budget antenna input (SMA-f)
```

2.5 m can reach from a window sill / balcony rail to the Pi just inside.
Lightning arrestor optional at low mounting height. What matters is **open
sky** without reflective surfaces nearby (multipath).

**Option B — roof/mast (best sky view, needs protection):**

```
Antenna (TNC-f, roof)
  │  included 2.5 m TNC→SMA-m pigtail
  ▼
SMA-f/SMA-f lightning arrestor (DC-pass, GNSS band) — GROUNDED at the wall entry
  │  SMA extension cable
  ▼
simpleRTK2B Budget antenna input (SMA-f)
```

Roof-mount notes:

* **A lightning arrestor is mandatory** for a roof antenna in Sweden. It
  must be **DC-pass** (the F9P feeds the antenna LNA 3–5.5 V bias up the
  coax) and properly grounded. One strike without it destroys the F9P.
  Until you have one, unplug the antenna during thunderstorms.
* **Weatherproof every outdoor SMA joint** with self-amalgamating tape —
  SMA is more water-sensitive than TNC/N.
* Keep the antenna away from reflective surfaces (multipath).

### Verify antenna bias (both options)

Once at bring-up: check the F9P's antenna supervisor (u-center or
`ubxtool`) or measure 3.3–5 V DC on the SMA centre pin. Default is on, but
a disabled bias looks exactly like "no satellites".

### Data + power (receiver → Pi)

```
simpleRTK2B Budget
  │  USB (POWER+GPS port) — powers the board; the board powers the antenna bias
  │  → /dev/ttyACM0 @ 115200 (RTKBase adds a udev symlink /dev/ttyGNSS)
  ▼
Raspberry Pi 3 Model B+
  │  home LAN (Ethernet or WiFi) · web UI :80
  │
  ├─ str2str_tcp.service              UBX from receiver → internal bus 127.0.0.1:5015
  │    ├─ str2str_ntrip_A.service          → rtk2go.com:2101/<your mountpoint>
  │    ├─ str2str_local_ntrip_caster       → NTRIP caster on :2101, mountpoint LOCAL
  │    └─ str2str_file.service             → raw UBX → microSD (temporarily, for PPP)
  ├─ rtkbase_archive.timer                 → zips + rotates raw logs nightly
  └─ rtkbase_web.service                   → Flask web UI on :80
```

The whole station runs off one 5 V / 3 A plug. Use a good supply — Pi 3B+
plus F9P draws more than a phone charger delivers, and a brown-out mid
survey-in restarts the 24 h clock. Only the antenna (+ arrestor) lives
outdoors; receiver + Pi stay inside.

## Software install

### 1. Base OS

* Raspberry Pi OS **Lite 64-bit**, headless (set a hostname such as
  `homebase`, enable SSH at flash time in Raspberry Pi Imager). RTKBase
  needs Debian **≥ 12** (Bookworm or newer). **Debian 13 "Trixie" works**
  but needs RTKBase **≥ v2.7.0** (release note: "Now compatible with
  Debian 13 (Trixie)"); `--all release` below fetches the latest release,
  so just confirm it is 2.7.0+.
* Ethernet is most stable; WiFi works. Give the Pi a fixed IP or DHCP
  reservation so you can find the web UI.
* (Optional) Tailscale for remote access:
  `curl -fsSL https://tailscale.com/install.sh | sh && sudo tailscale up`.
  With MagicDNS on, the rover can then use `http://<hostname>:2101/LOCAL`
  from anywhere — see [`operations.md`](operations.md).

### 2. RTKBase (upstream)

**Don't roll your own str2str scripts** — [Stefal/rtkbase](https://github.com/Stefal/rtkbase)
wraps RTKLIB's `str2str` with a web UI, NTRIP outputs, caster mode and log
archiving:

```bash
cd ~ && git clone https://github.com/Stefal/rtkbase.git
cd rtkbase && tools/install.sh -u <your-user> --all release
```

Replace `<your-user>` with your Linux username. This installs RTKLIB, the
Flask web UI (`rtkbase_web.service`, port 80), the `str2str_*` services,
chrony + gpsd time sync and the nightly archive timer. The build takes a
while on a Pi 3B+ but works fine.

### 3. No custom configuration needed

Upstream RTKBase detects the F9P natively and the SD card is the default
data location. Nothing to install beyond step 2.

### 4. settings.conf

Apply the changes in [`settings-deltas.md`](settings-deltas.md) via the
web UI or by editing `~/rtkbase/rtkbase/settings.conf`. The installer
auto-detects the F9P and already sets `com_port='ttyGNSS'` (udev symlink →
ttyACM0), `receiver='U-blox_ZED-F9P'`, `receiver_format='ubx'` and
`antenna_info='ADVNULLANTENNA'`. What you fill in yourself: the local
caster mountpoint name (`LOCAL`, empty by default), the RTK2Go details, and
`position=` (from the procedure below).

### 5. Web-UI password

Set a password before exposing the UI: write a long random value into
`new_web_password=` in `settings.conf`, save, and
`sudo systemctl restart rtkbase_web`. RTKBase scrypt-hashes it into
`web_password_hash` and clears the plaintext field. Archive the plaintext
in `secrets/` (gitignored).

### 6. NTRIP outputs

* **Local caster:** enable `str2str_local_ntrip_caster` in the web UI.
  This is what your rover uses. It is anonymous (no user/password) — fine
  on a LAN/tailnet, set credentials if you ever expose it to the internet.

* **RTK2Go:** register a mountpoint at rtk2go.com (email form). **Two
  hard-won lessons:**
  1. Pushing with a not-yet-activated personal password for hours matches
     RTK2Go's IP-ban trigger. Use the documented community temp password
     until the activation email arrives, then switch to your personal one.
  2. When rotating credentials: `systemctl disable --now` the push service
     **first**, then edit, then re-enable. Auth failures look like network
     glitches to str2str, so a merely restarted service happily retries a
     bad password straight into a 4 h ban cycle. (See
     [`operations.md`](operations.md).)

  Clients pulling *from* RTK2Go need a valid **email address as the NTRIP
  username** (any password). The local `LOCAL` caster needs nothing.

## Initial base position — survey-in, then PPP

The base broadcasts its own position in the RTCM stream (messages
1005/1006), and every rover position is measured *relative to it*. Getting
it wrong doesn't break RTK Fix — it silently shifts the whole survey. Two
stages:

### Stage 1 — survey-in (day 1, gets you operational)

RTKBase has no built-in survey-in, so drive the F9P's TMODE3 directly with
`ubxtool` (from gpsd, installed by RTKBase). Only `str2str_tcp` holds the
serial port (gpsd reads the TCP bus), so stop it while you talk to the
receiver:

```bash
sudo systemctl stop str2str_tcp
ubxtool -f /dev/ttyGNSS -P 27.50 \
    -z CFG-TMODE-MODE,1 \
    -z CFG-TMODE-SVIN_MIN_DUR,86400 \
    -z CFG-TMODE-SVIN_ACC_LIMIT,25000     # 24 h, 2.5 m gate
sudo systemctl start str2str_tcp
```

(`-P` is the receiver's protocol version; check with `ubxtool -p MON-VER`.)
The keys land in RAM, so don't reboot mid-survey. After 24 h, stop
`str2str_tcp` again and harvest:

```bash
ubxtool -f /dev/ttyGNSS -P 27.50 -p NAV-SVIN        # valid=1 active=0, meanAcc
ubxtool -f /dev/ttyGNSS -P 27.50 -p NAV-HPPOSLLH    # lat/lon/height (ellipsoidal!)
```

Write that lat/lon/**ellipsoidal** height (not MSL) into `settings.conf`
`position=` — via the web UI: *Settings → Main service → Options → Base
coordinates* (a collapsed panel; the height field requires 2–6 decimals).
Saving there restarts the stream services. RTKBase broadcasts the position
from `settings.conf` (`str2str -p`) regardless of the receiver's own TMODE,
so it survives reboots.

Ours converged to ~6 cm (its own estimate) after 24 h. That is good enough
to fly RTK the same week — *relative* accuracy to the rover is already
cm-class. But the survey-in's *absolute* position carries a real systematic
bias (ours proved to be **0.31 m** in 3D — multipath, broadcast ephemeris
and atmosphere don't average out in 24 h). Hence PPP.

### Stage 2 — raw logging for PPP

Enable `str2str_file` **temporarily** and log ≥24 h of raw UBX
observations. On a Pi 3B+ the logs land on the microSD → to limit SD wear,
disable `str2str_file` again once the RINEX is built. Grab the full-day
archive zip from `~/rtkbase/rtkbase/data/`.

### Stage 3 — PPP via NRCan CSRS-PPP

**What/why:** Natural Resources Canada runs a free (including commercial
use) Precise Point Positioning service: upload ~24 h of raw observations,
their solver processes them against IGS precise orbits/clocks and emails
back your antenna position with **mm-level formal uncertainty**. This
replaces the survey-in guess with ground truth.

**Prerequisites:** antenna in its final position, ≥24 h of raw UBX logged,
a free NRCan account
(<https://webapp.csrs-scrs.nrcan-rncan.gc.ca/geod/tools-outils/ppp.php>).

**Build the RINEX** (on the Pi; `convbin` ships with RTKLIB/RTKBase, ~2 min):

```bash
mkdir -p ~/ppp_work/<date> && cd ~/ppp_work/<date>
unzip ~/rtkbase/rtkbase/data/<date>_00.zip
convbin \
    -r ubx -v 3.04 \
    -od -os -ot -ol \
    -ti 30 -tt 0.5 \
    -hm HOMEBASE \
    -ha "0/ADVNULLANTENNA" \
    -hp "<approx ECEF X/Y/Z from survey-in, m>" \
    -hr "0/UBLOX_ZED-F9P/1.0" \
    -o HOMEBASE_30s.<yy>O \
    -n HOMEBASE_30s.<yy>N \
    <date>_00-00-00_GNSS-1.ubx
gzip -k HOMEBASE_30s.<yy>O
```

Use `-ti 30`: both NRCan and AUSPOS decimate to 30 s internally for static
solutions, so 1 Hz buys nothing but upload time and file-size risk. Tip:
`convbin`'s progress output is one giant CR-separated line — pipe logs
through `tr '\r' '\n'` before `tail`.

**Submit:** log in → **CSRS-PPP** → mode **Static** → **ITRF** tab
(**not** NAD83 — the form defaults to NAD83, and a NAD83(CSRS) solution
sits ~2.3 m off ITRF in Sweden while passing every quality gate; verify
`SYST` in the `.sum` says `IGc20`). Upload `HOMEBASE_30s.<yy>O.gz`.
Results by email within hours (`.sum`, `.pos`, `.csv`, `.pdf`).

**Product timing:** submitting the day after collection uses ultra-rapid
products — Galileo is silently dropped and GLONASS runs without ambiguity
fixing, so the solution is effectively GPS-only. Still mm-class. Resubmit
the same file once IGS Finals are out (~2 weeks) to add Galileo and tighten
the sigmas.

**Quality gates** (from `.sum`): ambiguity resolution >90 % fixed,
carrier-phase residuals ≤ ~1 cm, a-posteriori variance factor ≈ 1.0.

**Cross-check (optional, free):** upload the same `.gz` to AUSPOS
(<https://gnss.ga.gov.au/auspos>, no account). Different method (network
solution against nearby IGS stations, not PPP), so agreement within
~1–2 cm is genuine validation.

**⚠️ Antenna caveat:** because the antenna is **not** NGS-calibrated we
use `ADVNULLANTENNA` → PPP reports `ANT NOT FOUND` and returns the
antenna's **electrical phase centre**, not the physical mount point (ARP).
A systematic height offset of a few cm remains in the absolute height.
**For private RTK this does not matter** — your rover's relative accuracy
is unaffected, and RTK2Go neighbours get correct *relative* geometry. If
ArduSimple ever publishes an antex/PCO for the antenna, use its name in
`-ha` for true ARP height.

**Apply the result:** enter the PPP lat/lon/ellipsoidal height in the web
UI (*Main service → Options → Base coordinates*), which restarts the stream
services. If you edit `settings.conf` by hand instead, also
`sudo systemctl restart rtkbase_web` — the web app caches settings at
startup and would keep showing the old position (the RTCM broadcast is
already correct). Back up `settings.conf` first. Putting the F9P itself in
fixed mode (`CFG-TMODE-MODE=2` + ECEF) is optional — RTKBase broadcasts
from `settings.conf` regardless.

Record of the actual submission, results and deltas:
[`ppp/2026-09-01/README.md`](ppp/2026-09-01/README.md).

## Commissioning checklist

* [ ] `systemctl is-active str2str_tcp rtkbase_web str2str_local_ntrip_caster` — all `active`
* [ ] Web UI shows satellites across GPS+GAL+GLO+BDS on **L1+L2**
* [ ] Local caster answers with a SOURCETABLE listing `LOCAL`. **Note:**
  the RTKLIB caster needs a real NTRIP request (with a `User-Agent`
  header) — a bare `curl http://<pi>:2101` returns 0 bytes. Probe with:
  ```bash
  exec 3<>/dev/tcp/127.0.0.1/2101
  printf 'GET / HTTP/1.0\r\nUser-Agent: NTRIP probe\r\n\r\n' >&3
  timeout 3 head -c 600 <&3      # → SOURCETABLE 200 OK ... STR;LOCAL;...
  ```
  or with a real NTRIP client. The mountpoint serves RTCM3 MSM7
  (1005/1006 + 1077/1087/1097/1107/1127 + 1230).
* [ ] RTK2Go SOURCETABLE lists your mountpoint with the right lat/lon
  (populates ~200 s after the push starts). Read it at
  `http://monitor.use-snip.com/?hostUrl=rtk2go.com&port=2101` — the Pi
  itself can't fetch the full table (connection reset after ~14 KB).
* [ ] `str2str_file` wrote raw UBX during the survey window (disable again afterwards)
* [ ] A rover reaches **RTK Fix** against the local caster

## Operations

Start/stop/restart and credential rotation: [`operations.md`](operations.md).

## Related

* [`settings-deltas.md`](settings-deltas.md) — settings.conf changes vs upstream
* [`ppp/2026-09-01/README.md`](ppp/2026-09-01/README.md) — the PPP submission record
* [Stefal/rtkbase](https://github.com/Stefal/rtkbase) — upstream RTKBase
