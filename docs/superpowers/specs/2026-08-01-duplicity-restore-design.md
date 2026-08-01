# Design: Conditional duplicity restore of /data

Date: 2026-08-01

## Purpose

Allow the playbook to restore the contents of the `/data` partition from a
Backblaze B2 duplicity backup when `/data` is empty, and to skip the restore
cleanly when it is not needed.

## Requirements

- Prompt at the start of the playbook for a B2 backup URL in the form
  `b2://DUPLICITY_API_KEY_ID:DUPLICITY_API_KEY@BUCKET_NAME`. The prompt is
  shown with `private: no`.
- If the URL is left empty, the restore is skipped entirely.
- If `/data` already contains files, the restore is skipped even when a URL
  was provided.
- The emptiness check ignores `lost+found`, which a fresh ext4 filesystem
  always contains.
- The backups are GPG-encrypted, so the restore needs a passphrase. Prompt for
  it only immediately before the restore step, and only when the restore will
  actually run. The passphrase must not be echoed or logged.
- Ensure `duplicity` is installed via `apt`.
- Restore the latest backup state.

## Approach

Add a new task file `tasks/restore.yml` imported from `main.yml` after the
`/data` mount task, matching the existing `tasks/authorized-keys-sync.yml`
pattern. A `vars_prompt` for the B2 URL is added at the play level of
`main.yml`.

## Tasks

1. **Install duplicity** — `apt` module, `state: present`. Runs only when the
   B2 URL is non-empty.
2. **Check `/data` emptiness** — `find` module on `/data`, `file_type: any`,
   `recurse: no` (top level only), results registered. Runs only when the B2
   URL is non-empty.
3. **Prompt for GPG passphrase** — `pause` module with `prompt` and
   `echo: no`, `register`, `no_log: yes`. Runs only when the B2 URL is
   non-empty and `/data` has no entries besides `lost+found`.
4. **Restore** — `command` module running
   `duplicity restore --force {{ duplicity_b2_url }} /data` with
   `environment: { PASSPHRASE: <prompt input> }` and `no_log: yes`.

## Skip logic

Restore and duplicity install run only when `duplicity_b2_url` is non-empty
**and** `/data` contains no entries other than `lost+found`. An empty prompt
and an already-populated `/data` therefore both skip identically.

## Integration

- Add `vars_prompt` block to `main.yml` with `name: duplicity_b2_url`,
  `private: no`.
- Add `- import_tasks: tasks/restore.yml` to `main.yml` after the
  `Ensure data partition is mounted` task.

## Security

- The passphrase prompt and the restore task use `no_log: yes`.
- The passphrase is never persisted; it is passed only as the `PASSPHRASE`
  process environment variable for the duplicity restore.
- The B2 URL (containing API credentials) is only ever in the restore
  command, which is `no_log`.

## Out of scope

- Regular scheduled backups (the user runs duplicity backups separately).
- Restoring a specific point in time (always the latest state).
- Handling of additional encryption schemes beyond duplicity's default GPG.
