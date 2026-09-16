# settings.conf — changes vs upstream RTKBase defaults

Not a full `settings.conf` snapshot (upstream defaults move with releases
and most of it is generic) — only what changes for THIS install, with
reasons.

The live file is `~/rtkbase/rtkbase/settings.conf` on the Pi — the only
authoritative copy. This document lists *what* changes and why.

## `[main]`

| Key | Value | Reason |
|---|---|---|
| `position` | `'<lat> <lon> <h>'` | Set from survey-in, then replaced by the PPP result (see [`SETUP.md`](SETUP.md)). After install this holds RTKBase's default (Nantes, FR — `47.098… -1.265…`); replace it. Height is **ellipsoidal**. |
| `com_port` | `'ttyGNSS'` | **Set automatically by the installer.** RTKBase creates a udev symlink `/dev/ttyGNSS → ttyACM0` and points here — leave it. |
| `com_port_settings` | `'115200:8:n:1'` | u-blox default. |
| `receiver` | `'U-blox_ZED-F9P'` | **Auto-detected.** Native receiver profile in RTKBase — no custom config needed. Tagged in the sourcetable. (`receiver_firmware='1.51'` is filled in too.) |
| `receiver_format` | `'ubx'` | str2str consumes UBX from the F9P and converts to RTCM3 for the casters. |
| `antenna_info` | `'ADVNULLANTENNA'` | RTKBase default, and correct here: the Survey multiband antenna is **not NGS-calibrated**. Placeholder antex → PPP returns the phase centre, not the ARP. Fine for relative RTK (see the antenna caveat in SETUP.md). |

## `[local_ntrip_caster]`

| Key | Value | Reason |
|---|---|---|
| `local_ntripc_port` | `'2101'` | NTRIP default, expected by the rover. |
| `local_ntripc_mnt_name` | `'LOCAL'` | Mountpoint name the rover points at. |
| `local_ntripc_user` / `_pwd` | empty (anonymous) | Caster is only reachable on the home LAN/tailnet — low risk. **Note:** if the Pi is ever exposed to the internet, set user/pwd. |

## `[ntrip_A]` (RTK2Go push)

Credentials are kept locally in `secrets/ntrip-creds.local` (gitignored).
Values on the Pi:

| Key | Value |
|---|---|
| `svr_addr_a` | `rtk2go.com` |
| `svr_port_a` | `2101` |
| `svr_pwd_a` | community temp password until activation, then personal (see `secrets/`) |
| `mnt_name_a` | `<your chosen mountpoint>` |
| `rtcm_msg_a` | upstream default |

**Two hard-won RTK2Go lessons:**

1. **Use the community temp password until the activation email arrives.**
   Pushing with a not-yet-activated personal password for hours matches
   RTK2Go's IP-ban trigger ("Repeated connection attempts using a
   non-working password can lead to your IP being temporarily banned").
2. **`systemctl disable --now` before rotating credentials**, not just
   `stop`. Auth failures are indistinguishable from network glitches at the
   str2str level, so a running push service grinds on with a bad password —
   against RTK2Go, straight into repeated 4 h IP bans. Procedure in
   [`operations.md`](operations.md).

## `[general]` — per-install secrets (never share)

- `web_password_hash` — scrypt hash of the web-UI password.
- `flask_secret_key` — signing key for session cookies.

Live only in `settings.conf` on the Pi. Restore from an SD backup if lost.

## Deliberately unused

- `[ntrip_B]` — second NTRIP push (e.g. Onocoy). This base only pushes to
  the local caster + RTK2Go.
- `[local_storage]` — logs stay in the default data directory on the SD
  card; `str2str_file` is only enabled while collecting data for PPP.
