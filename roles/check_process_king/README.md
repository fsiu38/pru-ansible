# check_process_king

Ansible 2.9-compatible test role for three maintenance phases.

## Configuration: Server -> Phase -> Status

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
        - "啟動中"
  server02:
    before:
      up:
        - "xpopserver stat"
    after:
      up:
        - "xpopserver stat"
```

`before`, `during`, `after` are phases. `up` means the
expected text should appear; `down` means it should not appear.
Missing hosts, phases, or statuses are skipped.

Call the role with `check_phase: before` (or `during` / `after`).
Store real server definitions in inventory `host_vars` / `group_vars`
instead of this public repository.

**Current limitation:** The role runs `check_process.sh '<entry>'`
for each configured string, and registers output in
`check_process_king_result`. It does **not yet enforce** up/down
expectations. Verify whether the script accepts one quoted argument,
what its stdout contains, and its exit-code conventions first.
A missing string must not be treated as a successful DOWN check
when the script itself fails.

`check_mode: false` runs the script even under Ansible check mode.
Use only with a read-only script. Do not use this test role as a
production maintenance gate yet.
