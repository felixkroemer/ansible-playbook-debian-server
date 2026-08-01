# Conditional duplicity Restore Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a conditional duplicity restore of `/data` from a B2 backend, prompted for at playbook start and skipped when no URL is given or `/data` is already populated.

**Architecture:** Add `tasks/restore.yml` following the existing `import_tasks` pattern. A `vars_prompt` at the play level of `main.yml` collects the B2 URL. `restore.yml` installs duplicity, checks whether `/data` is empty (ignoring `lost+found`), prompts for the GPG passphrase via the `pause` module only when the restore will run, and runs `duplicity restore --force`. The whole block is gated on the B2 URL being non-empty.

**Tech Stack:** Ansible (vars_prompt, apt, find, pause, command, debug modules), duplicity.

## Global Constraints

- Target host: Debian (use `apt`).
- B2 URL prompt: var name `duplicity_b2_url`, form `b2://DUPLICITY_API_KEY_ID:DUPLICITY_API_KEY@BUCKET_NAME`, `private: no`.
- Restore runs only when `duplicity_b2_url` is non-empty **and** `/data` contains no top-level entries besides `/data/lost+found`.
- GPG passphrase is prompted via `pause` with `echo: no`, registered, and marked `no_log: true`.
- The prompt and restore tasks are marked `no_log: true`; the passphrase is passed only as the `PASSPHRASE` environment variable.
- Restore the latest backup state using `duplicity restore --force`.
- Use short-form module names consistent with existing task files.
- Verification is `ansible-playbook --syntax-check main.yml` (no automated test framework in this repo); full functional verification is manual by the user.

---

### Task 1: Create the duplicity restore task file

**Files:**
- Create: `tasks/restore.yml`

**Interfaces:**
- Consumes: `duplicity_b2_url` (string, from `vars_prompt` in `main.yml`; empty string means skip).
- Produces: `tasks/restore.yml` — installs duplicity, checks `/data` emptiness, prompts for the GPG passphrase, and restores. Consumed by `main.yml` via `import_tasks`.

- [ ] **Step 1: Create `tasks/restore.yml`**

```yaml
- name: Skip duplicity restore when no backup URL provided
  debug:
    msg: "No B2 backup URL provided; skipping duplicity restore"
  when: duplicity_b2_url == ''

- block:
    - name: Install duplicity
      apt:
        name: duplicity
        state: present

    - name: Check whether /data is empty
      find:
        paths: /data
        file_type: any
        recurse: no
      register: data_contents

    - name: Prompt for duplicity GPG passphrase
      pause:
        prompt: "Enter the duplicity GPG passphrase (required to decrypt the restore)"
        echo: no
      register: duplicity_passphrase
      no_log: true
      when: data_contents.files | rejectattr('path', 'equalto', '/data/lost+found') | list | length == 0

    - name: Restore /data from duplicity backup
      command: duplicity restore --force {{ duplicity_b2_url }} /data
      environment:
        PASSPHRASE: "{{ duplicity_passphrase.user_input }}"
      no_log: true
      when: data_contents.files | rejectattr('path', 'equalto', '/data/lost+found') | list | length == 0
  when: duplicity_b2_url != ''
```

- [ ] **Step 2: Verify file syntax**

Run: `python3 -c "import yaml, sys; yaml.safe_load(open('tasks/restore.yml')); print('OK')"`
Expected: `OK`. (Full Ansible syntax validation happens in Task 2 once `main.yml` imports this file.)

- [ ] **Step 3: Commit**

```bash
git add tasks/restore.yml
git commit -m "feat: add duplicity restore task file"
```

### Task 2: Wire restore into main playbook

**Files:**
- Modify: `main.yml:16` (insert import line directly after the mount task) and `main.yml:31` (add `vars_prompt` after the `vars:` block)

**Interfaces:**
- Consumes: `tasks/restore.yml` from Task 1.
- Produces: updated `main.yml` that prompts for the B2 URL at play start and runs the restore tasks after mounting `/data`.

- [ ] **Step 1: Add `vars_prompt` to `main.yml`**

Append after line 31 (`    nfs_exports: [ "/data    *(rw,sync)" ]`):

```yaml
  vars_prompt:
    - name: duplicity_b2_url
      prompt: "B2 backup URL (b2://DUPLICITY_API_KEY_ID:DUPLICITY_API_KEY@BUCKET_NAME)? Leave empty to skip restore"
      private: no
```

- [ ] **Step 2: Add import line to `main.yml`**

Insert directly after line 15 (`        state: mounted`), before the `Ensure .ssh directory exists` task:

```yaml
    - import_tasks: tasks/restore.yml
```

- [ ] **Step 3: Verify file syntax**

Run: `ansible-playbook --syntax-check main.yml`
Expected: PASS, no syntax errors. This statically parses the newly imported `tasks/restore.yml`.

- [ ] **Step 4: Commit**

```bash
git add main.yml
git commit -m "feat: prompt for B2 URL and import duplicity restore"
```
