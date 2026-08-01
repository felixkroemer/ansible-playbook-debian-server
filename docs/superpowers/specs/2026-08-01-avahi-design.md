# Design: Avahi mDNS setup for .local access

Date: 2026-08-01

## Purpose

Ensure the avahi daemon is installed and running on the Debian server so it is
reachable on the local network via its `.local` mDNS address.

## Requirements

- Install `avahi-daemon` (and `avahi-utils` for verification tooling).
- Enable and start the `avahi-daemon` systemd service.
- Advertise the system's existing hostname (no custom name config).
- No firewall changes.
- Verify at runtime that the `.local` hostname actually resolves to an IP.

## Approach

Add a new task file `tasks/avahi.yml` imported from `main.yml`, matching the
existing `tasks/authorized-keys-sync.yml` pattern.

## Tasks

1. **Install packages** — `apt` module, package list `[avahi-daemon, avahi-utils]`,
   `state: present`.
2. **Enable and start service** — `systemd` module on `avahi-daemon`,
   `enabled: yes`, `state: started`.
3. **Runtime verification** — run `avahi-resolve-host-name {{ ansible_hostname }}.local`
   with a retry loop (up to 10 attempts, 3s apart) and assert the output
   contains an IPv4 address, confirming avahi is actively publishing.

## Integration

Add `- import_tasks: tasks/avahi.yml` to `main.yml` after the existing
`authorized-keys-sync` import.

## Out of scope

- Firewall rules (user has no firewall on this server).
- Custom hostname advertisement.
- Hardening of avahi configuration.
- Services other than hostname publishing.
