# secrets/ — credentials for the home base

This directory holds credentials and per-install secrets for the base. The
`*.local` files are gitignored and must never be committed.

Expected files (created when you set up each service):

- `ntrip-creds.local` — RTK2Go push credentials (mountpoint + password,
  both the community temp password and the personal one).
- `rtkbase-web-pwd.local` — RTKBase web-UI password (plaintext archive; the
  live scrypt hash lives in `settings.conf` on the Pi).

## Handling

Keep these files **local**. The RTK2Go and web-UI passwords are low-impact
if they leak on your own LAN, but directly abusable if they end up public
(a public git repo, cloud storage).

For backup, copy the directory to a private location (USB stick, NAS on
your own network). A fresh SD-card image of the Pi also captures the live
`settings.conf` with the hashed web password.

## Rotation

When a credential changes: edit the file here, then apply it on the Pi
using the procedure in [`../operations.md`](../operations.md)
("Rotating NTRIP credentials — in this order").
