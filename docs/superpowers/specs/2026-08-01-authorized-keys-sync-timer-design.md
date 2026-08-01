# Design: Periodic authorized_keys sync via systemd timer

Date: 2026-08-01

## Goal

When SSH public keys are added to the GitHub Gist, the Debian server should pick
them up automatically within ~5 minutes, without requiring a manual
`ansible-playbook` re-run. The server runs a systemd timer that periodically
re-fetches `authorized_keys` from the Gist URL already used by the playbook.

## Approach

Two systemd drop-in units managed by Ansible:

1. `authorized-keys-sync.service` (oneshot) — fetches the keys.
2. `authorized-keys-sync.timer` — triggers the service every 5 minutes.

The existing one-shot `wget` task in `main.yml` stays, so the first playbook run
applies keys immediately before the timer starts.

## Components

### `authorized-keys-sync.service` (template -> /etc/systemd/system/)

- `[Unit]`
  - `Description=Sync authorized_keys from GitHub Gist`
  - `After=network-online.target`
  - `Wants=network-online.target`
- `[Service]`
  - `Type=oneshot`
  - `ExecStart=/bin/sh -c 'wget -q -O /home/fekr/.ssh/authorized_keys.tmp <GIST_URL> && mv /home/fekr/.ssh/authorized_keys.tmp /home/fekr/.ssh/authorized_keys && chown fekr:fekr /home/fekr/.ssh/authorized_keys && chmod 600 /home/fekr/.ssh/authorized_keys'`

The fetch writes to a temp file and only `mv`s it into place on success, so a
partial or aborted download can never truncate `authorized_keys`.

### `authorized-keys-sync.timer` (template -> /etc/systemd/system/)

- `[Unit]`
  - `Description=Periodic authorized_keys sync`
- `[Timer]`
  - `OnBootSec=5min`
  - `OnUnitActiveSec=5min`
  - `Persistent=true`
- `[Install]`
  - `WantedBy=timers.target`

`Persistent=true` catches up after the machine was off for a while.

## Ansible tasks added to `main.yml`

1. Template `authorized-keys-sync.service` to `/etc/systemd/system/`.
2. Template `authorized-keys-sync.timer` to `/etc/systemd/system/`.
3. `systemd: daemon_reload=yes`
4. `systemd: name=authorized-keys-sync.timer enabled=yes state=started`

The Gist URL and target user/path stay hardcoded in the template, matching the
current playbook style.

## Failure handling

A failed fetch (no network, gist temporarily down) makes `wget` exit non-zero.
The service logs the failure to journald; the timer simply retries on the next
5-minute tick. No mail alert. Existing keys are untouched because of the
temp-file + `mv` pattern.

## Testing

1. `ansible-playbook --check` (via `./command.sh` with check flag).
2. Full `ansible-playbook` run.
3. On the server: `systemctl status authorized-keys-sync.timer` shows `active
   (waiting)`.
4. On the server: `systemctl list-timers authorized-keys-sync` shows a `NEXT`
   trigger.
5. `cat /home/fekr/.ssh/authorized_keys` contains the expected keys with
   ownership `fekr:fekr` and mode `0600`.
