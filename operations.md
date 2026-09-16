# Operations — status, start, stop, restart

Day-to-day operation of the running station. For what the station *is*
and how to build it, see [`SETUP.md`](SETUP.md).

## Access

- **Web UI:** `http://<pi-ip>/` on the home LAN (or the Tailscale
  hostname if enabled). Toggle, configure and monitor every service there.
- **SSH:** `ssh <your-user>@<pi-ip>`
- **Rover NTRIP URL:** `http://<pi-hostname>:2101/LOCAL` — over Tailscale
  with MagicDNS this works from anywhere on 4G, anonymous, independent of
  RTK2Go uptime and carrier port blocking. Use the hostname, not the
  numeric tailnet IP — it is typo-proof when entered by hand in the field.

## Service map

| Service | Role |
|---|---|
| `str2str_tcp` | receiver pipeline — UBX from the F9P → internal bus `127.0.0.1:5015`. Everything else feeds from here. |
| `str2str_ntrip_A` | push to RTK2Go (`rtk2go.com:2101/<your mountpoint>`) |
| `str2str_local_ntrip_caster` | local NTRIP caster on `:2101`, mountpoint `LOCAL` (what the rover uses) |
| `str2str_file` | raw UBX logging → SD (PPP source data; run temporarily) |
| `rtkbase_archive.timer` | zips + rotates the raw logs |
| `rtkbase_web` | Flask web UI on `:80` |

## Status

```bash
systemctl is-active str2str_tcp str2str_ntrip_A str2str_local_ntrip_caster rtkbase_web
journalctl -u str2str_tcp -n 30 --no-pager
journalctl -u str2str_ntrip_A -n 30 --no-pager   # RTK2Go push health
```

A healthy push shows `[CC---] ~47 kbps rtk2go.com/<mountpoint>`. Occasional
`[CW---] connecting...` lines are RTK2Go-side hiccups; str2str reconnects
by itself.

## Restart

**Receiver pipeline** — opens a new serial session and reconnects the
NTRIP outputs. Plain restart, nothing re-configures the receiver.

```bash
sudo systemctl restart str2str_tcp
```

**One NTRIP output only** (without disturbing the receiver):

```bash
sudo systemctl restart str2str_ntrip_A            # RTK2Go
sudo systemctl restart str2str_local_ntrip_caster # local caster
```

## Stop / start everything (e.g. hardware service)

```bash
sudo systemctl stop  str2str_tcp str2str_ntrip_A \
                     str2str_local_ntrip_caster str2str_file rtkbase_web

sudo systemctl start str2str_tcp str2str_ntrip_A \
                     str2str_local_ntrip_caster str2str_file rtkbase_web
```

## Reboot

```bash
sudo reboot
# All RTKBase services come back automatically at boot. Verify after ~30 s:
ssh <your-user>@<pi-ip> 'systemctl is-active str2str_tcp rtkbase_web'
```

## Changing the base position

Use the web UI: *Settings → Main service → Options → Base coordinates*
(collapsed panel; height needs 2–6 decimals). Saving restarts the stream
services. If you edit `settings.conf` by hand instead, also
`sudo systemctl restart rtkbase_web`, or the UI keeps showing the old
position (the RTCM broadcast is already correct).

## Rotating NTRIP credentials — in this order

Auth failures are indistinguishable from network glitches at the str2str
level, so a running push service grinds on with a bad password forever —
against RTK2Go, straight into repeated 4 h IP bans:

```bash
sudo systemctl disable --now str2str_ntrip_A   # DISABLE, not just stop
# edit the credential in settings.conf (and secrets/ntrip-creds.local)
sudo systemctl enable --now str2str_ntrip_A
journalctl -u str2str_ntrip_A -f               # expect a steady [CC---] connected
```

## Raw logging on/off (for PPP)

Only run `str2str_file` during the survey window to limit SD wear:

```bash
sudo systemctl enable --now str2str_file    # on, before ≥24 h logging
# ... build RINEX, submit to NRCan (see SETUP.md) ...
sudo systemctl disable --now str2str_file   # off again afterwards
```
