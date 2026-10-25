# check_process_king

Ansible 2.9-compatible role. Only `tasks/main.yml` is needed; no `defaults/main.yml`.

## Usage

Without `check_phase`: run `/prustaff/dsops/reboot/hanyo/check_process.sh` once per targeted host and display stdout. Nonzero exit status fails the play.

With `check_phase: before`, `during`, or `after`: match strings in stdout based on server-specific `process_checks`. Missing host/phase/up/down entries are skipped (including the script when there are no checks).

```yaml
process_checks:
  server01:
    before:
      up:
        - dmgr
        - nodeagent
      down:
        - "啟動中"
    during:
      down:
        - dmgr
        - nodeagent
    after:
      up:
        - dmgr
        - nodeagent
```

- `up`: string must appear in stdout.
- `down`: string must not appear in stdout.
- One script invocation per host/phase, regardless of number of entries.
- Script exit failure or mismatched expected string fails the play.
- `process_checks` may be supplied through inventory variables.
- Optional `check_process_king_script` overrides the script path; absent means `/prustaff/dsops/reboot/hanyo/check_process.sh`.
- The command runs under Ansible check mode too (`check_mode: false`); only use a read-only script.

Example:

```bash
ansible-playbook check_process_king.yml
ansible-playbook check_process_king.yml -e check_phase=before
```

Not yet verified against actual script output or a Tower host.
