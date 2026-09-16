# rtk-home-base — private RTK base station (ZED-F9P + Raspberry Pi 3B+)

A hobby RTK base station built on [Stefal/rtkbase](https://github.com/Stefal/rtkbase),
documented end to end: hardware, wiring, install, config deltas, and the
survey-in → PPP procedure that gives the base a mm-class absolute position.
It serves:

* **Local NTRIP caster** on the Pi (`:2101/LOCAL`) — low-latency corrections
  for my own rover/drone on the home LAN or over Tailscale, independent of
  the internet
* **RTK2Go** — public community caster (own mountpoint)

Because the receiver is a real ZED-F9P, upstream RTKBase supports it
natively: the build is a clean upstream install plus a handful of
`settings.conf` changes. No custom scripts, no systemd drop-ins.

## Hardware (summary)

* **Raspberry Pi 3 Model B+** — microSD, USB 2.0
* **ArduSimple simpleRTK2B Budget** (u-blox **ZED-F9P**, L1/L2,
  GPS+GAL+GLO+BDS) over USB → `/dev/ttyACM0`
* **ArduSimple Survey GNSS multiband antenna** (TNC-f, not NGS-calibrated),
  ships with a 2.5 m TNC→SMA pigtail
* microSD for OS + raw logs

Full BOM and wiring in [`SETUP.md`](SETUP.md).

## Repo layout

```
rtk-home-base/
├── README.md            — this file
├── SETUP.md             — build + commission from scratch (BOM, wiring, install, survey-in + PPP)
├── settings-deltas.md   — settings.conf changes vs upstream RTKBase defaults
├── operations.md        — day-to-day ops: status, start/stop/restart, credential rotation
├── ppp/2026-09-01/      — record of the NRCan CSRS-PPP submission that fixed the base position (procedure + results; data files kept local)
└── secrets/             — credentials; kept local, never committed (see secrets/README.md)
```

## Quick install

Assumes Raspberry Pi OS Lite (Debian ≥ 12; Bookworm or Trixie) on the Pi
and the simpleRTK2B connected over USB. Full guide in [`SETUP.md`](SETUP.md).

```bash
# On the Pi:
cd ~ && git clone https://github.com/Stefal/rtkbase.git
cd rtkbase && tools/install.sh -u <your-user> --all release
```

Then apply [`settings-deltas.md`](settings-deltas.md) via the web UI (or by
editing `~/rtkbase/rtkbase/settings.conf`) and run the position procedure
(survey-in → PPP) in [`SETUP.md`](SETUP.md).

## Current state

Operational since 2026-07-08. Antenna at its permanent spot since
2026-08-31; broadcast position is the NRCan CSRS-PPP solution from
2026-09-02 (σ95 ≈ 4 / 3 / 12 mm) — see
[`ppp/2026-09-01/README.md`](ppp/2026-09-01/README.md).

## Related

* [Stefal/rtkbase](https://github.com/Stefal/rtkbase) — upstream RTKBase
* [NRCan CSRS-PPP](https://webapp.csrs-scrs.nrcan-rncan.gc.ca/geod/tools-outils/ppp.php) — free PPP service used for the base position
