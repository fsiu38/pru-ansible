# check_process_king

Read-only per-host process checks (Ansible 2.9 compatible).

Set `before_check_process` in inventory variables, for example:

```yaml
before_check_process:
  app01:
    - process_a
    - process_b
```

Invoke the role with `roles: [check_process_king]`. Unlisted hosts are skipped.
Each process is passed as the **first argument** to `check_process.sh`. Verify that
the script accepts an argument and is read-only before using this role.
`check_mode: false` intentionally runs the script even under Ansible check mode.

Results are registered as `before_check_process_result`; failures are recorded
but do not fail the play (`failed_when: false`). Review each item's `rc` and
`stdout` before relying on the result. This is a testing role, not a production gate.

No internal server names are committed because this repository is public.
