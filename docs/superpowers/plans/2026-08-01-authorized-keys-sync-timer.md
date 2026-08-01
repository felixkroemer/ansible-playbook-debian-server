# Periodic authorized_keys Sync Timer Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a systemd timer managed by the Ansible playbook that re-fetches the SSH `authorized_keys` file from the GitHub Gist every 5 minutes.

**Architecture:** Two systemd drop-in units are templated by Ansible into `/etc/systemd/system/`: a oneshot `.service` that fetches the keys via `wget` (writing to a temp file then atomically moving it into place) and a `.timer` that triggers the service every 5 minutes. Ansible enables and starts the timer; the existing one-shot task in `main.yml` remains for immediate keys on first run.

**Tech Stack:** Ansible, systemd units, wget, Jinja2 templates.

## Global Constraints

- Gist URL: `https://gist.githubusercontent.com/felixkroemer/c628ea1015e56f7d2f42f0c92c67ec92/raw`
- Target user: `fekr`; authorized_keys path: `/home/fekr/.ssh/authorized_keys`
- Final file must be owned `fekr:fekr` with mode `0600`
- Fetch interval: 5 minutes (`OnBootSec=5min`, `OnUnitActiveSec=5min`)
- `Persistent=true` on the timer
- The Gist URL and user/path stay hardcoded in the templates (matching current playbook style, per spec)

---

### Task 1: Add systemd unit templates

**Files:**
- Create: `templates/authorized-keys-sync.service.j2`
- Create: `templates/authorized-keys-sync.timer.j2`

**Interfaces:**
- Produces: two Jinja2 templates that render to `/etc/systemd/system/authorized-keys-sync.service` and `/etc/systemd/system/authorized-keys-sync.timer`. Used verbatim by Task 2's `template` module calls.

- [ ] **Step 1: Create `templates/authorized-keys-sync.service.j2`**

```ini
[Unit]
Description=Sync authorized_keys from GitHub Gist
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/bin/sh -c 'wget -q -O /home/fekr/.ssh/authorized_keys.tmp https://gist.githubusercontent.com/felixkroemer/c628ea1015e56f7d2f42f0c92c67ec92/raw && mv /home/fekr/.ssh/authorized_keys.tmp /home/fekr/.ssh/authorized_keys && chown fekr:fekr /home/fekr/.ssh/authorized_keys && chmod 600 /home/fekr/.ssh/authorized_keys'
```

- [ ] **Step 2: Create `templates/authorized-keys-sync.timer.j2`**

```ini
[Unit]
Description=Periodic authorized_keys sync

[Timer]
OnBootSec=5min
OnUnitActiveSec=5min
Persistent=true

[Install]
WantedBy=timers.target
```

- [ ] **Step 3: Verify templates render**

Run: `python3 -c "import yaml"` to confirm PyYAML availability for template render check below (expected: no output, exit 0).

Run: `python3 - <<'EOF'
import pathlib
service = pathlib.Path("templates/authorized-keys-sync.service.j2").read_text()
timer = pathlib.Path("templates/authorized-keys-sync.timer.j2").read_text()
assert "[Unit]" in service and "[Service]" in service
assert "[Unit]" in timer and "[Timer]" in timer and "OnUnitActiveSec=5min" in timer
assert "gist.githubusercontent.com" in service
print("OK")
EOF`
Expected: prints `OK`

- [ ] **Step 4: Commit**

```bash
git add templates/authorized-keys-sync.service.j2 templates/authorized-keys-sync.timer.j2
git commit -m "feat: add systemd unit templates for authorized_keys sync"
```

---

### Task 2: Wire timer tasks into main.yml

**Files:**
- Modify: `main.yml:23-25` (after the existing "Copy ssh authorized_keys from Github" task, before `vars_files`)

**Interfaces:**
- Consumes: the two templates created in Task 1 (paths `templates/authorized-keys-sync.service.j2`, `templates/authorized-keys-sync.timer.j2`).
- Produces: `main.yml` that installs, reloads, enables, and starts `authorized-keys-sync.timer` on the target server.

- [ ] **Step 1: Add the timer tasks to `main.yml`**

Insert after the existing "Copy ssh authorized_keys from Github" task (currently lines 23-25):

```yaml
    - name: Copy authorized-keys-sync systemd service unit
      template:
        src: templates/authorized-keys-sync.service.j2
        dest: /etc/systemd/system/authorized-keys-sync.service
        mode: '0644'
    - name: Copy authorized-keys-sync systemd timer unit
      template:
        src: templates/authorized-keys-sync.timer.j2
        dest: /etc/systemd/system/authorized-keys-sync.timer
        mode: '0644'
    - name: Reload systemd daemon for new units
      systemd:
        daemon_reload: yes
    - name: Enable and start authorized-keys-sync timer
      systemd:
        name: authorized-keys-sync.timer
        enabled: yes
        state: started
```

- [ ] **Step 2: Verify YAML syntax**

Run: `ansible-playbook --syntax-check main.yml`
Expected: exit 0 with `playbook: main.yml` in output (no errors)

- [ ] **Step 3: Verify playbook with check mode**

Run: `ansible-playbook --check --ask-become-pass --ask-vault-pass main.yml`
Expected: all existing tasks report `ok`/`changed` (check mode), no failures. The four new tasks report `changed` on first check.

- [ ] **Step 4: Commit**

```bash
git add main.yml
git commit -m "feat: add systemd timer tasks for periodic authorized_keys sync"
```

---

### Task 3: Apply and verify on server

**Files:**
- None (runtime verification only)

**Interfaces:**
- Consumes: the playbook changes from Task 2.

- [ ] **Step 1: Run the full playbook**

Run: `./command.sh`
Expected: play completes with no failed tasks; `enabled`/`started` report `changed` (or `ok` on re-run).

- [ ] **Step 2: Verify timer is active**

Run on server: `systemctl status authorized-keys-sync.timer`
Expected: `Active: active (waiting)`.

- [ ] **Step 3: Verify next trigger is scheduled**

Run on server: `systemctl list-timers authorized-keys-sync --all`
Expected: a row for `authorized-keys-sync.timer` with a `NEXT` value within ~5 minutes.

- [ ] **Step 4: Verify authorized_keys is fresh and correct**

Run on server: `ls -la /home/fekr/.ssh/authorized_keys`
Expected: owned `fekr:fekr`, mode `-rw-------`.

Run on server: `cat /home/fekr/.ssh/authorized_keys`
Expected: contains the current Gist keys (public keys starting with `ssh-` or `ecdsa-`).

- [ ] **Step 5: Commit any fix-ups (only if Step 1-4 revealed problems)**

If the playbook or templates needed corrections, fix them, re-run the affected step, then:

```bash
git add -A
git commit -m "fix: correct authorized_keys sync timer setup"
```

If nothing needed fixing, skip this step.
