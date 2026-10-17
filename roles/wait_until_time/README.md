# wait_until_time role (Ansible 2.9 / Tower)

Reusable **time gate** for Tower Workflows. The next task / Workflow Job must not start before the configured JST timestamp.

## Inputs

- `maintenance_date` (**required**, `YYYY-MM-DD`): the maintenance START date in Japan.
- `wait_target_time` (**required**, `HH:MM`): target time in JST.
  - `00:00` to `23:59` = time on `maintenance_date`.
  - `24:00` to `24:59` = **the following day**, midnight hour. E.g. `24:05` = next-day `00:05`.
- `wait_timezone`: only `Asia/Tokyo` is supported (default).
- `wait_max_seconds`: maximum future waiting duration, default 86,400 s.
- `wait_max_late_seconds`: tolerated lateness, default 3,600 s.

The date is intentionally required: each Tower node can start before or after midnight, so inferring the date from **today** would be unsafe.

## Usage in an independent Tower Workflow node

Job Template Playbook: [`wait_until_time.yml`](../../wait_until_time.yml)

Provide the two variables via Job Template extra vars / Survey and run before the protected Job. **Connect the next node with Success only.**

```bash
ansible-playbook -i localhost, -c local wait_until_time.yml \
  -e '{"maintenance_date":"2026-10-18","wait_target_time":"24:05"}'
```

To guard a task in the same Playbook:

```yaml
- name: Gate a maintenance task
  hosts: localhost
  connection: local
  gather_facts: no
  tasks:
    - name: Wait until the allowed start time
      include_role:
        name: wait_until_time
      vars:
        maintenance_date: "2026-10-18"
        wait_target_time: "23:00"
    - name: Next task after the gate
      debug:
        msg: "The scheduled gate is now open"
```

## Proposed maintenance timeline

Example: `maintenance_date: "2026-10-18"` (JST).

| Gate | JST actual date/time | Protected operation |
|---|---|---|
| `23:00` | Oct 18, 23:00 | Backup |
| `23:30` | Oct 18, 23:30 | IWS Limit / stop new scheduling, then **check running jobs are drained** |
| `24:00` | Oct 19, 00:00 | Splunk Suppress |
| `24:05` | Oct 19, 00:05 | FTA Unlink |
| after unlink | Based on operational checks | Stop FTA → OS precheck → OS Patch |

Suggested Tower pattern:

```text
Wait(23:00) → Backup
  → Wait(23:30) → IWS Limit / Drain Validation
  → Wait(24:00) → Splunk Suppress
  → Wait(24:05) → FTA Unlink
  → Stop IWSFTA / Verify Stopped → OS Before → OS Patch
```

**Important:** current `IWS_before.yml` runs Limit → Unlink → Stop together in one Job; it **must be split or refactored** before time gates can be inserted between those IWS operations. This feature branch does **not** change IWS Playbooks or patching.

## Safety / implementation notes

- This role uses `date` on the **Tower controller / execution environment**, with `TZ=Asia/Tokyo`; Linux GNU coreutils `date` and JST tzdata must be installed. No SSH to target hosts is needed.
- If launched early, `pause` blocks until the gate. After waiting, the role checks the clock again; manually continuing `pause` early will fail the Job rather than permitting work.
- If launched **after** the gate but within `wait_max_late_seconds`, the role succeeds immediately. Beyond that, it fails to prevent stale maintenance runs.
- If the gate is further away than `wait_max_seconds`, fail rather than hold a Tower worker indefinitely.
- The Tower Job Timeout must exceed the maximum expected wait. Long waits occupy a Tower execution slot; **Tower Schedules** are generally better for launching the first 23:00 Backup without consuming a slot.
- Correct wait time does **not** guarantee Backup success, IWS Job Drain, valid maintenance authorization, or successful Splunk suppression. These require separate checks.
- Clock sync (NTP), timezone data, Workflow Success links, permissions and failure recovery must be validated before Production use.

## Validation

Run on a test controller with **Ansible 2.9**:

```bash
ansible-playbook --syntax-check wait_until_time.yml
ansible-playbook -i localhost, -c local wait_until_time.yml \
  -e '{"maintenance_date":"2026-10-18","wait_target_time":"24:05"}'
```

Test cases: future gate (wait), past gate within 1 hour (pass), stale gate (fail), invalid `24:60` (fail), `24:05` next-day mapping, and manual early `pause` interruption (fail).


## Sequential read-only test: uname at 23:00, df at 24:05

The separate [`test_wait_sequence.yml`](../../test_wait_sequence.yml) demonstrates calling the same role **twice in one Playbook**, then running commands on selected Linux hosts:

```text
Wait until maintenance_date 23:00 JST
  → uname -a (all selected hosts)
  → Wait until maintenance_date + 1 day 00:05 JST (24:05)
  → df -h (all selected hosts)
```

This test defaults to `localhost` and only runs **read-only** commands. The `wait_until_time` role is delegated to the Tower controller and invoked once using `apply: delegate_to: localhost` and `run_once`; `uname` and `df` run on the selected hosts. Using the **same hosts pattern for the entire Play** avoids the problem of separate localhost-only plays being excluded by a Tower host Limit.

Example: test on controller (change the date to your actual maintenance start date; do **not** use this particular past date as an active window):

```bash
ansible-playbook -i localhost, -c local test_wait_sequence.yml \
  -e '{"maintenance_date":"2026-10-18"}'
```

To run on real Linux targets, configure actual credentials/inventory and pass their host group or host pattern:

```bash
ansible-playbook -i /path/to/real-inventory test_wait_sequence.yml \
  -e '{"maintenance_date":"2026-10-18","maintenance_hosts":"linux_patch_targets"}'
```

For quick tests, `uname_at` and `storage_at` can override the two default times:

```yaml
maintenance_date: "2026-10-18"
maintenance_hosts: "linux_patch_targets"
uname_at: "23:00"
storage_at: "24:05"
```

If a command fails on any selected host, `any_errors_fatal: yes` prevents advancing to the next time-gated action. The second gate only opens **after all selected hosts complete `uname`**, and no earlier than 24:05 JST. The role's lateness limit still applies. The `df -h` check reports filesystem space usage; it does not test disk integrity or storage hardware health.

**Test status:** YAML structure checked, but not executed against Ansible 2.9/Tower or real target hosts. Do not merge before a syntax check, controller clock validation, multi-host test and a short-interval end-to-end dry run.
